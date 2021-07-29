---
layout: post
title: "Dependency Injection in Scala"
date: 2012-04-22 22:44
comments: true
categories: 
---

這是 4/23 要在 Scala Taiepi Meeting 3 用的講稿，先打出來放這。


Dependency Injection & Inversion of Control
=============================================

`Dependency Injection & Inversion of Control`_ 是 Martin Fowler 在 2004 年所
提出來的一個的概念，Martin Fowler在這篇文章中指出，DI可以三種型式來實作，這觀念後來由 Spring
Source 及 Google 實做出來，變成 Java Enterprise 應用中，不可或缺的一塊。

在 Ioc 觀念的出現前，物件相依的元件(Component)，多是於物件建立時，就帶入(binding)的，Martin
的供獻在於，IoC把物件對元件的相依性拆出來，變成可以替換的實作。

底下是 Martin 所舉的例子，接著我們會看到用 DI 改進的方式

.. code-block:: java

  class MovieLister {
    private MovieFinder finder;
    public MovieLister() {
      finder = new ColonDelimitedMovieFinder("movies1.txt");
    }
  }




Martin 提出的 DI 型式分別是 Constructor Injection, Setter Injection, and Interface Injection.

Constructor Injection
---------------------

.. code-block:: java

  class MovieLister...
    public MovieLister(MovieFinder finder) {
        this.finder = finder;
    }


Setter Injection
-----------------

.. code-block:: java

  class MovieLister...
    private MovieFinder finder;

     public void setFinder(MovieFinder finder) {
      this.finder = finder;
     }
  }


Interface Injection
-------------------

.. code-block:: java

  public interface InjectFinder {
    void injectFinder(MovieFinder finder);
  }

  class MovieLister implements InjectFinder...
    public void injectFinder(MovieFinder finder) {
      this.finder = finder;
    }

  class Tester...
    private void registerComponents() {
      container.registerComponent("MovieLister", MovieLister.class);
      container.registerComponent("MovieFinder", ColonMovieFinder.class);
    }

    private void registerInjectors() {
      container.registerInjector(InjectFinder.class, container.lookup("MovieFinder"));
      container.registerInjector(InjectFinderFilename.class, new FinderFilenameInjector());
    }
  }

.. _Dependency Injection & Inversion of Control: http://martinfowler.com/articles/injection.html


JSR-330_ Annotation
-------------------

而後，這些 DI 的用法，也被 J2EE 給採用，變成 JSR-330_ 標準，用法如下


Constructor Injection
^^^^^^^^^^^^^^^^^^^^^

.. code-block:: java

  class MovieLister...
    @Inject public MovieLister(MovieFinder finder) {
        this.finder = finder;
    }

  public interface MovieFinder {
      List findAll();
  }

Property Injection
^^^^^^^^^^^^^^^^^^^

.. code-block:: java

  class MovieLister {
    @Inject private MoveFinder finder;
  }

.. _JSR-330: http://docs.oracle.com/javaee/6/api/javax/inject/package-summary.html


Dependency Injection in Scala
=============================

花了許久時間解釋 DI 於 Java 的演進，我們總算可以進入本文的正題 Dependency Injection in Scala。在 Scala 中
實做 DI 的方式有有那些呢，接下來我們要談的就是 DI in Scala 的幾個選擇。

- JSR-330_
- Cake Pattern
- SubCut
- Functional DI - Reader Monad


JSR-330_
-------------

由於 Scala 本身在編譯時，會被編成 Java Byte Code ，所以可以直接套用支源 JSR-330_ 的
框架，但是在使用上有一些小眉角，就是，若是使用 Constructor Injection 時，需要用 `@Inject()` 標
在 class 定義之前，使用 Property Injection 時，由於 Scala 的 `val` 被定義成 final 的，所以不
能夠被用在 Property Injection 之上，在這時候，只能用 `var` 來做。

為了達到 immutable 的效果，我通常都是用 Constructor Injection 的。( 另一個原因是我不喜歡 AOP )


