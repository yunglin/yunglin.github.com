---
layout: post
title: "parallel external merge sort"
date: 2013-03-19 21:43
comments: true
categories: 
---

Last week, I was working on importing a large input file into the system. Part of the process involved with
reading a large file (~10GB) from remote server and them sort the inputs locally.

At that moment, I decided to give Scala's new `Future API`_ a try. Within one day, I wrote a parallel external
merge sort that can

    1. Read the file from remote server and split it into smaller chunks.
    2. Use in-memory quicksort to sort each chunk and then save chunks into files.
    3. Perform merge sort on the files generated in the previous step.

The most amazing thing is that multiple threads could work on these tasks simultanously. While one thread is busy
reading bytes from remote server, other threads would perform quicksort on the part that has already been read. Once,
all of the data has been written to file system. Another thread will start to merge sort these files.

Here is how I do it.

Read Lines From InputStream
----------------------------

.. code-block:: scala

    val soure: Stream[Int] = Source.fromInputStream(inputStream).getLines().toStream.map(_.toInt)

The above code will turn a **InputStream** into a **Stream[Int]** , and then allow us to perform operation on it
without having to fully read it into memory first. But still, this is a huge Stream. Because we are going to use
in-memory sort, I need to split it into smaller pieces first.


The following code will **lift** a Stream into a Stream of Stream. Again, this operation does not require read the whole
original stream into memory. All the operation only happens logically.


.. code-block:: scala

  /**
   * Lift a Stream into a Stream of Stream. The size of each sub-stream is specified by the chunkSize
   * @param stream        the origin stream.
   * @param chunkSize     the size of each substream
   * @tparam A
   * @return              chunked stream of the original stream.
   */
  private def lift[A](stream: Stream[A], chunkSize: Int): Stream[Stream[A]] = {

    def tailFn(remaining: Stream[A]): Stream[Stream[A]] = {
      if (remaining.isEmpty) {
        Stream.empty
      } else {
        val (head, tail) = remaining.splitAt(chunkSize)
        Stream.cons(head, tailFn(tail))
      }
    }
    val (head, tail) = stream.splitAt(chunkSize)
    return Stream.cons(head, tailFn(tail))
  }


Perform Quick Sort
--------------------------------

After we have split **InputStream** into **Streams**, we can start to read each sub-stream into memory and perform
quicksort on them.


.. code-block:: scala

    val linesStream: Stream[Stream[Int]] = lift(soure, chunkSize)
    val chunkCounter = new AtomicInteger(0)

    val sortedFileDir = Files.createTempDir()
    sortedFileDir.deleteOnExit()

    // read source stream, read n entries into memory and save it to file in parallel.
    val fileFutures: List[Future[File]] = linesStream.map(
      s => {
        val chunk = chunkCounter.getAndIncrement
        Future {
          val sorted = s.sorted
          val ret = new File(sortedFileDir, "%d".format(chunk * chunkSize))
          val out = new PrintWriter(ret)

          try {
            sorted.foreach(out.println(_))
          } finally {
            out.close()
          }
          ret
        }
      }).toList


Perform Merge Sort
---------------------------------------

Because I want to perform mergesort on all of results, I have to turn **List[Future[File]]** into **Future[List[File]]**
first. So that I can instruct the **Future** to do merge sort once it has all the pieces.


