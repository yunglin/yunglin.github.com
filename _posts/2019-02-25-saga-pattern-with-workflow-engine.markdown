---
layout: post
title: "使用 Workflow Engine 來實作 Microservices SAGA pattern"
date: 2019-02-25T00:00:00-08:00
---

最近離開了領英，比較有空閒讀讀書，想想到底之前在實作上出了那些的問題，跟要怎麼處理這些問題。

先推薦兩本書 [SOA Patterns](https://amzn.to/2H2g8y3) 和 [Microservices Patterns](https://amzn.to/2Sr593b) ，這兩本書都有提到怎麼在微服務下處理 transaction 的方式，兩本書的方法都是 [SAGA pattern](https://microservices.io/patterns/data/saga.html) + CQRS + idempotent 。

但是如果你有實作過 SAGA Pattern 的話，就會知道使用 Message Driven 的方式在兩個微服務間溝通的話，整個程式碼的可讀性會非常的低，除了原先溝通並實做細節的兩位工程師外，大概不會有第三個人能夠理解整個交易是怎麼完成的。

{% img /images/2019-02-25/Saga_Choreography_Flow.001.jpeg %}
註：本文圖片取自 https://microservices.io/patterns/data/saga.html

在實作細節上，使用MDA的架構，微服務要提供sync & async API使用，是額外的成本，在領英，所有的微服務預設都是只有提供 sync API 的，你要請對方多開一個 async 的界面，是要求爺爺告奶奶的。

因此，在技術上管理上比較可行的，就是把流程的管理都拉到自己管理的微服務當中，在裡面使用 orchestrator 來管理對外乎叫的狀態記錄處理跟錯誤重送。

{% img /images/2019-02-25/Saga_Choreography_Flow.002.jpeg %}

當然，這又免不了要寫一大堆 glue code 把所有的東西黏在一起，然後在資料褲裡面又要開一堆的 Table 來記錄運行的狀況，這非常的花功夫，在實作上很花時間又不好進行修改沒有彈性。

今天早上看到 InfoQ 的一則新聞 [Experiences Moving from Microservices to Workflows at Jet.com](https://www.infoq.com/news/2019/02/migrate-microservices-workflows) 突然腦洞大開，是啊 orchestrator saga 可以使用如 Apache Camel 等的 workflow engine 來實作，微服務還是只需要提供 sync API ，交由 workflow engine 來做狀態追蹤跟錯務重送，然後中間的狀態，可以透過 Camel 存在 ActiveMQ 之中，然後整個流程的定義，可以透過 Camel Java/Scala/… DSL 高階語言來編寫，整個可讀性大增。

      from("file:src/data?noop=true")
          .choice()
              .when(xpath("/person/city = 'London'"))
                  .to("file:target/messages/uk")
              .otherwise()
                  .to("file:target/messages/others");
              
原來我在 2014 年左右就找到答案並且在飛向的程式碼中實作過，只是這幾年我自己被領英的專案壓力追著跑（每一季都同時跑五個以上專案），忙到忘掉了。