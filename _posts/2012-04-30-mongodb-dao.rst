---
layout: post
title: "Mongodb Scala DAO"
date: 2012-04-30 11:50
comments: true
categories: [scala, mongodb]
---

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