.. code-block:: scala

  case class @Inject() MovieLister(finder: MovieFinder)

  class MovieLister2 {

    var finder: MovieFinder = _

  }


Cake Pattern
------------

Cake Pattern 大概是除了 JSR-330 外，最常被提到在 Scala 下的實作 DI 的方式了，Cake Pattern 是由
Scala 之父 Martin Odersky 的一篇論文 `Scalable Component Abstractions`_ 中提到，不過大
多數人看過的版本，應該是 Jonas Bonér 的 `Real-World Scala: Dependency Injection (DI)`_

Cake Pattern 是利用 Scala Mixin 的功能，讓物件被創造時，才把相依的元件，透過 Mixin 的方式綁在一起。

.. _Scalable Component Abstractions: http://lamp.epfl.ch/~odersky/papers/ScalableComponent.pdf
.. _Real-World Scala\: Dependency Injection (DI): http://jonasboner.com/2008/10/06/real-world-scala-dependency-injection-di/


接下來，我們就看看要怎麼樣用 Cake Pattern 重新實作上面的 MovieFinder 的例子

.. code-block:: scala

  trait MovieFinderComponent {

    def finder: MovieFinder

    trait MovieFinder {
      def findAll: List[Movie]
    }
  }

  trait NilMovieFinderComponent extends MovieFinderComponent {

    val finder: MovieFinder = new NilMovieFinder

    class NilMovieFinder extends MovieFinder {
      def findAll: List[Movie] = {
        Nil
      }
    }
  }

  abstract class MovieLister {
    this: MovieFinderComponent =>

    def findByAuthor(author: String): List[Movie] = {
        finder.findAll.filter(m => m.author == author)
    }
  }

上面便是一個使用 Cake Pattern 實作 MovieFinder 所需要的程式碼；首先，我們要把所需的
元件，包在一個 Component trait (MovieFinderComponent) 裡頭，再把這個原元件的規格，
寫在 Component trait 中的另一個 trait 中(MovieFinder)，最後，再是訂一個取存這元件的
method `def finder: MovieFinder`.

在被注入的這一方 (MovieLister) ，我們使用 self-type annotation 來宣告，當要生成一個
MovieLister instance 時，我們必需要提供 MovieFinderComponent 的實作，同時，在MovieLister
內，我們可以透過 `finder` 這個 method 來呼叫 MovieFinder 的實作 ；這邊的寫法是

.. code-block:: scala

  val movieLister = new MovieLister extends NilMovieFinderComponent


Component Registry
^^^^^^^^^^^^^^^^^^

在應用上，由於我們的系統可能包含不只一個元件，而一個物件，可能須要多個不同的元件，因此，
Jonas建議我們，如　Guice 用 Module 來定義所有元件的實作類別，在Cake Pattern上，我們
可以用一個 ComponentRegistry Object 來把所有的實作類別指定好。

.. code-block:: scala

  object ComponentRegistry extends
    MyMovieFinderComponent with
    MyAuthorFinderComponent with
    MyUserRepositoryComponent


  object TestEnvironment extends
    MockMovieFinderComponent with
    MockAuthorFinderComponent with
    MockUserRepositoryComponent


  class Test {
    def testList {
      new lister = new MovieLister extends TestEnvironment
      // testing code.
    }
  }

.. note:: 一個動動腦的時間，在使用 ComponentRegistry 時，我們要怎麼寫，才會讓某一個 Component 的實作變成 Singleton


Pros and Cons of Cake Pattern
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Pros
 - no framework required, using only language features
 - type safe – a missing dependency is found at compile-time
 - powerful – “assisted inject”, scoping possible by implementing the dependency-providing method appropriately
Cons
 -  lot of boilerplate code


Problem of Cake Pattern
^^^^^^^^^^^^^^^^^^^^^^^

然而，從我的使用經驗來講， Cake Pattern 有個缺點，讓我不推薦各位來使用 Cake Pattern ，缺點是， Cake Pattern
只能使用在第一層相依性的元件上，造成實作上的程式碼的不一致性，以及可重用性的問題。

