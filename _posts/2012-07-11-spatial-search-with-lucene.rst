---
layout: post
title: "Spatial Search with Lucene"
date: 2012-07-11 16:06
comments: true
categories: 
styles: [data-table]
---

Before We Start
=====================================

As some of my blog readers may know, my previous startup project was an online location based search service. Thing
does not go well and I am going to open source some of the related work.

In this post, I will show you how to use Lucene to do spatial search as well as how it works internally.


Spatial Search
=====================================


Spatial Search and Geo Tagging 
---------------------------------------

Geo-tagging is a popular way to do spatial search. Instead of using the actual location to tag a POI, the geo-tagging
uses geotag to group items within a predefined range as a whole. When performing a search, it is simply pull out all
items with the same tag.

There are few popular ways to do geo-tagging, including
 - GeoHash
 - QuadTree

GeoHash
----------

GeoHash_ is way to encode latitude and longitude using base32 code. There is a fundamental flaw that makes it hard to
do spatial search. The problem is the geo hash generated is not continuous. The neighbor of a particular block may not
share the same prefix with its nearest eight neighbors. When using geohash to do spatial search, we need to calculate
the hash value of nearby neighbors by ourselves.

.. code-block:: none

  +-------+-------+-------+
  | ezefr | ezs48 | ezs49 |
  +-------+-------+-------+
  | ezefr | ezs42 | ezs43 |
  +-------+-------+-------+
  | ezefp | ezs40 | ezs41 |
  +-------+-------+-------+


.. _Geohash: http://en.wikipedia.org/wiki/Geohash


Quad-Tree
----------------------

Quad-Tree addresses the issue in geohash and adapt a better way to do geo-tagging. This example gives us a
great example on how Quad-Tree does geo-tagging.

.. image:: http://i.msdn.microsoft.com/dynimg/IC96238.jpg


The Quad-Tree has the following advantage when doing spatial search
 - it can be really fast if you stores data as a tri. Such as, Lucene, internally, it uses indexed tri to store
   NumericField.
 - the neighbors of a block share the longest common prefix.

For more extensive information regarding geo-tagging, you should check
`here <http://msdn.microsoft.com/en-us/library/bb259689.aspx>`__,
`here <http://www.maptiler.org/google-maps-coordinates-tile-bounds-projection/]>`__ and
`here <http://gist.github.com/1193577>`__.


Our Custom Approach
==========================================================

Lucene's `contrib-spatial`_ uses geo-tagging to implements spatial search and only support searching for
features near a point.

A pair of latitude and longitude gives us a quickstart for your location based application. However, not every
single feature in a LBS application can be described as a point. Let us take school district as an example.
The School A may be closer to your house than School B is, but your house is belong to school district for school B.
Another example is bike trails, bike trail is a line not a single point.

A full feature spatial search must support complicated geometric operations. This is why we create our own Lucene
spatial search implementation.

In the GIS area, Geometry are used to describe shape of features. The common geometries are:

.. _contrib-spatial: http://lucene.apache.org/core/old_versioned_docs/versions/2_9_1/api/contrib-spatial/index.html

Geometry primitives(2D)
---------------------------------------------

+------------+-----------------------------------------------+-------------------------+
| Type       | Text Format                                   | Example                 |
+============+===============================================+=========================+
| Point      |  POINT (30 10)                                |  |PointExample|         |
+------------+-----------------------------------------------+-------------------------+
| LineString | LINESTRING (30 10, 10 30, 40 40)              |  |LineStringExample|    |
+------------+-----------------------------------------------+-------------------------+
| Polygon    | POLYGON ((30 10, 10 20, 20 40, 40 40, 30 10)) |  |PolygonExample|       |
|            +-----------------------------------------------+-------------------------+
|            | POLYGON ((35 10, 10 20, 15 40, 45 45, 35 10), |  |PolygonExample2|      |
|            | (20 30, 35 35, 30 20, 20 30))|                |                         |
+------------+-----------------------------------------------+-------------------------+


Multipart geometries (2D)
---------------------------------------------

+-----------------+-------------------------------------------------+--------------------------+
| Type            | Text Format                                     | Example                  |
+=================+=================================================+==========================+
| MultiPoint      | MULTIPOINT ((10 40), (40 30), (20 20), (30 10)) | |MultiPointExample|      |
+-----------------+-------------------------------------------------+--------------------------+
| MultiLineString | MULTILINESTRING ((10 10, 20 20, 10 40)          | |MultiLineStringExample| |
|                 | (40 40, 30 30, 40 20, 30 10))                   |                          |
+-----------------+-------------------------------------------------+--------------------------+
| MultiPolygon    | MULTIPOLYGON (((30 20, 10 40, 45 40, 30 20)),   | |MultiPolygonExample|    |
|                 | ((15 5, 40 10, 10 20, 5 10, 15 5)))             |                          |
+-----------------+-------------------------------------------------+--------------------------+

