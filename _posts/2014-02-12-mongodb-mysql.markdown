---
layout: post
title: "Mongodb vs MySQL/RDBMS"
date: 2014-02-12 16:24
comments: true
categories: 
---

許多人從 RDBMS 跳到 NoSQL 的第一站都是 MongoDB ，所以這一篇，就是特別寫來解釋，為什麼很多人選用 MongoDB ，以及解釋你為什麼不該用 MongoDB 來換掉 RDBMS 。

## MongoDB 熱門的理由

MongoDB 熱門的原因是因為他提供了一個可以跟 SQL 相比的 [Query Language] ，在 NoSQL 不同面向的眾多方案中， MongoDB 所能支援的 AdHoc Query 是沒人能比的， Riak 跟 CouchDB 的 AdHoc Query
也不差，但是 Riak 在跑 [MapReduce Query] 時是不能透過 index 加速的， CouchDB 則是跟本不支援 index ，所以在這邊 MongoDB **看似** 是個很好的 MySQL drop-in replacement

使用 MongoDB 來取代 MySQL ，在初期，看來是學習成本最低，轉換風險最低的解決方案，所以多數商業公司，為了降低風險，在追循潮流一窩風的採用 NoSQL 時，選擇了 MongoDB 這個看似合理且安全的選項。

然而糖衣底下掩蓋的，卻是數不盡的陷井在那。

## Trade-Off


技術的選用，往往是一連串的取捨，拿某樣功能去換回了另一項功能，所以，接下來我們就來看看，我們從 MySQL/RDBMS 跳到 MongoDB 去，我們到底是跟魔鬼做了什麼樣的交易，拿了什麼樣的東西，去換回了什麼。

首先，我們先看看我們換出去的是什麼？我們換出去的是 ACID (Atomicity, Consistency, Isolation, Durability) 及 Relational Model

### ACID

對絕大多數的應用來說，交易的正確性跟不可否任性，是系統必要的基石， ACID 便是協助達成這一目標的工具之一。

ACID 雖然是一大重要工具，但也不是不能夠取代，像是 [CQRS Pattern] ， [Cassandra Summit] 有一個很好的講題就是介紹怎麼用 Cassandra 透過 CQRS 來寫一個給銀行用的轉帳系統。

[SAGA Pattern] 則是另一個在沒有 Transaction 下，來解決多事件互動的 Pattern 。

然而 CQRS 或 SAGA Pattern 都沒有 Transaction 好用，以及廣被眾人所了解，連我自己都只是讀過沒有親身實作過，要做的話，連底層的工具都要自己從頭開始作，或者是要去學一些非常新的工具如 [Datomic] 。

因為換出的是 ACID 這樣強大的工具，替代的方案又是高未知的空間，所以許多人最後選擇的替代方案就是 -- **頭埋起來假裝沒有問題**

而且 MongoDB 本身號稱的 BASE 跟本是 Eventual Inconsistency ，在
[Call me maybe: MongoDB] 這篇文章中就驗證到，當 Network Partition 發生時，既使 MongoDB
回復寫入成功，但是實際上還是會失敗的，所以就在這種底層實作不可靠下，你上層加什麼工具也是枉然，所以多數人就是假裝沒這回事，想說自己沒這麼倒楣。

### Relational Model

Relational Model 是三十年前那次 DB 戰爭中，打敗 Network Model, Hierarchical Model 剩出的王者，在三十年後，再次受到 Document Model 的挑戰，這次的結果如何呢？

Relational Model 的好處是可以透過正規化，減少資料的重複、提高資料的一致性，以及透過
SQL ，除了 AdHoc Query 外，還可以對多個 Table 進行 Joined Query，這個是 MongoDB / Document Model 所做不到的。

那這個特性重要嗎？我想透過一個例子就可以很簡單的介紹 Relational Model 的強大，跟 Document Model 在應用上會碰到的問題。

#### Relational Model Example

我們公司是做人肉搜索的，對於同一個人 / email 來說，我們會有很多來自不同來源的 metadata ，今天假設是兩個來源好了。

在 RDBMS 中，就是開兩個 Table ，各自記錄不同來源來的資料

    | TableA        | Email         | likes      |
    | ------------- |---------------| -----------|
    |               | billg@ms.com  |  1600      |


    | TableB        | Email         | tweets     |
    | ------------- |---------------| -----------|
    |               | billg@ms.com  |  9999      |

在 Document Model 可能就是用一個 Document 如

    {
        email: "billg@ms.com",
        likes: 1600,
        tweets: 9999
    }

在 Query 時，前者要一個 join 才可以取回關於這個 email 的所有資料，後者則是一個 lookup ，自然後者的效率較好，而且同樣可以對不同的欄位下 Query 條件。

然而，既然要用 NoSQL 就要玩 **BIG DATA** ，所以我們把問題修改一下，我們從兩個來源，分別拿到十億筆
email 的 likes 數，跟十億筆 email 的 tweets 數，我們想把這些資料寫到 DB 去再拿回來用。

用 RDBMS 自然是輕鬆愉快，兩個檔案輪流打開，一筆筆的加到 DB 中，DBMS對這種簡單的寫入可以做到 10,000 insert / sec 沒問題，大概 55 分鐘就可以做完了。

