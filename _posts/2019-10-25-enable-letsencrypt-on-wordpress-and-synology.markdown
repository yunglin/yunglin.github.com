---
layout: post
title: "Synology上設定 wordpress & let'sencrypt 的方法"
date: 2019-10-25T00:00:00-08:00
---

<!-- wp:paragraph -->
<p>快被 Synology 跟 Comcast 整慘了，正常運作的網站，突然有一天，因為 HTTPS Certificate 的原故連不上了，被整了好幾個小時，所以還是記下怎麼處理好的。<br><br>在 Synology DSM 6.3 上面要正常跑 wordpress &amp; nginx &amp; let's encrypt 的話</p>
<!-- /wp:paragraph -->

<!-- wp:list -->
<ul><li>先安裝 PHP 7.3 ，然後在 Web Station 中新增 PHP 7.3 ，記得要勾選 curl &amp; mysqli extension</li><li>在 wordpress virtualhost 的設定中選用 nginx &amp; php 7.3</li><li>然後到 Control Panel -&gt; Security -&gt; Certificate 的地方新增 Certificate from let's encrypt.</li></ul>
<!-- /wp:list -->

<!-- wp:paragraph -->
<p>如果新增失敗的話，用 ssh 登入，然後跑下面的指令</p>
<!-- /wp:paragraph -->

<!-- wp:quote -->
<blockquote class="wp-block-quote"><p>sudo /usr/syno/sbin/syno-letsencrypt new-cert -d &lt;host&gt; -m &lt;email&gt; -v</p></blockquote>
<!-- /wp:quote -->

<!-- wp:paragraph -->
<p>這樣子會吐出一堆 let's encrypt 給的 debug urls ，把 url 點開，就可以看是那邊出錯了。</p>
<!-- /wp:paragraph -->