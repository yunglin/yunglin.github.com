---
layout: post
title: "Lesson Learned form Developing Locadz SDK"
date: 2012-04-20 23:07
comments: true
categories: 
---

Introduction
------------

市面上大部份講 Android 的書，多數都是在講 API 要怎麼樣使用，但是，許多的書都沒有提到一個重要的問題，就是什
麼是 UI Thread (有時被稱做Main Thread) ，為什麼要有 UI Thread 以及為什麼不可以在 UI Thread 中執行耗時間的運算。

在本文中，我講談一談我們在開發Locadz SDK所使用到的一些技巧。

UI Thread
---------

UI Thread是Android(or other GUI library)用來處理 Event 的 thread ，這些 Event 可能是使用者處發的，如
Touch Event，或者是其它程式或者是系統底層的事件，如 Intent 或 `Location Updates`_.

在 UI Thread 處理完一個事件後， UI Thread 會重畫整個 Application ，而前面這句話解釋了處理 ANR 的兩個原則

- UI事件的處理要越快越好，在處理完前，UI不會有反應
- UI的更新，一定要發生在UI Thread，要不然，要等到下次UI Event被處發時才會一併被處理


.. _Location Updates: http://developer.android.com/reference/android/location/LocationListener.html


Lesson I: Use WeakReference to Avoid Strong Reference and Memory Leak
----------------------------------------------------------------------

有許多的文章談到如何用 AsyncTask_ 或 ResultReceiver_ 來把需要大量運算的程式，放在另一個 Thread 來做處理。

然而，這些範例都有著一個常備忽略的問題，那就是，這些 AsyncTask or ResultReceiver 常會把前景的Activity包進來
，如果說你的程式只有一個 Activity 的話，這或許不是什麼問題，但是若是你的 App 有多個畫面的話，這些在背景執行的程
式可能就會讓你的程式有 memory leak 的問題；如某家廣告商的 SDK 就有此問題。

一個可能的發生狀況是，Activity A透過 AsyncTask 去遠端下載一個圖片來顯示在 *Activity A* 之上，但因為某些因素
(如忘了設 connection timeout)，這個下載的程緒卡住了，既使 User 以從 *Activity A* 切換到 *Activity B* 之
上，這個 *Activity A* 還是不會被 GC 回收掉。


要避開因為有個 Strong Reference 造成 Activity 無法被回收的問題，我們在把有可能被回收的物件傳到另一個 Thread 中
被延後執行時，必需用 WeakReference_ 包住這個物件；如此一來，當Garbage Collector碰到一個已經不在前景的 Activity
時，Garbage Collector會把這物件處理掉，如此一來，就不會有 memory leak 的問題。

.. _AsyncTask: http://developer.android.com/reference/android/os/AsyncTask.html
.. _ResultReceiver: http://stackoverflow.com/questions/3197335/restful-api-service/3197456#3197456
.. _WeakReference: http://docs.oracle.com/javase/1.5.0/docs/api/java/lang/ref/WeakReference.html

.. code-block:: java

  /**
   *  AsyncTask to Load Image
   */
  public class DownloadImagesTask extends AsyncTask<Uri, Void, Bitmap> {

    WeakReference<imageview> imageViewWeakReference = null;

    public DownloadImagesTask(ImageView imageView) {
      imageViewWeakReference = new WeakReference<Imageview>(imageView);
    }

    @Override
    protected Bitmap doInBackground(Uri... uri) {
      return downloadImage(uri);
    }

    @Override
    protected void onPostExecute(Bitmap result) {
      ImageView imageView = imageViewWeakReference.get();
      if (imageView != null) {
        imageView.setImageBitmap(result);
      }
    }

    private Bitmap downloadImage(Uri url) {
       ...
    }
  }



Lesson II: Use IntentService to Run Business Logic
---------------------------------------------------

AsyncTask是 Android 最常被用來處理複雜運算時用的工具，透過AsyncTask，我們可以在背景處裡
一些複雜的運算，再把結果放回前景之上。

但據我的經驗，使用 AsyncTask 同時間來擔任 MVC 中的 View & Controller 的工作，最後往往是
把程式碼弄成一團麵線。因此，在開發 Locadz SDK 時，我們把一些跟 UI 無關的運算，都切出來
變成 IntentService 或者是非 Inner class 的 AsyncTask ，把所有的運算邏輯從 Activity 中切出
來，增加重用的可能。

然後運算的結果，再透過 `getHandler().post(...)`_ 更新到 UI 之上.


.. _getHandler().post(...): http://developer.android.com/reference/android/os/Handler.html#post(java.lang.Runnable)


