---
layout: post
title: "[未完] Introduction to Actor Model"
date: 2012-07-19 14:42
comments: true
categories: 
---

The Challenge
===========================================================================

在今天的講題開始前，我要先花一點時間講一下，我們這一代程式設計師面對的挑戰是什麼，為什麼我們需要一個新的程式架構？

大家應該都很熟悉摩爾定律，摩爾定律是指：一個尺寸相同的晶片上，所容納的電晶體數量，約每隔18個月便會增加一倍，性能也將提升一倍。
在Dr. Moore於1965年提出摩爾而後的40年間，摩爾定律準確的預測電晶體的發展，每18月就有更快一倍的處理器出來，每18個月記憶體的就倍
增。對於軟體工程師來說，摩爾定律讓我們的程式，不用更新軟體，每隔一段時間，效能就自動增加。

然而這一切，在 2005 年左右開始崩壞，那年，我在 ACM Queue 九月號讀到 "The Future of Microprocessors",
"The Free Lunch Is Over: A Fundamental Turn Toward Concurrency in Software."等文章，提到，軟體工程師的免費午餐
已經結束了，未來，處理器的時脈將不會隨著電晶體數而增長，只有處理器的核心數依然跟隨著摩爾定律，每18個月增加一倍，而後，類似的
文章開始大量的出現在各樣的軟體雜誌之中。 Intel於 2006 年一月發表了個人電腦用的雙核處理器 - Core Duo， 2007 年四核的Core 2 Quad
發表，我想現在在場的聽眾中，有不少的是用 Intel i7 八核的處理器。

對軟體工程師來說，新的現實是 Amdahl's Law ，一個程式能被多核心處理器加速的程度，不會高於程式中不可平行處理的部份的倒數。
也就是說，如果程式中只有50%的部份是可平行處理的，那麼，多核心的處理器最多只能幫你加速兩倍，不管你有雙核，四核，還是六十四
核等。如果程式中有 95% 的部份是可平行處理的，縱使你有兩千核的處理器，還是只能幫你加速 20 倍。

不管你是寫單機版程式，或者是多人用的網路系統，如何善用多核心的處理器，是現在不得不開始面對的問題。

Concurrency and Parallelism
=============================================================================

接下來我們要先定義一下名詞，我們中文中講的平行處理，代表了兩個不同的涵意，一個是 Concurrency ， 另一個是 Parallelism

Parallelism 講的是，程式中有兩個以上的部份，可以同時間被處理，而不會互相干擾。

Concurrency 指的是，程式中的兩個以上的部份，可以同時間被處理，或者透過分時分享資源，同時間被處理器處理。

然而，對軟體工程師來說，這兩者都是很困難的，因為我們的程式不免會用到共享的資源，在 Parallelism 中，如何找出共享的資源
將這些模組分離出來最小化，是一種挑戰，在 Concurrency 中，如何確保共享的資源不變成程式中的瓶頸，又是另一個挑戰。

在 Java 6 Concurrency API 就提供了不少工具來協助處理 Concurrency 的問題，如 Executor, CountDownLatch, Semaphore
, DelayQueue 等工具。


Issue: Shared Memory Concurrency
===================================================================================

那 Shared Memory Concurrency 的問題在那呢？我記得我在學寫網頁時，第一次寫 Web Counter 時就學到了，原來小
到一個 counter++ ，都會因為 thread 存取的順序，而發生 race condition 造成結果與預測的不同。Shared Memory
Concurrency的難點就在於，如何正確的使用 Lock 及 Lock 的不可預測性。

由於作業系統設計時，為了效能，不會將 Lock 設計成完全的公平，當有三個 Thread 依序去要一個 Lock 時，你執行三次，
可能三次都是不同的 Thread 第一個拿到這個 Lock ，這個特性，造成了測試時及執行時的不可決定性，無法在測試時，模擬可
能發生的所有狀況。

