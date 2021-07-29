---
layout: post
title: "My experience with Scala on Android"
date: 2011-11-16T00:53:00-08:00
comments: false
---

<div class='post'>
用Scala開發Android，是個滿有趣的經驗，大致上來因為我切入的時間點較晚，所以大部份的問題已經被前人所解決，就用 https://github.com/jberkel/android-plugin 把專案用 sbt 開好後，就可以開始寫 android.<br /><br />我用的 IDE 是 intellij 11 EAP，用 sbt-idea 把 idea project 設好後，把預設的 asset pa th 從 .idea_module 改成 src/main ，就可以開始開發了。<br /><br /><div class="separator" style="clear: both; text-align: center;"><a href="http://4.bp.blogspot.com/-OQlhhS6AeRY/TsNxXNa-1FI/AAAAAAAAC9A/H23HPm8vOy4/s1600/Screen%2Bshot%2B2011-11-16%2Bat%2B4.15.56%2BPM.png" imageanchor="1" style=""><img border="0" width="600" src="http://4.bp.blogspot.com/-OQlhhS6AeRY/TsNxXNa-1FI/AAAAAAAAC9A/H23HPm8vOy4/s1600/Screen%2Bshot%2B2011-11-16%2Bat%2B4.15.56%2BPM.png" /></a></div><br />在開發上，除了前一回碰上的<a href="http://blog.yunglinho.com/2011/11/data-modeling-with-jackson-json-and_5795.html">proguard問題</a>外，我還碰上另一個比較嚴重的問題－不能在Scala裡寫 AsyncTask<br /><br />這問題跟<a href="https://issues.scala-lang.org/browse/SI-3622">SI-3622</a> <a href="https://issues.scala-lang.org/browse/SI-3494">SI-3494</a>有關，看來是在 Scala 2.8解掉的問題，2.9又跑回來了，我這邊看到的狀況是<br /><br /><pre class="brush: scala">override protected def doInBackground(params: Params*): Result<br /></pre><br />會被Scala compiler專換成<br /><pre class="brush: java">public Result doInBackground(Seq params)<br />    public Result doInBackground(Params[] params)<br /></pre><br /><br />而底下的code，則是scala compiler會吐出 overides nothing.<br /><pre class="brush: scala">override protected def doInBackground(params: Array[Params]): Result<br /></pre><br />不管怎樣，都跟Android要求的<code>protected Result doInBackground(Params[] params)</code>不同，所以在runtime時會跑出NoSuchMethodError.<br /><br />解決方法是在 java 裡寫個 bridge interface ，幫 scala compiler 搞不定的東西，在這個 interface 裡定意好<br /><br /><pre class="brush: java">public abstract class SAsyncTask&lt;Params, Progress, Result> extends AsyncTask&lt;Params, Progress, Result> {<br /><br />    protected abstract Result doInBackground(Seq&lt;Params> params);<br /><br />    @Override<br />    protected Result doInBackground(Params... paramses) {<br />        return doInBackground(JavaConversions.asScalaBuffer(Arrays.asList(paramses)));<br />    }<br />}<br /></pre></div>
<h2>Comments</h2>
<div class='comments'>
<div class='comment'>
<div class='author'>yunglin</div>
<div class='content'>
多補幾個有關repeated parameters的討論，看來是Spec本身有問題，然後實作時一改再改，一路從2.7開始修修補補過來<br /><br />https://issues.scala-lang.org/browse/SI-1342<br />https://issues.scala-lang.org/browse/SI-1459<br />http://stackoverflow.com/questions/5512402/overriding-a-repeated-class-parameter-in-scala</div>
</div>
</div>
