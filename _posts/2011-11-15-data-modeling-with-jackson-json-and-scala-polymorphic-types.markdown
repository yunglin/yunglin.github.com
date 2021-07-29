---
layout: post
title: "Data Modeling With Jackson Json and Scala - Polymorphic Types"
date: 2011-11-15T08:21:00-08:00
comments: false
---

<div class='post'>
關於 Jackson Json 對多型的支援，原開發者已經有一篇 <a href="http://www.cowtowncoder.com/blog/archives/2010/03/entry_372.html">blog</a> 很清楚的解釋該怎麼做，因此，我只寫下在 scala 上面要注意的眉角。<br /><br />首先，我先寫下理想中，Jackson在Scala上該怎麼使用，在來談，為什麼不能這麼用<br /><pre class="brush: scala">@JsonTypeInfo(use = JsonTypeInfo.Id.NAME, include = JsonTypeInfo.As.PROPERTY, property = "type")<br />@JsonSubTypes(Array(<br />     new JsonSubTypes.Type(name = "flood", value = classOf[Flood]),<br />     new JsonSubTypes.Type(name = "mountainslide", value = classOf[MountainSlide])<br />)) <br />trait Disaster {}<br /><br />case class Flood @JsonCreator()(<br />    @BeanProperty @JsonProperty("event") event: String,<br />    @BeanProperty @JsonProperty("address") address: String<br />) extends Disaster<br /><br />case class MountainSlide @JsonCreator()(<br />    @BeanProperty @JsonProperty("event") event: String,<br />    @BeanProperty @JsonProperty("landmark") landmark: String<br />) extends Disaster<br /><br /></pre><br />在 scala 2.8.0 上面，我碰上的問題是<a href="https://issues.scala-lang.org/browse/SI-2764">不能在annotation內使用 java enum</a> 如： <code>use = JsonTypeInfo.Id.NAME</code> <br /><br />在 scala 2.9.1 上，上面的問題消失了，不過碰上另一個問題，當一個 annotation 上面有兩個 java enum，scala compiler 及 java compiler 都不會吐出錯誤，但是annotations從bytecode中消失了，<a href="https://issues.scala-lang.org/browse/SI-5165">SI-5165</a>中紀錄了這問題。<br /><br />不管怎樣，在現在這時間，若是你想用Jackson + Scala來處理多型，那麼，最頂層類別，一定要用Java來寫，才不會碰上上述兩個問題。</div>
