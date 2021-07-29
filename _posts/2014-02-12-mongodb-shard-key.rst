---
layout: post
title: "MongoDB - Sharding"
date: 2014-02-12 14:27
comments: true
categories: 
---


Sharding_ 是一個我非常討厭 MongoDB 的地方，在多台 DB 的解決方案中，免不了要用一個方式來決定那個 object 該放在那台機器上，一個解決方式就是用某個欄位來當 shard key 。

在 MongoDB 中，這個 Shard Key 一決定後就不能改了，然後 MongoDB 會幫你把 object 按 shard key 的值域，平均分配在 n 台機器上。

問題是，平均分派值域並不代表值會是平均非配的，例如 email ，因為姓名是集中在幾個字母上的，所以按值域分派會有 hot spot 產生，a-d 的特別多人，然後 x y z 的沒有人。

在其它的 DB 中，每個 shard 機器所負責的值域是可以讓 dba 去調整的，但 MongoDB 是不行的，所以必然會產生 hot spot 而且你無法去躲開。

有些人會自己去產生 UUID 來當 shard key ，在 MongoDB 2.4 中，則是加入了 hashed shard key 自動幫你把 shard key field 拿去算 hash key 出來當 shard key 用。但是，這還是免不了某些值域是 hot spot 的問題。

既使 MongoDB 能夠平均分派儲存的值域，但是最終我們要看的是，怎麼樣平均分派 read/write operation 。

因為在讀取時，也會發生因為某些原因，讓使用者對某些 object 的存取較其它物件多出 n 倍，所以每個 shard 所負責的值域一定要是 tunable ，要不然遲早一定會出問題。

所以，扔了 MongoDB 吧

.. _Sharding: http://docs.mongodb.org/manual/tutorial/choose-a-shard-key/