.. |PointExample| image:: http://upload.wikimedia.org/wikipedia/commons/thumb/c/c2/SFA_Point.svg/51px-SFA_Point.svg.png
.. |LineStringExample| image:: http://upload.wikimedia.org/wikipedia/commons/thumb/b/b9/SFA_LineString.svg/51px-SFA_LineString.svg.png
.. |PolygonExample| image:: http://upload.wikimedia.org/wikipedia/commons/thumb/3/3f/SFA_Polygon.svg/51px-SFA_Polygon.svg.png
.. |PolygonExample2| image:: http://upload.wikimedia.org/wikipedia/commons/thumb/5/55/SFA_Polygon_with_hole.svg/51px-SFA_Polygon_with_hole.svg.png
.. |MultiPointExample| image:: http://upload.wikimedia.org/wikipedia/commons/thumb/d/d6/SFA_MultiPoint.svg/51px-SFA_MultiPoint.svg.png
.. |MultiLineStringExample| image:: http://upload.wikimedia.org/wikipedia/commons/thumb/8/86/SFA_MultiLineString.svg/51px-SFA_MultiLineString.svg.png
.. |MultiPolygonExample| image:: http://upload.wikimedia.org/wikipedia/commons/thumb/d/dc/SFA_MultiPolygon.svg/51px-SFA_MultiPolygon.svg.png



How to Calculate QuadTree value for Geometries
---------------------------------------------------

When hashing a ``Geometry primitive``, we will calculate the minimum bounding rectangle(MBR) of this geometry. For
example, the minumum bouding box for |PolygonExample| is the grid made up of (1, 1), (1, 4), (4, 4), (4, 1).

We will calculate the quadtree hashcode for the MBR, the quadtree hashcode for the MBR is the longest common prefix of
quadtree values for the four corners.

When indexing this geometry with Lucene, we will store these fields in lucene
 - geometry: POLYGON ((3 1, 1 2, 2 4, 4 4, 3 1))
 - geometry__quadtree: QuadTree value of the polygon.

Implementation Detail
======================================================================

The number of digits we used in the quad-tree implementation is 17 digits. This allows us to lay items on an 600 x 600
meter block. This number comes from 40075000 / 2 ^16 (Earth Circumference / level of details). The 17 digits number
will be stored as a long value. The digits we used are 1(upper left), 2(upper right), 3(lower left), 4(lower right),
and 0 (whole block in the given detail).

When encoding a coordinate, we will stores the quadtree value as a NumericField in Lucene. Internally, Lucene will use
indexed trie to store NumericField. The "Indexed Trie" uses buckets to group terms. We will use the default
precisionStep(4) for now.

Index Phase
--------------------------------

During the index phase, we will store the geometry in its text form and in quadtree value. Each of them has their own
use during search phase.

.. code-block:: scala

 val point = Point(24.123, 18.921)
 val code = point.quadtree


 val raw = new Field("location", point.wkt, Field.Store.YES, Field.Index.NOT_ANALYZED)
 val field = new NumericField("location" + "__quadtree")
 field.setLongValue(code)

 doc.add(raw)
 doc.add(field)


Search Phase
--------------------------------

The search phase has two parts
 - filter: fetches all possible features that may contains the geometry X by using Lucene's range query against the
   quadtree field.
 - select: for each possible results, run geometric operation to see if result Y contains geometry X.

.. code-block:: scala

  val field = "location"
  // create a point.
  val point = context.makeShape(Point(24, 18))

  // turn point into search range.
  val circle = point.buffer(Distance("500m"))

  // create a query container for complex queries.
  val query = new BooleanQuery()

  // filtering: search for items in this range
  query.add(QuadTreeRangeQueryFactory.buildLocalQuery(field, circle.quadtree))

  // selecting and scoring: the value source will return the distance between
  // the point and each feature.
  query.add(new ValueSourceQuery(context.makeValueSource("location", point));

  val founds = indexSearcher.search(query)


Open Source Projects
=======================================

Scala Shapely
---------------------------------------

Shapely_ is Scala binding for JTS Topology Suite. The goal of Shapely is to provide an easy to use factory to
create JTS geometry class instances. It allows us to create various geometry shape with easy to use factory methods.

.. code-block:: scala

  // create a point
  Point(30, 10)

  // create a line string
  LineString((30, 10), (10, 30), (40, 40))

  // create a polygon
  Polygon((30, 10), (10, 20), (20, 40), (40, 40), (30, 10))

  // create a polygon with a hole in it.
  Polygon(
      Seq((35, 10), (10, 20), (15, 40), (45, 45), (35, 10)),
      Seq((20, 30), (35, 35), (30, 20), (20, 30)))

  // create a multi-point
  MultiPoint((10, 40), (40, 30), (20, 20), (30, 10))

  // create a multi line-string.
  MultiLineString(
      Seq((10, 10), (20, 20), (10, 40)),
      Seq((40, 40), (30, 30), (40, 20), (30, 10))
  )

  // create a multi-polygon.
  MultiPolygon(
      Seq(
          Seq((30, 20), (10, 40), (45, 40), (30, 20))
          ),
      Seq(
          Seq((15, 5), (40, 10), (10, 20), (5, 10), (15, 5))
          )
      )
  )

.. _Shapely: https://bitbucket.org/bluetang/common-shapely/


Lucene Spatial
-------------------------------------------

`Lucene Spatial`_ is the geo-spatial module for lucene built on the top of shapely. The
lucene-spatial modules uses a SpatialContext instance to encode location fields and to
create location queries.

.. code-block:: scala

  // create a context.
  val context = new SpatialContext

  // create a shape from the representation of a geometry.
  val shape = context.makeShape("Point(0 10)")

  // create lucene fieldables for a location field.
  val fields = context.makeFieldables("field", shape).toList

  // create a location query to look for features within 5KM radius.
  val query = context.makeQuery("field", shape, new Distance(5, Distance.Unit.KM))


.. _Lucene Spatial: https://bitbucket.org/bluetang/lucene-spatial/
