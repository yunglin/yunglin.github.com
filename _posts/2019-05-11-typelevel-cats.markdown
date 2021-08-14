---
layout: post
title: "typelevel cats 學習筆記"
date: 2019-05-11T00:00:00-08:00
---

最近在學 typelevel/cats 底下這一行程式碼，想了兩天才理解為什麼要寫成這樣，Functional Programming真的是太困難了，有點回到大一學遞迴的感覺。

![img](/images/2019-05/IO-async.png)


`(Either[Throwable, A]) => Unit)` 是一個 Callback function / Listener interface ， `(CB => Unit)` 表示這個 Callback function 可以在不同的 Thread 執行。

為什麼要寫成這樣？我還在想….