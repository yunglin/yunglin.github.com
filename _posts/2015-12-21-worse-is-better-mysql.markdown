---
layout: post
title: "MySQL 劣即是夯"
date: 2015-12-21 00:42
comments: true
categories: 
---

MySQL transactional lock 在 throughout 上 遠遠不及實作 mvcc 的 PostgrSQL，但在使用上，卻是大大的勝過 pg

原因是因為，在 pg 在寫程式裡，要處理很多因為碰撞產生的 rollback ，即使 db 會幫你處理，但是如果有外部系統呢？是要去招回還是要用 2 phase commit 呢？還有在 UI 上是要讓使用者馬上再試一次，還是請他五分鐘後再試一次了？

所以最簡單的實作就是乾脆把整個 table 鎖住，一次一個人寫入，這樣就不用做例外處理了。

科科科 [劣即是夯](https://gist.github.com/audreyt/3985324#%E5%8A%A3%E5%8D%B3%E6%98%AF%E5%A4%AF) 啊
