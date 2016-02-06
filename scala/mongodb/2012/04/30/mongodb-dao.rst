<!DOCTYPE html>
<html>

  <head>
  <meta charset="utf-8">
  <meta http-equiv="X-UA-Compatible" content="IE=edge">
  <meta name="viewport" content="width=device-width, initial-scale=1">

  <title>Mongodb Scala DAO</title>
  <meta name="description" content="這篇文章將說說，我使用 MongoDB & Cashbah 的一些心得，這程式的主架構是我從 simplistic_ 學來的。">

  <link rel="stylesheet" href="/css/main.css">
  <link rel="canonical" href="http://yourdomain.com/scala/mongodb/2012/04/30/mongodb-dao.rst">
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
    <h1 class="post-title" itemprop="name headline">Mongodb Scala DAO</h1>
    <p class="post-meta"><time datetime="2012-04-30T11:50:00-07:00" itemprop="datePublished">Apr 30, 2012</time></p>
  </header>

  <div class="post-content" itemprop="articleBody">
    這篇文章將說說，我使用 MongoDB & Cashbah 的一些心得，這程式的主架構是
我從 simplistic_ 學來的。

.. _simplistic: https://github.com/aboisvert/simplistic


The Abstract DAO
=================

底下是一個很簡單的 CRUD 的 MongoDB DAO. 在這邊，我們建立一個 DAO 時，我們會把這
個 DAO 用的儲存空間 MongoCollection 傳進來，另外要注意的一點是，這個 MongoCollection 的
WriteConcert 必需是 Strict 的。

.. code-block:: scala

  /**
   *  Base class for all DAO classes.
   */
  abstract class AbstractDAO[T, ID <: Serializable](protected[locadz] val mongoCol: MongoCollection) {

    /** builder to turn mongodb object into model object */
    implicit protected val builder: DBObject => Either[DataAccessException, T]

    /** builder to turn model object into mongodb object */
    implicit protected val extractor: T => DBObject

    /** this dao expects the underlying mongoCol would throw exception when error occurs. */
    require(mongoCol.getWriteConcern != null)
    require(mongoCol.getWriteConcern.getW >= 1, "Write must be >= 1, was %d".format(mongoCol.getWriteConcern.getW))

    /** insert object into the underly*/
    def insert(obj: T): Either[DataAccessException, ID] = {
      try {
        val dbObj: DBObject = extractor(obj)
        val res: WriteResult = mongoCol.insert(dbObj)
        Right(dbObj.getAs[ID]("_id").get)
      } catch {
        case e: MongoException      => Left(new DataAccessException(e))
        case e: DataAccessException => Left(e)
      }
    }

    def save(obj: T): Option[DataAccessException] = {
      try {
        val res = mongoCol.save[T](obj)
        None
      } catch {
        case e: MongoException      => Some(new DataAccessException(e))
        case e: DataAccessException => Some(e)
      }
    }

    def findById(id: ID): Either[DataAccessException, Option[T]] = {
      try {
        mongoCol.findOneByID(id) match {
          case None        => Right(None)
          case Some(dbObj) => builder.apply(dbObj).right.map(Option(_))
        }
      } catch {
        case e: DataAccessException => Left(e)
      }
    }

    def delete(id: ID): Option[DataAccessException] = {
      try {
        mongoCol.remove(MongoDBObject("_id" -> id))
        None
      } catch {
        case e: MongoException => Some(new DataAccessException(e))
      }
    }

    def find[T](dbObj: DBObject)(implicit builder: (DBObject) => Either[DataAccessException, T]): Either[DataAccessException, Iterable[T]] = {

      try {
        Right(mongoCol.find(dbObj)
          .flatMap(found => {
            builder.apply(found).fold(
              e => throw e,
              r => Some(r))
          }).toStream)

      } catch {
        case e: DataAccessException => Left(e)
      }
    }
  }


這邊有兩個需要被實作的東西

- `val builder: DBObject => Either[DataAccessException, T]`
- `val extractor: T => DBObject`

而怎麼建立這兩個東西，就是全文的重點了。

How to use AbstractDAO
======================

先來看一個範例；實作上，在 UserDAO object 上，我們先把欄位的名稱及資料型態，一個個的定好

- 'val Name = attribute("name", Conversion.String)`
- 'val Email = attribute("email", Conversion.String)'
- 'val CreateDate = attribute("ctime", Conversion.JodaDate)'

而這個 attribute method 做的是，把資料名稱存起來，然後當這個 Attribute instance

- 收到一個 DBObject 時，把對應的欄位取出來，並將值給轉成定義的格式
- 收到一個 property value 時，把他轉成 (name -> value) pair

第一個用法的這個方式，用在從 *builder* 上面，一個個的把值從 DBObject 抓出來再來建立物件，
第二個用法用在，把物件轉成 DBObject 時，及用在建立 query 時。


UserDAO 實作
--------------

.. code-block:: scala

  class UserDAO(mongoCol: MongoCollection) extends AbstractDAO[User, String](mongoCol) {

    import UserDAO._

    implicit protected val builder: (DBObject) => Either[DataAccessException, User] = UserDAO.dbObjectToUser _

    implicit protected val extractor: (User) => DBObject = UserDAO.userToDBObject _

    def this(dbCol: DBCollection) = this(dbCol.asScala)

  }

  object UserDAO {

    import Attributes._

    val Name = attribute("name", Conversion.String)

    val Email = attribute("email", Conversion.String)

    val CreateDate = attribute("ctime", Conversion.JodaDate)

    def dbObjectToUser(dbObj: DBObject): Either[DataAccessException, User] = {

      try {
        val id = dbObj.getAs[String]("_id").get

        Right(
          new User(id,
            Name(dbObj),
            Email(dbObj),
            CreateDate(dbObj)
          )
        )
      } catch {
        case e: DataAccessException => Left(e)
      }
    }

    implicit def userToDBObject(user: User): DBObject = {

      val ret = MongoDBObject("_id" -> user.id)
      ret += Name(user.externalId)
      ret += Email(user.verifiedId)
      ret += CreateDate(user.createDate)
      return ret
    }
  }