今天，假設我們電影的數量一直成長，所以我們開始分門別類來存放這些電影的類別，而我們在找作者時，只在各個子類別下
找；為了這個變更，我們不只需要新增一些元件，我們連舊有的 MovieLister 都需要更改他的實作才行。

.. code-block:: scala

  trait CategorizedMovieFinderFactoryComponent {

    trait  CategorizedMovieFinderFactory {
      def create(category: String): MovieFinder
    }
  }

  abstract class MovieListerFactory {
    this: CategorizedMovieFinderComponent

    def create(category: String): MovieLister = new MovieLister(finder)
  }

  // WTF, the implementation has to change here?!
  case class MovieLister(finder: MovieFinder) {
    def findByAuthor(author: String): List[Movie] = {
        finder.findAll.filter(m => m.author == author)
    }
  }




SubCut_
========

SubCut_ 是由 Dick Wall 這位 Java Posse Podcaster 所完成的專案，由於我個人並沒有實際使用的經驗
，所以只能就他所提供的文件做個概述。

.. code-block:: scala

  object SomeModule extends NewBindingModule({ implicit module =>
    import module._  // optional but convenient

    bind [ServiceA] toSingle Y
    bind [Z] toProvider { codeToGetInstanceOfZ() }
  })

  class SomeService(param1: String, param2: Int)(implicit val bindingModule: BindingModule)
      extends SomeTrait with Injectable {

      val service1 = inject [ServiceA]
  }

在 SubCut 中，是讓需要被注入的元件庫，使用 implicit variable 的方式，從執行環境中帶入，然後直接讓
SomeService對整個 Module 做存取。

然而，因為 SubCut 的這一個缺陷，我個人是不用去使用 SubCut 的，這個缺點，在 Martin Fowler 文中被稱
做 `Service Locator`_ ，使用 Service Locator Pattern 的缺點是，你把整個環境都傳給了需要被注入的物
件，讓物件自己在環境中去挖寶；這正好就把把 Inversion of Control 的拿交出的控制權又還給了物件本身；
除了去閱讀程式碼外，你無法單看 class constructor 就可以了解到，某一個物件到底是相依在那些元件之上。

好不容易爭來的控制權，又還了一大半回去

.. _SubCut: https://github.com/dickwall/subcut/blob/master/GettingStarted.md
.. _Service Locator: http://martinfowler.com/articles/injection.html#ServiceLocatorVsDependencyInjection

Reader Monad
============

跳脫了從 OO 出發而來的 DI 方式， Runar Oli 在 Northeast Scala Symposium 2012上 ，提出了一個
純用 `Functional Programming 的 DI 實作`_ (slides_)


.. _Functional Programming 的 DI 實作: http://www.youtube.com/watch?v=ZasXwtTRkio
.. _slides: http://dl.dropbox.com/u/4588997/Runar-NEScala-DI.pdf

Runar 用的例子是一個使用 JDBC 的範例

.. code-block:: scala

  def setUserPwd(id: String,
                 pwd: String,
                 c: Connection) = {
    val stmt = c.prepareStatement(
      "update users set pwd = ? where id = ?")
    stmt.setString(1, pwd)
    stmt.setString(2, id)
    stmt.executeUpdate
    stmt.close
  }


上面的程式碼，我們可以用 Functional Way 改寫成底下的方式，讓 `setUserPwd` 改成
回傳一個 (Connection => Unit) 的函式，這個函式，收進一個 DB Connection 然後對
這個 Connection 做一些操作。

.. code-block:: scala

  def setUserPwd(id: String,
                 pwd: String): Connection => Unit =
    c: Connection => {
      val stmt = c.prepareStatement(
        "update users set pwd = ? where id = ?")
      stmt.setString(1, pwd)
      stmt.setString(2, id)
      stmt.executeUpdate
      stmt.close
  }


接著，我們可以用一個 DB Monad 把所有 DB Operation 都抽像化，讓這些 DB Operation 可
以堆疊起來。

