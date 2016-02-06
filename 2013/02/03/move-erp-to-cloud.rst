<!DOCTYPE html>
<html>

  <head>
  <meta charset="utf-8">
  <meta http-equiv="X-UA-Compatible" content="IE=edge">
  <meta name="viewport" content="width=device-width, initial-scale=1">

  <title>中小企業ERP系統上雲端？</title>
  <meta name="description" content="::">

  <link rel="stylesheet" href="/css/main.css">
  <link rel="canonical" href="http://yourdomain.com/2013/02/03/move-erp-to-cloud.rst">
  <link rel="alternate" type="application/rss+xml" title="Your awesome title" href="http://yourdomain.com/feed.xml">
</head>


  <body>

    <header class="site-header">

  <div class="wrapper">

    <a class="site-title" href="/">Your awesome title</a>

    <nav class="site-nav">
      <a href="#" class="menu-icon">
        <svg viewBox="0 0 18 15">
          <path fill="#424242" d="M18,1.484c0,0.82-0.665,1.484-1.484,1.484H1.484C0.665,2.969,0,2.304,0,1.484l0,0C0,0.665,0.665,0,1.484,0 h15.031C17.335,0,18,0.665,18,1.484L18,1.484z"/>
          <path fill="#424242" d="M18,7.516C18,8.335,17.335,9,16.516,9H1.484C0.665,9,0,8.335,0,7.516l0,0c0-0.82,0.665-1.484,1.484-1.484 h15.031C17.335,6.031,18,6.696,18,7.516L18,7.516z"/>
          <path fill="#424242" d="M18,13.516C18,14.335,17.335,15,16.516,15H1.484C0.665,15,0,14.335,0,13.516l0,0 c0-0.82,0.665-1.484,1.484-1.484h15.031C17.335,12.031,18,12.696,18,13.516L18,13.516z"/>
        </svg>
      </a>

      <div class="trigger">
        
          
          <a class="page-link" href="/about/">About</a>
          
        
          
        
          
        
          
        
      </div>
    </nav>

  </div>

</header>


    <div class="page-content">
      <div class="wrapper">
        <article class="post" itemscope itemtype="http://schema.org/BlogPosting">

  <header class="post-header">
    <h1 class="post-title" itemprop="name headline">中小企業ERP系統上雲端？</h1>
    <p class="post-meta"><time datetime="2013-02-03T17:11:00-08:00" itemprop="datePublished">Feb 3, 2013</time></p>
  </header>

  <div class="post-content" itemprop="articleBody">
    ::

 ※ 引述《NightWind (gin and tonic)》之銘言：
 : 我們是一家中小企業
 : 大約三年前才把古早的DOS系統
 : 換裝成鼎新的ERP
 : 現在當初買的伺服器主機保固也要到了
 : 老闆突發奇想把ERP系統裝在中華HiCloud那樣的雲端虛擬主機上
 : 節省定期維護主機與擔心主機突然掛掉的費用
 : 請問真的有人會這樣做的嗎？
 : 又如果有的話，台北地區有沒有規劃的廠商？


HiCloud 沒用過，所以用 AWS_ 的例子來解釋要這麼做會碰上那些例子。

第一個問題是 VM 的可靠度是比你自己買伺服器等級的機器還來的低的， AWS EC2 的
合約是說 99.95% Available ，但是這不包括你 VM 被 AWS會有隨機關掉的機會。當
被關掉後，你是要自己手動或自動重啟才有 99.95%

第二個問題是，台灣多數的 ERP是2-tier的架構而非 3 tier 的，若是用 2-tier 的
軟體，使用者端要從本機連到 AWS去，一個來回的時間從 <10ms跳到 100ms 以上，如
果程式寫的不好，本來一個流程的轉換要跑兩個 SQL command，原來是 (10ms*2 + SQL execution time) * 2，
還有機會是小於 100ms看起來很即時就可以反應，一搬到雲端，時間就跳昇到 100ms * 4，變的超級鈍。

再來，放到 AWS 去，還要考慮台美專線速度的問題，過去幾年來，台灣對外的海纜，幾乎
每兩年都會斷線超過一天。所以放在雲端你的架構能提供的，就只剩不到 99.5%

在 EC2 上，如果要用 EBS讓 VM 當機後資料還在的話， EBS的效能是有口皆呸的
，但是如果不用 EBS，那 VM 一死，上面的資料又全消失了，變成你自己要做 Data Replication

最後，如果要用台灣的雲端，不用要 HiCloud，比較建議的環境會是開發資源及支援較多
的 `MS Azure`_ 或是使用 joynet_ 技術的 MiCloud_ 會比較妥當。

縱觀來講，搬到雲端有其它新的挑戰要處理，第一個看軟體要先 3tier 化，再來是要
做DB online replication 保護資料，最後，是公司對外的專線要能跟的上資料量。

如果前述問題都能解決的話，貴公司得到的會是

