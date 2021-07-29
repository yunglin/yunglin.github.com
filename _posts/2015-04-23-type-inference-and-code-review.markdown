---
layout: post
title: "Type Inference 在實務上碰上的問題"
date: 2015-04-23 23:36
comments: true
categories: 
---

最近在讀 **Implementing Domain-Driven Design** 這本書，學習一下別人
是怎麼整理好整個大型軟的架構，並且重新檢視一下我們公司的程式碼的問題。

在 DDD 的概念中，一個模組 **Domain** / **Bounded Context**  應該是一個獨立的觀念，一個大型程式，應該是由許多 **Bounded Context** 組合而成，整個程式的架構應該僅量保持這些的純淨性，**Bounded Context** 的互用，要把上下行的關係定義清楚，程式碼間不應該在一個區塊中引入多個 **Bounded Context**。

在整實作上，如何合檢視一個 class 是否有妥善的處理引用入的 **Domain** ，最簡單也最好用的方式就是看檔頭 **import** 的地方

```
import com.codahale.metrics.Meter

import org.apache.commons.lang.StringUtils
import org.jboss.netty.handler.timeout.TimeoutException
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import com.yunglinho.common.JsonSerializer

import com.yunglinho.app.core.model.Location
import com.yunglinho.app.feature.SocialProfile
import com.yunglinho.app.service.LocationService

import scala.concurrent.Await
import scala.concurrent.ExecutionContext
import scala.concurrent.Future
import scala.concurrent.duration._
```

過去我排 import 的慣用法是按 **有授權疑慮** 的順序去排

  * com.* 的在最上面
  * org, net 排第二
  *  接著是排自己公司的東西，按 common, base, app 的順序去排，common 跟 based 是放 utilities & framework ， app 就是放一個專案的東西了
  * 最後scala & java 的東西

這樣排的好處是， code review 一眼看過去，就可以知道用了那些版權有疑慮的東西，而且 import 時，不會把 java & scala 內建的東西蓋過去，同時也知道這支程式碼用了多少其它的東西

然而，這個機制，在 **Scala's Type Inference** 的影響下，變的很不可靠。

**Type Inference** 是初學 Scala 時，覺得很方便的一個功能，能夠在寫程式時，少打很多的字，例如：

```
   // 在沒有 Type Inference 下， statement 的左右兩邊都要宣告型態
   val location: Location = new Location("Taipei")

   // 左邊的這邊其實是可以省下的
   val location = new Location("Taipei")
   
   // 左右兩邊都沒有 *Location* 的存在，但是卻是被引入在這邊了 
   val location = locationService.lookup("Taipei")
```
這看似方便的功能，卻有著非常深遠且負面的影響，在上面的例子中 *Location*  這類別，雖然在 import 時沒有出現過，但是卻是在 Reviewer 意料之外中，被帶入到了這程式碼中。

當然，在上面的例子中，*Location* & *LocationService* 是一套的觀念，所以沒有真的引入意料之外的 *Domain* 進來，但是如果今天程式碼寫的不好，是有個 *ApplicationContextHolder.getCalendarService* 的話，那就不一樣了。

在一行行 code review 時，有可能會抓到這問題，但是如果你沒有空一行行的看呢？或者是你是在 *refactor* 數千行程式碼時，那要怎樣一行行的去看呢？

所以，設計程式語言真的是很困難，一個看似方便的功能，卻在意料之外的地方造成困擾，讓我最近有點想回去改用 Java 。當然 Scala 還是個很好的語言，能夠幫我增加兩三倍的生產力，不過，卻會讓團隊工作時造成困擾。