.. code-block:: scala

  val saveTmpFiles: Future[List[File]] = Future.sequence(fileFutures)

  val ret: Future[File] = saveTmpFiles.map {
      files => {
    var merged = files

    while (merged.length > 1) {
      val splited = merged.splitAt(merged.length / 2)
      val tuple = splited._1.zip(splited._2)

      val m2 = tuple.map {
        case (f1, f2) => {
          val ret = new File(sortedFileDir, f1.getName + "-" + f2.getName)

          val source1 = Source.fromFile(f1)
          val source2 = Source.fromFile(f2)
          val out = new PrintWriter(ret)

          try {
            val stream1 = source1.getLines().toStream.map(_.toInt)
            val stream2 = source2.getLines().toStream.map(_.toInt)
            merge(stream1, stream2).foreach(out.println(_))
            ret
          } finally {
            out.close()
            source1.close()
            source2.close()

            FileUtils.deleteQuietly(f1)
            FileUtils.deleteQuietly(f2)
          }

        }
      }
      merged = if (merged.length % 2 > 0) {
        m2 :+ merged.last
      } else {
        m2
      }
    }
    merged.head
  }


  /**
   * Merge two streams into one stream.
   * @param streamA
   * @param streamB
   * @return
   */
  private def merge[A](streamA: Stream[A], streamB: Stream[A])(implicit ord: Ordering[A]) : Stream[A] = {

    (streamA, streamB) match {
      case (Stream.Empty, Stream.Empty) => Stream.Empty
      case (a, Stream.Empty) => a
      case (Stream.Empty, b) => b
      case _ => {
        val a = streamA.head
        val b = streamB.head

        if (ord.compare(a, b) > 0) {
          Stream.cons(a, merge(streamA.tail, streamB))
        } else {
          Stream.cons(b, merge(streamA, streamB.tail))
        }
      }
    }
  }


Give this Method a Pretty Face.
-------------------------------------------------------

So how does the method signature of this parallel external merge sort look like?

In fact, it is quite simple. It takes an **InputStream** and returns a **Future[File]**. So that, everything
happens asynchronously, nothing blocks the main thread. You can send an inputStream to this method, go to do other
things first and then come back to wait for the result.

.. code-block:: scala

  def sort(inputStream: InputStream, chunkSize: Int = 2000000): Future[File] = ???


Limits Number of Threads Running at the Same Time.
--------------------------------------------------------

Because this parallel external mergesort is an IO and memory intense operations, we can not run too many of it
simultaneously. We must put a constraint on the number of threads it can use at a time. Otherwise, we may receive
OutOfMemoryError or having many threads writing to disk simultaneously.

Also, this constraint must be a global constraint. No matter how many requests has been sent to this method at the
same time, it should only use up-to *N* threads.

Luckly, this is quite easy to do with Scala's Future API. All we need to do is to provide a fixed size thread pool
for this method. So that it won't spawn new thread by itself, instead, it uses threads provided by this global thread
pool.

.. code-block:: scala

  /**
   * limits number of reading and sorting can be executed simultaneously. Because this is an IO
   * bound operation, unless the inputstream is coming from a slow http connection, otherwise, 5
   * is more than enough.
   */
  private val GLOBAL_THREAD_LIMIT = {
    val ret = Runtime.getRuntime.availableProcessors() / 2
    if (ret > 5) {
      5
    } else {
      ret
    }
  }

  private lazy implicit val executionContext =
    ExecutionContext.fromExecutorService(Executors.newFixedThreadPool(GLOBAL_THREAD_LIMIT))


Put Everything Alltogether
-------------------------------------------------------------

.. code-block:: scala

  import com.google.common.io.Files
  import java.io.{PrintWriter, File, InputStream}
  import java.util.concurrent.Executors
  import java.util.concurrent.atomic.AtomicInteger

  import org.apache.commons.io.FileUtils

  import scala.concurrent.{ExecutionContext, Future}
  import scala.io.Source

  object InputStreams {

  /**
   * limits number of reading and sorting can be executed simultaneously. Because
   * this is an IO bound operation, unless the inputstream is coming from a slow
   * http connection, otherwise, 5 is more than enough.
   */
  private val GLOBAL_THREAD_LIMIT = {
    val ret = Runtime.getRuntime.availableProcessors() / 2
    if (ret > 5) {
      5
    } else {
      ret
    }
  }

  private lazy implicit val executionContext =
    ExecutionContext.fromExecutorService(Executors.newFixedThreadPool(GLOBAL_THREAD_LIMIT))

  def sort(inputStream: InputStream, chunkSize: Int = 2000000): Future[File] = {

    // open source stream
    val soure = Source.fromInputStream(inputStream).getLines().toStream.map(_.toInt)
    val linesStream = lift(soure, chunkSize)
    val chunkCounter = new AtomicInteger(0)

    val sortedFileDir = Files.createTempDir()
    sortedFileDir.deleteOnExit()

    // read source stream, read n entries into memory and save it to file in parallel.
    val saveTmpFiles: Future[List[File]] = Future.sequence(
      linesStream.map(s => {
        val chunk = chunkCounter.getAndIncrement
        Future {
          val sorted = s.sorted
          val ret = new File(sortedFileDir, "%d".format(chunk * chunkSize))
          val out = new PrintWriter(ret)

          try {
            sorted.foreach(out.println(_))
          } finally {
            out.close()
          }
          ret
        }
      }).toList
    )

    // perform merge sort.
    saveTmpFiles.map {
      files => {
        var merged = files
        while (merged.length > 1) {
          val splited = merged.splitAt(merged.length / 2)
          val tuple = splited._1.zip(splited._2)

          val m2 = tuple.map {
            case (f1, f2) => {
              val ret = new File(sortedFileDir, f1.getName + "-" + f2.getName)

              val source1 = Source.fromFile(f1)
              val source2 = Source.fromFile(f2)
              val out = new PrintWriter(ret)

              try {
                val stream1 = source1.getLines().toStream.map(_.toInt)
                val stream2 = source2.getLines().toStream.map(_.toInt)
                merge(stream1, stream2).foreach(out.println(_))
                ret
              } finally {
                out.close()
                source1.close()
                source2.close()

                FileUtils.deleteQuietly(f1)
                FileUtils.deleteQuietly(f2)
              }

            }
          }
          merged = if (merged.length % 2 > 0) {
            m2 :+ merged.last
          } else {
            m2
          }
        }
        merged.head
      }
    }
  }

  /**
   * Lift a Stream into a Stream of Stream. The size of each sub-stream is specified
   * by the chunkSize.
   *
   * @param stream        the origin stream.
   * @param chunkSize     the size of each substream
   * @tparam A
   * @return              chunked stream of the original stream.
   */
  private def lift[A](stream: Stream[A], chunkSize: Int): Stream[Stream[A]] = {

    def tailFn(remaining: Stream[A]): Stream[Stream[A]] = {
      if (remaining.isEmpty) {
        Stream.empty
      } else {
        val (head, tail) = remaining.splitAt(chunkSize)
        Stream.cons(head, tailFn(tail))
      }
    }
    val (head, tail) = stream.splitAt(chunkSize)
    return Stream.cons(head, tailFn(tail))
  }


  /**
   * Merge two streams into one stream.
   * @param streamA
   * @param streamB
   * @return
   */
  private def merge[A](streamA: Stream[A], streamB: Stream[A])(implicit ord: Ordering[A]) : Stream[A] = {

    (streamA, streamB) match {
      case (Stream.Empty, Stream.Empty) => Stream.Empty
      case (a, Stream.Empty) => a
      case (Stream.Empty, b) => b
      case _ => {
        val a = streamA.head
        val b = streamB.head

        if (ord.compare(a, b) > 0) {
          Stream.cons(a, merge(streamA.tail, streamB))
        } else {
          Stream.cons(b, merge(streamA, streamB.tail))
        }
      }
    }
  }


.. _Future API: http://docs.scala-lang.org/sips/pending/futures-promises.html