但是 MongoDB 呢？因為兩個屬性都是在同一個 Document 上，所以要馬是先把十億筆 (email, likes) ，先寫到 MongoDB 中，再下十億筆 Read + Update 把 Document
讀回來再加上 tweets ，如果很不幸你的 Document Id 不是 email ，那就是要下十億筆 Search + Update ，不管怎麼算，都不是太漂亮。

另一個作法是，你先把兩個十億筆的資料在本機的記億體中做 join ，然後再寫到 MongoDB 去，ㄜ... 這樣搞你不覺得累嗎？這麼小的應用問題就要自己動手刻程式碼，而且
前面我們不是說 big data 嗎？如果資料不是十億筆而是一百億筆，你放不進記億體中，那你要先用 radix sort 寫到 file system 再讀回來嗎？

然後你忙了這麼多後， MongoDB 給你的，因為 global write lock ，還是一樣是 10,000 insert /sec ，然後因為 MongoDB 實作的問題，如果你的 Document 有 indexed field
，那麼寫到兩百萬筆後會越寫越慢，這問題在成熟的 DB 上是沒有的。

我想，這個小例子就說明了 Relational Model 的強大之處及 Document Model 弱的地方，當你要既有的資料新增欄位時， Relational Model 是可以很簡單的增加一個
 column 或 table 就做到，在 MongoDB 上，則是要跳過重重關卡才可以達成。

在軟體需求不穩定時，或常有變更時 Relational Model 對變革的適應能力，較 Document Model 高出許多。

## MongoDB 的長處

在交換出 `ACID` 與 `對Model 對變革的適應能力` 後，我們從 MongoDB 到底換回來了什麼呢？

 + cluster 帶來的更多的儲存空間
 + cluster 帶來的更 scalable read operation.
 + 在某個資料大小下，一個非常快速的 index

前兩個長處就不用多談了，Opensource RDBMS的長處並不在這塊，跟從一開始就要做 cluster 的 MongoDB 是沒得比的，MongoDB 目前已有超過 100 node 的單一 cluster。

第三個先略過不說，等說完結論再回來說

## 結論

我們看完了 Trade-off 後，我們要來思考一下，我們這個交易到底划不划算？

對我來說，我們換出了 ACID 跟 `對Model 對變革的適應能力` 後，我們換回來的僅僅只有 Scalable storage & scalable read ，
就我看來，這是一筆極差的交易，因為這個換回來的東西大多只能應用在一塊很小的地方，一個需要大量儲空間、一個需要大量讀取能力的應用，而且後者還可以被 cache 所取代。

前面講的 `在某個資料大小下，一個非常快速的 index` 就是指這個應用範圍， [Navigenics 的 Dick Wall] 就講過，他曾用 MongoDB 用來儲存要配對的 DNA Sequence
，他的資料大小，剛好是大到無法用 Redis 放在記億體中，但是 index 的大小剛好是可以透過 MongoDB 全部放在記億體中，剩餘的資料則是存在硬碟上，就這麼剛好透過
MongoDB ，在下複雜 Query 時，他的搜尋速度可以達到 MySQL 方案的一百倍。

然而，真正 big data 的應用，如 log storage & analysis ，MongoDB 是無法達到對 write operation 的需求的。 MongoDB 能適用的問題空間實在太小，而且既使你目前的需求是在 MongoDB
能達到的範圍內，但需求一變化，就長到 MongoDB 無法承擔的空間去了。

這也是我未來不會採用 MongoDB 的原因，在小量資料時，我會用 PostgreSQL / MySQL ，在資料量大時，會用 Postgres + Cassandra + ElasticSearch

## 附注

既然我說 MongoDB 的這筆交易不是個好交易，那什麼是好交易呢？

我覺得 Cassandra 這個就是個好交易，我們拿 ACID, AdHoc Query 出去換。是的我們把 AdHoc Query 都拿出去換掉了，換回來的是一次只能問兩個問題的 Query ，而且這兩個問題還是在 Table Design 時就要決定的。

但我們換回來的是幾乎無上限的 Read & Write operation 能力 ，在[Netflix 的測試中]，Cassandra 單一節點可以有 10,000 rw/sec ，然後一路隨著增加到 288 個節點，每加一個點可以增加 10,000 rw/sec 。

這個才是我說的好交易，因為這個特殊能力，是其它解決方案都辦不到的；雖然說我們交易掉了 ACID & AdHoc Query ，但是透過適當的 Table Design ， Cassandra 的特性剛好可以用來解決 Big Data 中 log storage & analysis
這個大部份人需要解決的問題。

[Query Language]: http://docs.mongodb.org/manual/tutorial/query-documents/
[MapReduce Query]: http://docs.basho.com/riak/latest/dev/using/mapreduce/
[CQRS Pattern]: http://martinfowler.com/bliki/CQRS.html
[Cassandra Summit]: http://www.youtube.com/watch?v=BGxnjKd4MFQ
[SAGA Pattern]: http://msdn.microsoft.com/en-us/library/jj591569.aspx
[Datomic]: http://www.datomic.com/
[Call me maybe: MongoDB]: http://aphyr.com/posts/284-call-me-maybe-mongodb
[Navigenics 的 Dick Wall]: https://twitter.com/dickwall
[Netflix 的測試中]: http://techblog.netflix.com/2011/11/benchmarking-cassandra-scalability-on.html