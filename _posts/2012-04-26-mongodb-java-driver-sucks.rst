---
layout: post
title: "Problem in Mongodb's java and scala driver."
date: 2012-04-26 18:45
comments: true
categories: [mongodb]
---

MongoDB 的 Java Driver 支援透過 Transformer_ 去掛上 encodingHook 及 decodingHook，
使用方式可以參考 `Salat URISerializer`_.

原本我以為透過這樣，我就可以透過自定 encoding and decoding Hook ，去動態
的把 MongoDB 內部的資料型態，在抓取時，自動轉成我要的形態。

例如透過底下的程式碼，就在呼叫 `getAs[URL]("url")`_ 時，才即時的把 String 轉成 URL 。

.. code-block:: scala

  BSON.addEncodingHook(class[URL], new Transformer {
    def transform(o: AnyRef): AnyRef = o match {
      case u: URL => u.toExternalForm 
      case _      => o
    }
  })
  BSON.addDecodingHook(class[URL], new Transformer {
    def transform(o: AnyRef): AnyRef = o match {
      case s:String => new URL(s)
      case _ => throw new IllegalArgumentException(o.toString)
    }
  })


結果，事情不如我預期所想的， `decodingHook` 是掛在 MongoDB 儲存的資料
型態上，而不是你所需要的資料型態上；換個方法講， `decodingHook` 是在
BSON driver 把資料讀出來時，就呼叫的。

實作上，如果你去細看 Salat URISerializer ，他的 `decodingHook` 是註冊
在 `String Type` 而非 `URI Type` ，而實作時，是把字串加上了 "`URI~`" 來
判斷，這個字串是否是該被 URI Serializer 轉型成 URI。


如果你去看 `org.bson.BSON` 的實作，你會看到底下這段恐怖的程式碼，BSON
在讀出資料時，會把對該型別的 `Transformer` 全跑一遍，而這個東西又是個
Gloabal Variable，這樣一來出了三個問題

- 所有 Custom Data Type 都需要被轉成 String 並加上 Prefix 才可以寫入，
  你不可以把一個 Custom Data Type 轉成 Integer ，如果你這麼做了，所有
  的 Integer ，包含本來就該是 Integer 的欄位，未來在讀出來時，全都會變成
  這個 Custom Data Type ！

- 你的 MongoDB Driver 仰賴所有的使用者都清楚他在做什麼，也清楚別人在
  做什麼，如果有一個 Transformer 出錯了，或者是 Prefix 重覆了，那麼
  你自定的 Custom Data Type 也會出錯。

- 你 MongoDB 中的資料，為了儲存 Data Type 於資料欄位中而被污染了！

.. code-block:: java

    public static void addDecodingHook( Class c , Transformer t ){
        _decodeHooks = true;
        List<Transformer> l = _decodingHooks.get( c );
        if ( l == null ){
            l = new Vector<Transformer>();
            _decodingHooks.put( c , l );
        }
        l.add( t );
    }

    public static Object applyDecodingHooks( Object o ){
        if ( ! _anyHooks() || o == null )
            return o;

        List<Transformer> l = _decodingHooks.get( o.getClass() );
        if ( l != null )
            for ( Transformer t : l )
                o = t.transform( o );
        return o;
    }

SCALA-66_ 說 3.0.0 release 時，會有一個 per field, per data type 的
type conversion機制，不過 3.0.0-M2 時還沒把這塊做進去，所以看來有得等了

我是自己有參照 simplistic_ 刻一個機制，可以在 DAO 中讀進讀出物件時做轉
換，但是在寫 Query 時，仍是有些不便。

如果你有這需求的話，可以跟我連絡，我把 source code 給你。

.. _Transformer: http://api.mongodb.org/java/2.5/org/bson/Transformer.html

.. _Salat URISerializer: https://github.com/novus/salat/blob/9d7988fbd212b6a1423cb2be72065ebff3570777/salat-core/src/test/scala/com/novus/salat/test/util/UriConversionHelper.scala

.. _getAs[URL]("url"): http://api.mongodb.org/scala/casbah/2.1.5.0/scaladoc/com/mongodb/casbah/commons/MongoDBObject.html#getAs[A](String)(Manifest[A]):Option[A]


.. _SCALA-66: https://jira.mongodb.org/browse/SCALA-66

.. _simplistic: https://github.com/aboisvert/simplistic