此外 Lock 並不是容易去使用的，如果你在程式中使用了大量的 Lock ，將會造成程式中可被併行處理的部份減少，在 Java6
之前，若是你在沒有需要用 lock 的地方使用了 lock ， jvm 仍然會去要這個 lock ，在 Java6 Mustang Release中， HotSpot JVM
的實作引入了 lock elision 的觀念， HotSpot VM 將會透過執行階段的分析，去忽略掉這些多餘的 lock。

然而， Lock 仍帶入了許多問題，如 lock starvation ，當許多 thread 大量、重覆的取用同一個 lock 時，由於取得 lock
的機制並不是公平的，所以有可能會有 thread 一直取不到 lock 的狀況發生。

另外，若是多個 Thread 都要存取相同的數個 lock ，則會造成互相卡位的狀況，形成 live lock。

當然，你也可以說，用 Lock 太麻煩，而減少 Lock 的使用，那麼你的程式就會變得不 Thread-Safe ，在處理量大時，會發生
Race Condition造成不可預期的結果產生。

而用 Lock 最讓人絕望的是，若是 Lock 的順序弄錯了，會造成 Dead Lock ，變成，程式在設計的初期就要把各物件呼叫的順序
先定出來，才可以避免 Dead Lock 的產生，但是你能保證你的程式 1.0版、1.1版、2.0版的呼叫順序都不變嗎？


False sharing: Cache Line Issue
-----------------------------------------------------------------------------------------------------

如果說 Lock 的問題就夠讓人頭大了，那麼底下這個我最近讀到的問題，則會讓你更吃驚。在電腦的架構中，我們有主記憶體、處理器
上有L1 L2 L3快取，處理器存取資料時，會把資料放在快取中，在適當的時機 在把修改寫回主記憶體；這點，大家應該不難去想像。

然而，處理器在從主記憶體存取資料至快取時，是一個區塊一個區塊的去存取，所以在取用資料時，不只會拿回所需的資料，同時也會
取用到相鄰的資料，這在多核心的機器上造成，若是core A 及 core B會去存取相鄰的資料，那麼他鍆將互相 invalidate 對方的
L1快取，造成要從下層的快取或主計憶體上重新讀取資料。

在一篇測試報告中，在一台八核的機器上的測式結果，有False Sharing問題的程式將比沒有False Sharing問題的程式慢12倍，這
數字差不多就是L1及L2讀取速度的差異。依此推測，若是要從主記億體重讀的話，則會慢上200倍！


The Solution
===========================================================================================

A new high level programming model
-------------------------------------------------------------------------------------------

講完了問題，當然之後，當然要開始講解決方案了；許多大神、大師們覺得我們須要一個不同的 Programming Model
來解決這問題，他們覺的，這個新的解決方案要有底下幾個特性

- 要容易被理解，跟容易被測試。
- 換言之，也就是要deterministic，如果程式流程是輸入 A 產出 B ，那麼不管執行多少次，這個結果仍是要
  一樣的。
- 既然 shared mutable state 是問題的根源，我們就把這問題給移除。
- 能夠善用多核心所帶來的優點。

Possible Solutions
----------------------------------------------------------------------------

有個不同的解決方案被提出來，首先是 Functional Programming ，完全的把 Mutable State 移除，程式變
成如同數學函式一般可以被堆疊，程式函式的本身不帶有狀態，所有的狀態都是由外部傳入的，如此一來，程式本身
的運算處理就變的 deterministic

而既然函式本身不帶有狀態，那麼他們便可以平行的被處理，例如在 scala 中，就引如了 parallel collection
，讓軟體工程師可以很簡單的就會一群資料做平行的運算

.. code-block:: scala

  scala> List(1, 2, 3).par.map(_ + 2)
  res: List[Int] = List(3, 4, 5)

Functional Programming所需的 Lambda expression 將會在 Java8 時被實作出來，而 Parallel Collection
也有機會在 Java8 時出現。

在此順便打個廣告，台灣目前有個 functional programming user group 在運作中，若是各位有性去，可以上網
找 fpug.tw

而另一個可能的解決方案是 Actor Model ， Actor Model 將狀態封裝在Actor內部，Actor及Actor間，則是透過
非同步的機制去傳遞訊息。


