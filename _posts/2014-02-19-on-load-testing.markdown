---
layout: post
title: "從壓力測試回想到一些老故事"
date: 2014-02-19 00:55
comments: true
categories: 
---

最近壓力測試因為 [戶政系統] 的原因，上了幾天的新聞，講到壓力測試，就想到我這輩子職場上的一個恥辱，把一個產品搞倒的紀錄。

2007 年我還在 Rhapsody.com 時，當時我是擔任我們 webservice api 的 QA Lead ，做一個有 40 個內外部客戶的服務的守門員，每一次的產品更新，都要我跑完了數萬個測試，說可以出貨了才可以出貨。

四月時忙完跟 Tivo 的合作案，把系統的負荷量測到及調整到可以服務 500萬線上用戶的量後，另一個團隊又開發了新的 social 網站。

那是 facebook 才上線三年，或者說才對外公開服務的第一年，當所有人都還不知道什麼是社交功能時，rhapsody就已經開發出來社交功能，可以互相推薦音樂清單、可以 follow 別人、還可以自定 avatar 

在上線前，我問了我的主管的主管，需不需要對 social web 的上線做測試，當時大主管是說不用。沒想到一上線，這 social web 非常的熱門，一下子就有數萬個用戶從標準的頁面轉換到 social 版去，似乎，一個新的 killer app 就要產生了。

然而才上線兩三天，網站的速度就慢了下來，後來才知道，原來 social web 濫用了我們的一個 GET api ，我們的 GET api 允許一次抓多個 object ，結果 social web 一次一個 request 就抓了 500 object ，一個 object 4k 就好，一次就抓 2MB ，在加上 object serialization 中間所花的記憶體空間，一個 request 就要佔 6MB 以上，一下就把那個年代的 server 打垮了。

就這樣，這個 social web 就下線整修，但不知為什麼，就此沒有再上線過了....

[戶政系統]: http://www.cna.com.tw/news/aipl/201402180446-1.aspx
