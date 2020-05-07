---
layout: post
title: "Mongodb - Schemaless"
date: 2014-02-12 15:08
comments: true
categories: 
---

我覺得大家對 MongoDB 誤解最大的地方就是 schemaless ， schemaless 應該是個大問題才對，不知道為什麼世人都跟 billg 一樣，把問題當做優點來看。

Schemaless 的優點
--------------------

Schemaless 不是沒有優點的，在搭配上動態語言時， Schemaless 是個很好用的功能，不用設定 mapping configuration ，直接就可以打 `item?.some_collections[0]?.city` 就這樣一路對一個物件逛下去，不用為了新增一個欄位就要去改 mapping configuration 然後跑 db migration。

然而，這個特點，只有用在有 safe navigation 的動態語言上，才能享受到這些好處，要不然還是處處都要寫 null check ，要不然就會因為讀到前一版的 item 缺少某個欄位，而噴出了 NullPointerException。

Schemaless 與靜態語言
----------------------

在靜態語言中使用 Schemaless DB 時，schemaless 的好處就消失怠盡了，像是我們公司是用 [Morphia] 來寫 Model & DAO 的，若是要新增一個欄位，免不了是要去更新 Model class 然後重新 compile & deploy 才能加上一個欄位的。

相較於使用 Schemaful DB ，我們省下的，只有跑 DB Migration 的時間，當然，若是你有個超大的 DB ，那這省下的 off-line maintenance downtime 也可是很驚人的，但是，並不是所有的 db migration 都是要把整個系統下線才能跑的，大多數的 db migration 都是會把系統設計成(對DB)向後相容的，所以不用把整個系統啦下線就可以跑。

> 對DB向後相容的 Application / 向前相容的 Schema ：這邊指的是，先不要更新 Application ，先把 DB Migration 跑完，把資料庫的資料跟Schema更新完後，才把新版的 Application 上線。所以在某段時間內，是舊版的 Application 對著新版的 Schema 在跑的。


Schemaless 的問題
---------------------

許多大系統用 Schemaless DB 的原因是因為不想因為跑 DB Migration 把整個系統下線，所以就用 Schemaless 的系統來避開這問題。

但是我想說的是，這誤會可大了，就像前面講的在 Schemaful DB 中，向前相容的 Schema 變動，是不用把系統下線就可以跑的，只有向前不相容的，才要跑 Offline DB Migration 或 Online Data Conversion.

> Online Data Conversion 指的是，因為前後 Schema 不相容，所以開一個新的 Table 給新版本的物件用，每次讀取一個物件時，先去檢查是否已存在新的 Table 中，若沒有，則是從舊的 Table 讀出來，把格式更新，再存到新的 Table 中。
>
> 如此一來，就不用下線也可以把前後格式不相容的的 DB 更新。

前文所說的 db migration & online data conversion 就是 schemaless DB 的大問題，除非你是 **神** ，Model 一次就設計好，可以用一輩子，要不然不跑 db migration ，在你的 DB 中，必然會存在著 V1, V2,V3...Vn 版本的物件。

不跑 db migration ，那就表示你要維護可以對 V1, V2, V3....Vn 存取的 Model class ，你可能有一個 db item 一整年沒被讀取到，但是一年後突然用到，你要怎樣保證還讀的回來呢？？

最常見的做法就是，不去 **大幅度** 更新 Model ，削足適屨，既使你知道更新 Model 會讓你未來程式更好寫，但是因為不能去更新 Model ，那就只好去調整程式邏輯，癟腳的用不合適的 Model 來做新工作。

而既使你想跑 db migration ，但是因為 schemaless db 跟本不打算支援 db migration ，所以不會內建任何工具來幫助你，你只能很苦命的自己刻 online data conversion 的工具了。

後記
----------------

如果你對 DB Migration 的幾種方式有興趣的話，我建議你花一小時，把 Martin Fowler 與 Pramod Sadalage 的 [Agile Database Development] 聽完

[Morphia]: https://github.com/mongodb/morphia
[Agile Database Development]: http://www.se-radio.net/2012/06/episode-186-martin-fowler-and-pramod-sadalage-on-agile-database-development/