A Brief History of the Actor Model
=========================================================================================

在開始介紹 Actor Model 前，先簡單介紹一下 Actor Model 的歷史。 Actor Model 是於 1973 年 Carl Hewitt
命名的，而由 Gul Agha 這位先生於 1985 年左右更精確的描述 Actor Model 。

第一個大型的商業應用，則是由 Ericsson 於 1980 年代中期所完成的一個電信系統，這個系統非常的容錯，當大家在介
紹 Actor Model 時，多會提到這個例子，因為這個系統達到了 99.9999999% 九個九的 uptime ，也就是說
一整年下來，他停機的時間只有 31 ms ，非常的匪疑所思，因為，一般我們在寫的系統，能有兩個九，就已經是很了不
起的成就了。

這個專案另一個有名的產品是 erlang 這程式語言， erlang 在 1990 年代被 opensource ，後來被大量運用在
message queue 跟 Concurrency 的程式之中。例如大家常用的 RabbitMQ 就是利用 erlang 寫成的，據說他的核
心只有 5000 行 erlang 的程式碼。

Actor
===========================================================================================

那什麼是 Actor 呢， Actor 是一個非常輕量化的元件，在 Akka 的實作中，每一個 Actor 只占了約400 bytes
的記憶體空間，每一個 Actor 擁有自己的狀態以及行為，將些狀態及行為都被封裝在 Actor 裡面。

其他的 Actor 只能夠透過送訊息的方式，來與這個 Actor 來溝通，存取 Actor 的狀態及觸發行為；當 Actor 收
到外部的訊息時，這些訊息會被放在信箱裡，依照順序來被這個 Actor 來處理。

這個送訊息的及處理訊息的機制，是非同步的，你不能假設這個訊息一定會立刻被同一個 thread 處理，甚至，你不能
假設說，你一定會收到一個回應。

當 Actor 處理訊息時，他會被放在系統中預先配制好的 Thread 上執行、處理訊息。

很有趣的是，當你這樣設計一個系統時，你的系統便會變得非常的 scalable 及非常的輕量化。因為使用 Actor Model
你的 call stack 會變的很小，在我們用 imperative programming ，我們的 call stack 是從系統最外部，透過
一層層的 method call 累加上來起來，當 CPU 在 Context Switching ，在從一個 Thread 跳到另一個 Thread 去
執行時，需要讀入該 Thread 的 Call Stack ，當 Call Stack 越大時，Context Switch的成本就愈高。

在Actor Model 中，兩個 Actor 在傳遞訊息時，其實只有把這個訊息的 reference 丟到另一個 Actor 的信箱中，訊
息傳遞的成本非常的小，而 Actor 在被喚起處理訊息時，他的 call stack 是從他的信箱開始，而非整串的呼叫流程，因
此， context switch 的成本變的很小。

在 Actor Model 中，一個工作的處理，會變得很像我們在辦公室內做業的方法，當我們從系統外部收到一個訊息時，我們
會把這個工作連同這個工作的寄送者先記錄下來，放到信箱中延後處理，當前一手完成工作後，運行中的 Actor 會把處理
好的資料轉發給下一手，就這樣一層層轉發，最後在將完成的工作結果，送還給原始的寄送者。


Introduce Akka
==============================================================================================

接下來的部份，我們要介紹 Akka 這個 JVM 上 Actor Model 的實作。

Akka 是由 Jonas Boner 於 2009 年所發起的，在開發 Akka 之前， Jonas 是在 Terracotta 做 JVM Clustering 以及在 BEA
做 JRockit VM 的，具有多年食作 distributed system 的經驗。目前 Akka 背後的商業公司是 typesafe ， typesafe
同時也是開發 Scala 這個 JVM 上最盛行 JAVA.NEXT 語言的公司。

這個實作是跑在 JVM 上，提供了 Java 及 Scala 的 API ，在接下來的簡報中，我會用 Java API 為主 Scala API 為輔
來介紹怎麼使用 Akka。