.. code-block:: scala

  case class DB[A](g: Connection => A) {
    def apply(c: Connection) = g(c)

    def map[B](f: A => B): DB[B] =
      DB(c => f(g(c)))

    def flatMap[B](f: A => DB[B]): DB[B] =
      DB(c => f(g(c))(c))
  }

  def pure[A](a: A): DB[A] =DB(c => a)

  implicit def db[A](f: Connection => A): DB[A] = DB(f)

底下是堆疊起來的成果，透過 scala `for comprehension`_ 把 getUserPwd, setUserPwd 堆
疊起來，變成一個 changePwd method ，這一步步的把函式加成，符合了 Functional Programming 中
的 no side-effect ，在函式加成的過程中，我們並沒有去執行任何有 side effect 的呼叫，只是單純的
把函式加乘起來，等到最後再來選定執行環境。

.. code-block:: scala

  def changePwd(userid: String,
                oldPwd: String,
                newPwd: String): DB[Boolean] =
  for {
    pwd <- getUserPwd(userid)
    eq <- if (pwd == oldPwd) for {
            _ <- setUserPwd(userid, newPwd)
          } yield true
          else pure(false)
  } yield eq

那 DI 在這個 DB Monad 中要怎麼使用呢？


.. code-block:: scala

  abstract class ConnProvider {
    def apply[A](f: DB[A]): A
  }

  def mkProvider(driver: String, url: String) =
    new ConnProvider {
      def apply[A](f: DB[A]): A = {
        Class.forName(driver)
        val conn = DriverManager.getConnection(url)

        try { f(conn) }
        finally { conn.close }
      }
    }
  }

  lazy val sqliteTestDB =
    mkProvider("org.sqlite.JDBC", "jdbc:sqlite::memory:")
  lazy val mysqlProdDB =
    mkProvider("org.gjt.mm.mysql.Driver",
      "jdbc:mysql://prod:3306/?user=one&password=two")


Runar 是把 DB Connection 的建立用 ConnProvider 抽像化，透過這樣，
我鍆可以簡單的定意兩種不同的執行環境，一組是 MySQL 另一組是測試用的 SQLite，


.. code-block:: scala

  def runInTest[A](f: ConnProvider => A): A =
    f(sqliteTestDB)

  def runInProduction[A](f: ConnProvider => A): A =
    f(mysqlProdDB)

  def myProgram(userid: String): ConnProvider => Unit =
    r: ConnProvider => {
      println("Enter old password")
      val oldPwd = readLine
      println("Enter new password")
      val newPwd = readLine
      r(changePwd(userid, oldPwd, newPwd))
  }

  def main(args: Array[String]) =
    runInTest(myProgram(args(0)))

以上就是如何用 DB Monad 來達到 DB DI 的功用，這樣做的好處是

- Dead-simple. Just function composition.
- Explicit, type-safe dependencies.
- Lift any function.
- No frameworks, annotations, or XML.
- No initialization step.
- Doesn’t rely on esoteric language features.



More Useful Monad
-----------------

上面的 DB Monad ，只能對 DB 來做 DB ，然而，若是我們再多抽像化一層，把 DB Connection 變成
一個 **type parameter** ，那麼，我們就有一個 Reader Monad ，可以套用在許多不同應用上，
至於實際的用法， `Runar 的演講`_ 有給一個範例，請大家移步去看。

.. code-block:: scala

  case class Reader[C, A](g: C => A) {

    import Reader._

    def apply(c: C) = g(c)

    def map[B](f: A => B): Reader[C, B] = {
      c: C => f(g(c))
    }

    def flatMap[B](f: A => Reader[C, B]): Reader[C, B]  = {
        c: C => f(g(c))(c)
    }
  }

  object Reader {
    implicit def reader[A, B](f: A => B): Reader[A, B] = Reader(f)
  }

.. _for comprehension: http://stackoverflow.com/questions/1052476/can-someone-explain-scalas-yield
.. _Runar 的演講: http://www.youtube.com/watch?v=ZasXwtTRkio



總結
======

There is no silver bullet, choose your solution wisely.