Attribute & Conversion 部份實作
-------------------------------------

.. code-block:: scala

  trait Attribute[A, B <: Any] {
    val name: String
    val conversion: Conversion[A, B]

    protected implicit val manifestA: Manifest[A]
    protected implicit val manifestB: Manifest[B]
  }

  trait RequiredAttribute[A, B <: Any] extends Attribute[A, B] {

    def apply(value: A): Tuple2[String, B] = {
      name -> conversion.apply(value)
    }

    def apply(value: DBObject): A = {
      value.getAs[B](name).map(b => conversion.unapply(b)).getOrElse(throw new MissingRequiredAttributeException(name))
    }
  }

  def attribute[A, B <: Any](name: String, dataType: Conversion[A, B])(implicit manifestA: Manifest[A], manifestB: Manifest[B]) = {
    new SingleValueRequiredAttribute[A, B](name, dataType)
  }


  trait Conversion[A, B] {
    def apply(source: A): B
    def unapply(value: B): A
  }

  object Conversion {
    case object String extends Conversion[String, String] {
      override def apply(source: String) = source
      override def unapply(value: String) = value
    }

    /**
     * Conversion for DateTime to DateTime.
     */
    object JodaDate extends Conversion[DateTime, DateTime] {

      RegisterJodaTimeConversionHelpers()

      override def apply(source: DateTime): DateTime = source
      override def unapply(value: DateTime): DateTime = value
    }
  }


How to Use Attributes to Build Query
=====================================

前面的 Attribute(s) 不只是用來產做 builder 及 extractor 用的，他們最好用的地方在於
建立 query 時，如下

.. code-block:: scala

  def findByDateRange(start: DateTime, end: DateTime): Either[DataAccessException, User] = {
    return find(CreateDate.name $gth CreateDate(start)._2  $lt CreateDate(range)._2)
  }


這邊，我們可以看到用 Conversion 來描述資料型態的好處是，當你儲存的資料型態有變時，你不用去更
改你的 MongoDB query ，欄位的名稱、資料型別的轉換，都被封裝起來，使用者在寫 Query 時，不用去
記欄位的名稱，也不用去記這個欄位的型態是什麼， MongoDB 支不支援這個型態。


第二個好處是， Casbah 本來只支援 java primitive type & java.util.Date ，透過 Conversion ，
我們可以支援任意的資料型態於 query 中，只要透過新的 Conversion 類別，我們就可以增加 MongoDB 可以
支援的資料型態。


例如，如果我們想增加對 `java.net.URL` 的支援，只要訂義以下的 Conversion 即可

.. code-block:: scala

  case object Url extends Conversion[URL, String] {

    override def apply(source: URL) = source.toExternalForm

    override def unapply(value: String) = new URL(value)
  }



Appendix:
==========


Attribute.scala
----------------

.. code-block:: scala

  package com.locadz.model.mongodb

  import com.mongodb.DBObject
  import com.mongodb.casbah._
  import com.locadz.model.exception.MissingRequiredAttributeException

  /**
   * Date: 2/20/12
   */
  object Attributes {

    trait Attribute[A, B <: Any] {
      val name: String
      val conversion: Conversion[A, B]

      protected implicit val manifestA: Manifest[A]

      protected implicit val manifestB: Manifest[B]

    }

    trait OptionalAttribute[A, B <: Any] extends Attribute[A, B] {

      def apply(value: Option[A]): Tuple2[String, Any] = {
        val v = value.map(conversion.apply(_)).getOrElse(null)
        name -> v
      }

      def apply(value: DBObject): Option[A] = value.getAs[B](name).map(b => conversion.unapply(b))

    }

    trait RequiredAttribute[A, B <: Any] extends Attribute[A, B] {

      def apply(value: A): Tuple2[String, B] = {
        name -> conversion.apply(value)
      }

      def apply(value: DBObject): A = {
        value.getAs[B](name).map(b => conversion.unapply(b)).getOrElse(throw new MissingRequiredAttributeException(name))
      }
    }

    case class SingleValueRequiredAttribute[A, B <: Any](name: String, conversion: Conversion[A, B])(implicit val manifestA: Manifest[A], implicit val manifestB: Manifest[B]) extends RequiredAttribute[A, B]

    case class SingleValueOptionalAttribute[A, B <: Any](name: String, conversion: Conversion[A, B])(implicit val manifestA: Manifest[A], implicit val manifestB: Manifest[B])
      extends OptionalAttribute[A, B]

    def attribute[A, B <: Any](name: String, dataType: Conversion[A, B])(implicit manifestA: Manifest[A], manifestB: Manifest[B]) = {
      new SingleValueRequiredAttribute[A, B](name, dataType)
    }

    def optionalAttribute[A, B <: Any](name: String, dataType: Conversion[A, B])(implicit manifestA: Manifest[A], manifestB: Manifest[B]) = {
      new SingleValueOptionalAttribute[A, B](name, dataType)
    }

  }


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