Akka另外也引入了 remote actor 這個觀念，既然，我們所有的訊息傳遞都是非同步的，透過訊息的傳遞來呼叫，而這訊息本
身，又是不帶狀態，可以被 serialize 的，狀態的本身是存在 Actor 中的，那麼，為什麼我們自然有可能的將工作放到別台
機器上來處理。

除了 Actor Model 外，Akka還提供了許多 module 來跟外部的系統整合，如 akka-camel ，能讓 akka 串接上 camel 這
個 enterprise service bus ，便程它的一個 endpoint ，讓 Actor 能處理 camel 送來的訊息然後再將訊息轉發回 camel 中。


Define Actor
=================================================================================================

.. code-block:: java

  import akka.actor.UntypedActor;

  public class Counter extends UntypedActor {
    
     private int count = 0;
    
     public void onReceive(Object message) throws Exception {
       if (message.equals("increase") {
         count += 1;
      } else if (message.equals("get") {
        getSender().tell(new Result(count));
      } else {
        unhandled(message);
      }
    }
  }


Fault Tolerance in Akka
======================================================================================================

接下來我們要談的就是為什麼 ericsson 的系統能夠達到那麼高可靠度的原因以及 Akka 容錯的機制。

當一個系統在運作時，不免因為一些設計外的因素、或不可預期的輸入，造成系統產生錯誤的狀態，有時，這個錯
誤的狀態只是丟出一個 exception 就結束，但是在某些情況下，系統會處於一個混亂的狀態之中，無法再繼續處
理任何的訊息。

這個狀況，我相信大多數的 Windows 用戶都有碰上過，也非常清楚這個問題的解決方案，也就是 -- 重開機

在 Akka 中，Actor們如同現實生活中公司的組織，可以有低層的員工以及高層的 Supervisor ，當一個員工
在處理訊息時，若是有發生無法自己無法排解的錯誤狀況的話，他將會把這錯誤狀況，回報給他的上級，由上級來
決定如何處理。

如果說，一個 supervisor 管理的 workers 都是獨立的個體，沒有交互影響的問題，那麼， supervisor 可以
選擇將有問題的 worker 單獨重開機，當一個 actor 被重啟時，他的 actor reference 將保持不變，外面的 actor
仍可以透過原有舊的 actor reference 跟這個重啟後的 actor 溝通。而被重啟的 actor 的信箱，仍會保存的
尚未被處理的訊息在信箱內，而造成錯誤的訊息，則是在重啟時可以決定是否要保留重新處理。

若是一個 supervisor 下的 workers 有相互依賴的情況的話，那麼， supervisor 可以在某一個 worker 發
生錯誤時，重新啟動所有的 workers。

這個錯誤排解的機制不止是可以串上一層而以，而是可以一層層的堆疊起來，位在最上端的，則是前面所看過的 ActorSystem
；當一個錯誤是直屬主管無法排除的錯誤，那麼，這個錯誤訊息可以再轉發給更高層的主管，由他來解決。同樣的，重啟的觀
念也可以套用在這邊。當一個系統的子系統發生錯誤時，我們可以重新啟動整個環境


Remote Actor
=================================================================================================

在 Akka 的實作中 Actor 預設就是可以被 Remote 化的， Actor 在設計時，就已經是設計成 location transparent 及
可被配置在本機或是遠端的，一個原本設計成本機用的 Actor ，可以不經任何的程式變化，直接修改設定檔，變成在啟動一個
Actor時，自動就是啟動在遠端的。

而傳遞訊息給 Remote Actor 跟傳給本機的 Actor 是沒有差別的，同樣是透過 tell 及 ask 這兩個 API ，在底層實作上
Actor Reference 會帶有 Remote Actor 實際所在機器的位址，若是一個 Actor 是 Remote Actor 的話，被傳送的訊息
就會透過 Java Serialization, Protocol Buffer Serializer 或者是自定的 Serializer 來將訊息寫入到遠方。

而要使用那一種 Serializer ，是可以透過設定檔做切換的。

Remote Actor 的配置，也是可以透過程式來控製，下圖便是一個例子。

Routing & Clustering
================================================================================================

對於 Cluster 的設計，目前 Akka 的團隊還在重新設計中，目標將會是在下一個 release 時，將 cluster 的功能
加入到 Akka 中。

目前，Akka 則是只支援 Routing 而已。 Routing 的機制是說，你有一群一樣的 actor，他們可以處理同樣類型的訊息
，但是由誰來處理訊息，則是由 Router 來決定。

你可能會問，什麼時候需要這樣做呢？會有 Routing 的需求時，多是因為有處理量限制的問題，例如，我想要最多不超過五個
這類型的工作可以同時間被處理，或者是從使用者A來的需求，最多只讓它同時跑五個，但是從使用者B來的需求，最多可以跑十
個，而且兩個人的程序不能混用。

Akka 目前支援的 Routing 機制有

- RoundRobinRouter： 在一群 actor 中，依序將工作配發給他們。
- RandomRouter: 在一群 actor 中，隨意的將工作配發給他們。
- SmallestMailboxRouter: 在一群 actor 中，將工作配發給目前沒有在工作的，或者是待辦工作最少的 actor
- BroadcastRouter: 將工作配發給所有的 Actor
- ScatterGatherFirstCompletedRouter:  將工作配發給所有的 Actor，然後等待第一個回應，其餘的回應則是會被忽略捨棄掉。

Routing 的機制也可以跟 Remote Actor 混用，將 Remote Actor 配發在多台機器上，然後透過 Router 將訊息配
發給他們。


Performance
===================================================================================================

下圖的是 Akka 官方，於一台48核的 AMD 機器上，做簡單訊息傳送的 Performance Test 的測試結果。

在一開始的實作中，Akka是使用 java.util.concurrent.ThreadPoolExecutor 控制最多有多少個 Actor 能同
時被執行，然而在這 48核 的機器上，Akka Team發現，Akka無法 scale 超過 12 個併行的 actor ，不管你增加
多少個 actor ，同時間內被處理的最大量卡在每秒一百四十萬個訊息這個數字。然而，同時間 CPU 的 Load 並沒
有超過 10%

Akka Team猜想是因為 ThreadPoolExecutor 底層的 task queue, LinkedBlockingQueue 有一個共用的 lock，
最後造成擁塞，才會造成效率的不佳。

後來在經過了 Doug Lea 的協助，使用 Fork-Join 重新實作 task executor ，於是效能就拉高到每秒兩千萬個訊息
這數字。

Use Case
================================================================================================

那 Akka 可以被用在那些地方呢？目前 Akka 被應用在下面幾個地方。

- messaging system: 例如國內的 Cubie 就有用到 Akka 來處理訊息。
- Data Analysis: klout.com 例用 akka 來分析社群網站上如 facebook twitter 上訊息轉發的熱門程度
  以及誰是意見領袖等資料。
- 股票交易系統： Akka 的一大客戶群是倫敦的金融公司，Akka的幾個特性很適合用來做股市交易用。像是用 Actor
  來追蹤一個股票的現貨股價變化以及產生各種線型圖；又或者是說，結合 Rule Based Engine 可以做風險控管的
  模組。例如說交易員要下單買一支股票時，Akka可以幫你同時檢查，這個交易員他目前手上可用資金的是多少，
  交易員目前股票組合的風險是多少，公司目前整體股票組合的風險是多少。
  數十至數百個不同規則的檢查是可以透過 remote actor 分散在許多機器上併行處理的，一個交易的風險評估，並不
  會隨著檢查的數量而線性增長。
- 另外一個 Akka 被大量使用的環境是線上遊戲，過去有許多遊戲公司選擇用 MySQL 當後台，然後透過 cache 及 db
  sharding的方式，來增加 throughput ，後來他們發現，與其把 server 寫成 stateless 然後把狀態放在後台
  的 db 讓 db 變成效能的頻頸，不如把 state 放在 Actor 中，讓每個 Actor 代表一個遊戲中的角色，獨立維護
  它的狀態，這樣才能更 scalable.

Case Study
==================================================================================================



