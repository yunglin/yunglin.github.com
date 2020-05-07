---
layout: post
title: "Data Modeling With Jackson Json and Scala - lazy val"
date: 2011-11-15T08:50:00-08:00
comments: false
---

<div class='post'>
Scala中有個好用的功能是 lazy val - lazy initialized variable ，然而，Scala在實作這功能時，是會加入一個<br /><pre class="brush: java">public volatile int bitmap$0;</pre><br />若是這欄位沒有被從生成的 json 中替除的話，那麼，讀回來的 scala object 中，所有的lazy val，都會仍是 null ，不會被初始化。<br /><br />這個問題，只有做好unit-testing可以避免，因為這個欄位是 public 的，所以不需要setter or constructor， Jackson 就可以直接把值設定進去，只用unit-test仔細檢查生成的，確定沒有預料之外的欄位被寫入到 json 中，才可以免去後來在 runtime 發生不可預料的問題。<br /><br />至於怎麼去除這欄位呢，只要在class上標上<code>@JsonIgnoreProperties({"bitmap$0"})</code>即可</div>