.. code-block:: java

  /** Service that retrieve the ad unit allocations from external source and cache locally in SharedPreference. */
  public class AdUnitAllocationService extends IntentService {

    private static final int CACHE_EXPIRATION_PERIOD = 30 * 60 * 1000; // 30 minutes.

    private final static String PREFS_STRING_TIMESTAMP = "timestamp";
    private final static String PREFS_STRING_CONFIG = "config";

    // response code for possible result.
    public static final int RESULT_OK = 1;

    public AdUnitAllocationService() {
      super(AdUnitAllocationService.class.getCanonicalName());
    }

    @Override
    protected void onHandleIntent(Intent intent) {

      AdUnitContext adUnitContext =
       (AdUnitContext) intent.getParcelableExtra(IntentConstants.EXTRA_ADUNIT_CONTEXT);

      AdUnitAllocation adUnitAllocation = getAdUnitAllocation(adUnitContext);

      if (adUnitAllocation != null) {
        Ration ration = getRandomRation(adUnitAllocation.getRations());

        // send response through ResultReceiver.
        ResultReceiver receiver = intent.getParcelableExtra(IntentConstants.EXTRA_RECEIVER);

        Bundle resultData = new Bundle();
        resultData.putString(IntentConstants.EXTRA_ADUNIT_ID, adUnitContext.getAdUnitId());
        resultData.putSerializable(IntentConstants.EXTRA_RATION, ration);
        resultData.putSerializable(IntentConstants.EXTRA_EXTRA,
                                   adUnitAllocation.getExtra());

        receiver.send(RESULT_OK, resultData);
      }
    }

    /**
     * Select a random ration form the provided rations.
     * @param rations   the candidates.
     * @return a random ration from the candidates.
     */
    private Ration getRandomRation(List<Ration> rations) {
      //...
    }

    /**
     * Get the allocation configuration for the adunit.
     * @param adUnitContext the context of the adunit.
     * @return the allocation configuration for the adunit.
     */
    AdUnitAllocation getAdUnitAllocation(AdUnitContext adUnitContext) {
      //...
    }
  }


Lesson III: Use Disk Cache instead of (Main) Memory Cache
----------------------------------------------------------

底下的圖表，是Jeff Dean發表的，在談的是讀取資料的的效率，我們把這幾個數字先記起來，
再加一個代表UI設計時人體覺得是即時反應的反應時間上限 100 ms。然後我們再來談 Android UI 的設計。

.. image:: /images/2012-04-20/numbers-everyone-should-know.png

大家可以看到 Main Memory Reference(0.001ms) 與 Disk Seek(10ms) Disk Read(30ms) 的重大差
距，然而，後者的數字在 Mobile Phone 上就不是這樣了。在 Mobile Phone 上，傳統的硬碟扮演的角色，
被NAND Flash Memory, External SD Card所取代了。在存取效率上 NAND Flash Memory 雖然不
比 RAM 快，但是，也仍是 seek time ~1ms 的狠角色。

這 1ms 的負擔，雖比 0.001ms 的負擔高上百倍，以上，但是在 100ms 這UI 反應需求上，卻又變得很渺小了。

因此，在這邊，我會建議大家，若是有 cache 的需求時，直接往 `Internal Storage`_ 塞吧，不要放
在Main memory上，或用個SoftReferenceMap包著。


.. _Internal Storage: http://developer.android.com/guide/topics/data/data-storage.html#filesInternal


Lesson IV: Make All Public Method Async to Avoid UI Update Issue
-----------------------------------------------------------------

在上面第一個範例中有個錯誤，那就是DownloadImagesTask.onPostExecute()會在呼叫DownloadImagesTask.execute(...)的
那個 Thread 上執行，如果說，很不幸的，這個 DownloadImagesTask 並不是從 UI Thread 上來呼叫的話，那
麼，imageView.setImageBitmap(result)便有可能不會即時更新到UI之上。

如果你的開發環境會有這種問題，在包在層層呼叫後，無法確保 Method 是否是在 UI Thread 上執行；那麼我
會建議你把會更新 UI 的 Method ，標成 protected ，然後開放一個 public async method 出來，範例如下：

.. code-block:: java

  /**
    * Remove old ad views and push the new one.
    *
    * @param subView the adview to push.
    */
  protected void pushSubView(ViewGroup subView) {
    //....
  }

  /**
   *  submit a push view request to Android's handler. This will remove
   *  old ad view and push a new one to this layout asynchronously.
   *
   * @param subView   the adview to push.
   */
  public void submitPushSubViewRequest(ViewGroup subView) {
    Log.d(LOG_TAG, String.format("Scheduled pushSubView(%s)", subView));
    getHandler().post(new ViewAdRunnable(this, subView));
  }


  /**
   * Runnable runs on the Main Thread that pushes an AdView to the layout.
   */
  private static final class ViewAdRunnable implements Runnable {

    private final WeakReference<Adunitlayout> locadzLayoutWeakReference;

    private ViewGroup subView;

    public ViewAdRunnable(AdUnitLayout layout, ViewGroup subView) {
      locadzLayoutWeakReference = new WeakReference<Adunitlayout>(layout);
      this.subView = subView;
    }

    @Override
    public void run() {
      AdUnitLayout locadzLayout = locadzLayoutWeakReference.get();
      if (locadzLayout != null) {
        locadzLayout.pushSubView(subView);
      }
    }
  }