1. 更短的錯誤迴復時間：伺服器一死，跟廠商搬機器來迴復也要一個工作天以上，用雲端可以降到一兩小時

2. 更敏捷的硬體效能：如果硬體效能不夠，使用雲端系統，是直接選更高等級的VM重開，當下就可以昇級。
   不用等採買硬體所需的數星期到數個月的時間，也不必擔心折舊週期還沒到就又要昇級。


在是否該選用雲端方案的話，就看你們目前的 ERP是否是 3-tier 的，另外就是找顧問來來編
寫相關的工具及作業流程。

.. _AWS: http://aws.amazon.com
.. _MS Azure: http://www.microsoft.com/taiwan/windowsazure/
.. _joynet: http://joyent.com/
.. _MiCloud: http://micloud.tw/

  </div>

</article>

      </div>
    </div>

    <footer class="site-footer">

  <div class="wrapper">

    <h2 class="footer-heading">Your awesome title</h2>

    <div class="footer-col-wrapper">
      <div class="footer-col footer-col-1">
        <ul class="contact-list">
          <li>Your awesome title</li>
          <li><a href="mailto:your-email@domain.com">your-email@domain.com</a></li>
        </ul>
      </div>

      <div class="footer-col footer-col-2">
        <ul class="social-media-list">
          
          <li>
            <a href="https://github.com/jekyll"><span class="icon icon--github"><svg viewBox="0 0 16 16"><path fill="#828282" d="M7.999,0.431c-4.285,0-7.76,3.474-7.76,7.761 c0,3.428,2.223,6.337,5.307,7.363c0.388,0.071,0.53-0.168,0.53-0.374c0-0.184-0.007-0.672-0.01-1.32 c-2.159,0.469-2.614-1.04-2.614-1.04c-0.353-0.896-0.862-1.135-0.862-1.135c-0.705-0.481,0.053-0.472,0.053-0.472 c0.779,0.055,1.189,0.8,1.189,0.8c0.692,1.186,1.816,0.843,2.258,0.645c0.071-0.502,0.271-0.843,0.493-1.037 C4.86,11.425,3.049,10.76,3.049,7.786c0-0.847,0.302-1.54,0.799-2.082C3.768,5.507,3.501,4.718,3.924,3.65 c0,0,0.652-0.209,2.134,0.796C6.677,4.273,7.34,4.187,8,4.184c0.659,0.003,1.323,0.089,1.943,0.261 c1.482-1.004,2.132-0.796,2.132-0.796c0.423,1.068,0.157,1.857,0.077,2.054c0.497,0.542,0.798,1.235,0.798,2.082 c0,2.981-1.814,3.637-3.543,3.829c0.279,0.24,0.527,0.713,0.527,1.437c0,1.037-0.01,1.874-0.01,2.129 c0,0.208,0.14,0.449,0.534,0.373c3.081-1.028,5.302-3.935,5.302-7.362C15.76,3.906,12.285,0.431,7.999,0.431z"/></svg>
</span><span class="username">jekyll</span></a>

          </li>
          

          
          <li>
            <a href="https://twitter.com/jekyllrb"><span class="icon icon--twitter"><svg viewBox="0 0 16 16"><path fill="#828282" d="M15.969,3.058c-0.586,0.26-1.217,0.436-1.878,0.515c0.675-0.405,1.194-1.045,1.438-1.809c-0.632,0.375-1.332,0.647-2.076,0.793c-0.596-0.636-1.446-1.033-2.387-1.033c-1.806,0-3.27,1.464-3.27,3.27 c0,0.256,0.029,0.506,0.085,0.745C5.163,5.404,2.753,4.102,1.14,2.124C0.859,2.607,0.698,3.168,0.698,3.767 c0,1.134,0.577,2.135,1.455,2.722C1.616,6.472,1.112,6.325,0.671,6.08c0,0.014,0,0.027,0,0.041c0,1.584,1.127,2.906,2.623,3.206 C3.02,9.402,2.731,9.442,2.433,9.442c-0.211,0-0.416-0.021-0.615-0.059c0.416,1.299,1.624,2.245,3.055,2.271 c-1.119,0.877-2.529,1.4-4.061,1.4c-0.264,0-0.524-0.015-0.78-0.046c1.447,0.928,3.166,1.469,5.013,1.469 c6.015,0,9.304-4.983,9.304-9.304c0-0.142-0.003-0.283-0.009-0.423C14.976,4.29,15.531,3.714,15.969,3.058z"/></svg>
</span><span class="username">jekyllrb</span></a>

          </li>
          
        </ul>
      </div>

      <div class="footer-col footer-col-3">
        <p>Write an awesome description for your new site here. You can edit this line in _config.yml. It will appear in your document head meta (for Google search results) and in your feed.xml site description.
</p>
      </div>
    </div>

  </div>

</footer>


  </body>

</html>
