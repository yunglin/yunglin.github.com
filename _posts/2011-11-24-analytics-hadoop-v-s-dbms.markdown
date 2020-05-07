---
layout: post
title: "Analytics: Hadoop V.S. DBMS"
date: 2011-11-24T00:55:00-08:00
comments: false
---

<div class='post'>
最近兩周，我開始參加 <a href="http://www.blogger.com/www.taipei-gtug.org/">GTUG</a> 的活動，在活動間，有人問起，為什麼我公司選用DBMS(MySQL)來做資料分析，而不用 Hadoop。在這邊，我把跟公司的DB怪才同事的討論結果，稍為簡單敘述一下。<br /><br />首先，要先理解到的一點是，在 MapReduce 及 Hadoop 出現之前，Data Mining的工具及技術，已經在DBMS上建構了有30年了，相關的技術也發產的很成熟且有許多專書論述。<br /><br /><a href="http://en.wikipedia.org/wiki/Extract,_transform,_load">ETL(Extract, Transform, Load)</a>是這個領域的關鍵字，用 ETL 去找下去，會找到許多的書籍講怎麼做資料分析<br /><ul><li>Extract: 如何在雜亂的原始資料中，找出有用的訊號，並把這些訊號歸檔<br />例如：把 Apache 的 access.log 轉換成一對對的 (timestamp, session) </li><li>Transform: 如何把整理好的訊號，轉換成更好用的中介查詢表格<br />例如：把前面的 (timestamp, session) 轉換成以小時為單位的存取次數 (time-slice, count) </li><li>Load: 把整理好的資料，存到資料庫中。 <br /></li></ul>透過這個轉換過的Fact Table，我們可以把很昂貴的『每日訪問總數查尋』從 <br /><pre class="brush: sql"># pseudo code 應該是錯的，不過你知道我在表達什麼就好了 :P<br /><br />SELECT COUNT(UNIQUE session) FROM http_access<br />  GROUP BY DATE_FORMAT(timestamp,'%Y %m %d')<br /></pre>變成 <br /><pre class="brush: sql">SELECT SUM(count) FROM access_counts<br />  GROUP BY DATE_FORMAT(time_slice,'%Y %m %d')<br /></pre><br />假設，你的原始資料有一整年這麼多，總共有四千萬筆(40M)，那麼，前者就是把四千萬筆的資料讀出來，然後過濾加總，而後者，由於資料匯入時已先處理過了，所以要處理的只有24 * 365筆資料；而這兩者間所花費的時間差易，會隨著查尋次數而產生重大差異。&nbsp; <br /><br />試問，若是每次你老闆跟你要資料時，你要跟他說，要等半小時才可以，還是說等我五分鐘？  至於前面的半小時是怎麼算出來的呢？<br /><br />由於DB運算，瓶頸多是卡在硬碟存取上，在前面的例子中，40M筆資料，假設一筆是1K bytes，那麼，總資料量是 40GB ，從硬碟循序讀取 40GB 出來，所花的時間是 40GB / 40MB/s(硬碟速度) = 17分鐘  <br /><br /><h3>MapReduce & DBMS</h3><br />MapReduce是把資料分拆運算，然後再加總的技術；但是，MapReduce的概念對 RDBMS 並不是什麼新奇的事，在<a href="http://publib.boulder.ibm.com/infocenter/db2luw/v8/index.jsp?topic=/com.ibm.db2.udb.doc/admin/c0005292.htm">這份IBM的文件</a>中，便清楚的敘述到，DBMS會對 SQL 指令做編譯，DBMS內部就會依資料型態、INDEX檔、CPU Core的數量等，去做最佳化，這些，是Hadoop不會幫你做的。 <br /><br />既然DBMS那麼好那麼，為什麼會有 MapReduce 等技術的出現呢，那是因為，DBMS可以幫你scale up，但是最終，還是會碰上硬體的極限，那麼，這時只有 scale out ，才能解決這問題。  但是 scale out 的程式難度，總是比 scale up 困難，而且，一個不好的分析模式，並不會因為跑的快10倍，就變得比較好；因此， ETL 的概念，在使用 MapReduce 時仍是可以應用的，<br /><br /><br />講到這邊，什麼時候該用 Hadoop ，什麼時候該選用 DBMS ，應該很清楚了<br /><ol><li>首先，就是要先看你要找的分析訊號是什麼，定好分析的步驟及查詢表格</li><li>再來，看你的資料量會有多大，做些簡單的計算，看看是不是能在合理的時間內跑完</li><li>最後，看你這些報表倒底是有多常被取用，若是，這是內部的報表，只要一個月能出一次就好，那麼，Hadoop可能就大才小用；若是，你是要做產品，像Google Analytics做到即時大量分析，那麼你可能就需要用到Hadoop<br /><br />過去我在 <a href="http://medio.com/">Medio</a> 服務時，公司有幫美國幾大電信商分析 App 的購買行為，由於，這些是每個月才需產生一次的報表，而且，資料的訊號有待我們的 data expert 去尋找，因此，我們的資料專家是把資料倒到 MicroStrategy 再去下指令分析的，並沒有用到 Hadoop</li></ol></div>
<h2>Comments</h2>
<div class='comments'>
<div class='comment'>
<div class='author'>Sam Wong</div>
<div class='content'>
我不會Hadoop…想請教，用Hadoop不也得ETL嗎？<br /><br />也許Fact Table的設計可以節省 - 以Scale out MapReduce的bruteforce來換取事前精準按business requirement設計fact table schema - 但Hadoop也不能當OTLP用吧？Hadoop應該沒有ACID的能力吧？</div>
</div>
</div>
