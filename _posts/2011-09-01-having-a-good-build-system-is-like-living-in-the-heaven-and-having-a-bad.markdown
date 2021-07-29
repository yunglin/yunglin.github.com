---
layout: post
title: "Having a good build system is like living in the heaven, and having a bad....."
date: 2011-09-01T11:09:00-07:00
comments: false
---

<!--
  ~ Copyright (c) 2012. Blue Tang Studio LLC. All rights reserved.
  -->

<div class='post'>
Having a good build system is like living in the heaven, and having a bad build system is like living in the hell. I am just tired of dealing with all kind of build system. In short, all build systems are either suck or sucks more.<br /><br />Yesterday, I spent 4 hours to help a coworker to deal with library projects in Android. It wasn't a pleasure experience. Today, I spent another 4 hours to figure out how to configure an integration-test phase in SBT.<br /><br />Just to save you some time, here is how to do it with SBT's quick configuration DSL, build.sbt.<br /><br /><pre>import sbt._
<br />
<br />{
<br />    lazy val IntegrationTest = config("it") extend(Test) extend(Provided) extend(Optional)
<br />    lazy val itSettings: Seq[Project.Setting[_]] =
<br />        inConfig(IntegrationTest)(Defaults.testSettings) ++
<br />        Seq(
<br />             ivyConfigurations += IntegrationTest
<br />        )
<br />    seq(itSettings : _*)
<br />}
<br /></pre><br />That is all you need. The first line in the block defines a new configuration, it. The 'it' configuration extends all the dependencies from 'Test'. And the fifth line exposes this configuration to ivy and sbt.<br /><br /></div>
