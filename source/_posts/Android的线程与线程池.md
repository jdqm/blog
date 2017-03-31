---
title: Android的线程与线程池
date: 2017-03-31 11:33:17
tags:
	- Android
	- Thread
	- ThreadPool
categories: Android
---

## I. 线程
线程在Android系统中扮演者一个很重要的角色，从用途上来说，可以分为主线程和子线程，主线程一般用来处理界面与用户的交互，而子线程则往往用来执行一些耗时操作，例如I/O操作和网络访问，在Android3.0之后网络访问必须放到子线程中执行，否则会抛异常（NetworkOnMainThreadException），这样做的目的也是为了防止用户在主线程中做耗时操作，这样很容易引起ANR。在Android系统中还有一些扮演线程的角色：AsyncTask、IntentService和HandlerThread，虽然它们都有别于传统的线程，但是它们的本质仍然是传统的线程。

<!-- more -->
## II. 线程池
线程是操作系统中最小的调度单位，操作系统创建、销毁线程的开销代价是比较大的，试想一下如果不断地创建销毁线程，那么系统就会频繁地创建销毁线程，同时创建太多的线程在CUP调度时切换调度单位也是一个问题，除非线程数量少于CUP的核数才能保证绝对的并行。频繁地创建销毁线程带来不小的系统开销，这个时候线程池就是明智的选择，一个线程池会缓存一定数量的线程，通过线程池来避免频繁的创建销毁线程所带来的系统开销。Android中的线程池来源于Java，主要通过executor来派生特定类型的线程池，不同种类的线程池有不同的特性。

## III. AsyncTask

AsyncTask是一个抽象泛型类，```public abstract class AsyncTask<Params, Progress, Result>```，这三个参数中的Params是执行异步任务传入的参数类型，Progress是异步任务需要向外发出的进度值得类型，即publishProgress方法的参数类型，Result是doInbackground方法的返回值类型。下面是一个例子：
```
new MyAsyncTask("task1").execute("param1");
new MyAsyncTask("task1").execute("param1", "param2");
new MyAsyncTask("task1").execute("param1", "param2", "param3");
```
```
class MyAsyncTask extends AsyncTask<String, Integer, String>{

    private String name;

    public MyAsyncTask(String tag){
        name = tag;
    }

    /**
     * 在主线程工作，一般在这个方法中提示正在加载某些资源
     */
    @Override
    protected void onPreExecute() {
        super.onPreExecute();
    }
    /**
     * 在子线程工作(使用线程池)，在后台异步加载资源，可以通过publishProgress(Progress...         values)调用onProgressUpdate来更新进度
     *
     * @param params 这个参数是new MyAsyncTask().execute(params) 时传递的参数，其中 ... 表示参数是动态的,
     *               接收的时候是一个数组
     *
     * @return 返回值会传递到onPostExecute方法的参数
     */
    @Override
    protected String doInBackground(String... params) {
        //模拟耗时任务
        SystemClock.sleep(2000);
        Log.d(TAG, "doInBackground: " + name + ":params.length=" + params.length);
        int progress = 10;
        publishProgress(progress);
        return name;
    }

    /**
     * 在主线程工作，一般在这个方法根据参数去更新UI,例如消除掉正在加载的对话框，将结果刷新到界面
     *
     * @param s doInBackground的返回值
     *
     */
    @Override
    protected void onPostExecute(String s) {
        super.onPostExecute(s);
    }

    /**
     * 在主线程工作，用于更新进度，比如在下载的场景提示当前的下载进度
     *
     * @param values 在doInBackground中调用publishProgress(Progress... values)的参数
     */
    @Override
    protected void onProgressUpdate(Integer... values) {
        //Log.d(TAG, "onProgressUpdate: progress=" + values[0]);
    }
}
```

![此处输入图片的描述][1]

从打印结果可以看出任务时串行执行的，打印结果相差了2秒，我使用的模拟器版本是Android-22（5.1.1）。Android1.6之前的版本是串行的，1.6到Android3.0之间是并行的，3.0开始默认又变为串行的了，当然在3.0之后我们可以通过AsyncTask的 executeOnExecutor来实现并行。
```
new MyAsyncTask("task1 on executor").executeOnExecutor(AsyncTask.THREAD_POOL_EXECUTOR, "");
new MyAsyncTask("task2 on executor").executeOnExecutor(AsyncTask.THREAD_POOL_EXECUTOR, "");
new MyAsyncTask("task3 on executor").executeOnExecutor(AsyncTask.THREAD_POOL_EXECUTOR, "");
```
![此处输入图片的描述][2]

我们在代码中模拟通过SystemClock.sleep(2000)来模拟还是任务， 但是通过结果可以看到是在同一秒钟（46）打印的结果，这就意味着任务是并行处理的。

## IV. HandlerThread

HandlerThread继承自Thread，其实也没做什么，最主要的工作是帮我们做了 Looper.prepare() 和Looper.loop()，然后HanlderThread可以直接使用Handler。下面是HandlerThread的run方法的实现：
```
@Override
public void run() {
    mTid = Process.myTid();
    Looper.prepare();
    synchronized (this) {
        mLooper = Looper.myLooper();
        notifyAll();
    }
    Process.setThreadPriority(mPriority);
    onLooperPrepared();
    Looper.loop();
    mTid = -1;
}
```
你可能会发现其中有一句代码```notifyAll();```，这是干什么的，先看看getLoopere方法
```
/**
 * This method returns the Looper associated with this thread. If this thread not been started
 * or for any reason is isAlive() returns false, this method will return null. If this thread 
 * has been started, this method will block until the looper has been initialized. 
  * @return The looper.
 */
public Looper getLooper() {
    if (!isAlive()) {
        return null;
    }
   
    // If the thread has been started, wait until the looper has been created.
    synchronized (this) {
        while (isAlive() && mLooper == null) {
            try {
                wait();
            } catch (InterruptedException e) {
            }
        }
    }
    return mLooper;
}
```
你会发现里面有一个wait()，其实就是通知等待的代码，你可以执行了,这是一个同步机制。

HandlerThread用法的一个例子：
```
public class MainActivity extends AppCompatActivity {

    private static final String TAG = "MainActivity";

    private Handler mHandler;
    private HandlerThread mThread;
   
     @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        mThread = new HandlerThread("#HandlerThread");
        mThread.start();
        mHandler = new Handler(mThread.getLooper()){
            @Override
            public void handleMessage(Message msg) {
                super.handleMessage(msg);
                Log.d(TAG, "handleMessage: " + msg.what);
                mHandler.sendEmptyMessageDelayed(0, 1000);
            }
        };
    }


    @Override
    protected void onResume() {
        super.onResume();
        mHandler.sendEmptyMessageDelayed(0, 1000);
    }

    @Override
    protected void onPause() {
        super.onPause();
        mHandler.removeMessages(0);
    }

    @Override
    protected void onDestroy() {
        mThread.quit();
        super.onDestroy();
    }
}
```
由于HandlerTHread的run方法是无限循环，所以有必要去调用它的quit方法或者quitSafely方法去退出，这是一个良好的编程习惯。


## V. IntentService

IntentService是一种比较特殊的Service，它继承自Service，是一个抽象类，因此只有它的实现类才能使用。它内部封装了HandlerThread和Handler，从它的构造方法就可以看出来：
```
@Override
public void onCreate() {
    // TODO: It would be nice to have an option to hold a partial wakelock
    // during processing, and to have a static startService(Context, Intent)
    // method that would launch the service & hand off a wakelock.

    super.onCreate();
    HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
    thread.start();

    mServiceLooper = thread.getLooper();
    mServiceHandler = new ServiceHandler(mServiceLooper);
}
```
从这个构造方法中我们可以看到首先创建了一个HandlerThread，然后用它的Looper来创建了一个Handler，这就意味着IntentService的操作逻辑是在HandlerThread中执行了，所以IntetnService是可以用来做耗时操作的，另外由于它是Service的原因，所以它的优先级是要比普通的Thread高的。IntentService首次启动会调用这个构造方法，每次启动都会回调onStartCommad方法，看看onStartCommad方法：
```
@Override
public void onStart(@Nullable Intent intent, int startId) {
    Message msg = mServiceHandler.obtainMessage();
    msg.arg1 = startId;
    msg.obj = intent;
    mServiceHandler.sendMessage(msg);
}

/**
 * You should not override this method for your IntentService. Instead,
 * override {@link #onHandleIntent}, which the system calls when the IntentService
 * receives a start request.
 * @see android.app.Service#onStartCommand
 */
@Override
public int onStartCommand(@Nullable Intent intent, int flags, int startId) {
    onStart(intent, startId);
    return mRedelivery ? START_REDELIVER_INTENT : START_NOT_STICKY;
}
```
onStartCommad方法会调onStart方法，在onStart方法中，将消息发送到了mServiceHandler中
```
private final class ServiceHandler extends Handler {
    public ServiceHandler(Looper looper) {
        super(looper);
    }

    @Override
    public void handleMessage(Message msg) {
        onHandleIntent((Intent)msg.obj);
        stopSelf(msg.arg1);
    }
}
```
当消息到达handleMessage会继续掉onHandleIntent方法，当这个onHandleIntent方法执行结束，会调```stopSelf(int startId)```方法来停止服务,如果目前后多个后台任务，那么当onHandleIntent执行完最后一个任务时，```stopSelf(int startId)```才会直接停止服务。
```
@WorkerThread
protected abstract void onHandleIntent(@Nullable Intent intent);
```
这个方法是一个抽象方法，我们在自己的实现类中实现这个方法，在这个方法中做我们的操作逻辑。

例子：
```
public class LocalIntentService extends IntentService {

    private static final String TAG = "LocalIntentService";

    public LocalIntentService() {
        super(TAG);
    }

    @Override
    protected void onHandleIntent(Intent intent) {
        String action = intent.getStringExtra("action");
        SystemClock.sleep(1000);
        Log.d(TAG, "onHandleIntent: " + action);
    }


    @Override
    public void onDestroy() {
        super.onDestroy();

        Log.d(TAG, "LocalIntentService was Destroy");
    }
}
```
别忘了这是个Service，必须要在清单文件```<application>.....</application>```里面注册的。
```<service android:name=".LocalIntentService"/>```

执行如下代码
```
Intent intent = new Intent(this, LocalIntentService.class);
intent.putExtra("action","task1");
startService(intent);

intent.putExtra("action","task2");
startService(intent);

intent.putExtra("action","task2");
startService(intent);
```

![][3]

可以看到是三个任务结束后才停止服务的。

## VI. Android中的线程池

1）使用线程池的好处
①重用线程池中的线程，避免因为线程的创建和销毁带来的系统性能开销；
②能有效控制线程池的最大并发数，避免大量线程之间因互相抢占系统资源而导致阻塞的现象；
③能够对线程进行简单的管理，并提供定时执行和指定间隔时间循环执行等功能。

2）ThreadPoolExecutor
android中的线程池概念来源于java中的Executor，而Executor是一个接口，真正线程池的实现 是ThreadPoolExecutor，Android下的线程池主要分为4类，都是直接或间接配置ThreadPoolExecutor来实现的，看看ThreadPoolExecutor构造方法的参数：
```
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory) {}
                          
```   
①corePoolSize：线程池的核心线程数，默认情况下，核心线程在线程池中是一直存活的，即使它们处于空闲的状态。如果将ThreadPoolExecutor的allowCoreThreadTimeOut设置为ture，那空闲的核心线程在等待任务的过程中就会有超时策略，时间有keepAliveTime参数所指定，当等待时间超过keepAliveTime所指定的时间后，核心线程就会被终止；

②maximumPoolSize：线程池所能容纳的最大线程数，包括核心和非核心线程，当活动线程到达最大值，后续的任务就会被阻塞；

③keepAliveTiem：非核心线程闲置时的超时时长，当非核心线程空闲时间超过这个值，就会被终止。如果ThreadPoolExecutor的allowCoreThreadTimeOut设置为true，这个时间同样会作用在核心线程上；
④unit：keepAliveTime时间单位，这是一个枚举，如：TimeUnit.DAYS等;

![][4]

⑤workQueue：线程池中的任务队列，通过execute方法提交的Runable对象会存储在这个参数中;

⑥threadFactory：线程工厂，提供线程的创建功能;

⑦另外还有一个不常用的参数 RejectedExecutionHandler handler，在线程数量和任务队列到达上限时，会通过这个handler通知外界异常。

## VII. 线程池的的执行规则

1）如果线程池中的线程数量未到达核心线程的数量，那么会直接启动一个核心线程来处理任务；
2）如果线程池中的线程数已经达到或者超过核心线程数，那么任务会被插入到线程池的任务队列中排队等候；
3）如果步骤2中无法插入到任务队列中（原因可能是任务队列已经满了），这个时候如果线程池中的线程数未达到最大容量，则会直接启动一个非核心线程来处理任务；
4）如果步骤3中线程数量已经达到最大值，那么就拒绝执行任务， 这个时候就会通过 RejectedExecutionHandler handler这个参数来通知调用者。

## VIII. 线程池的分类

### **FixedThreadPool**
通过Executors的newFixedThreadPool方法来创建，这种线程池只有核心线程，没有超时策略，当所有线程处于活动状态时，任务会处于等待状态。由于这种线程池只有核心线程，并且即使空闲也不会被回收（除非线程池被回收），这就意味着它能够快速响应外界的请求。
```
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}
```
通过newFixedThreadPool方法的实现可以看到，线程池的最大容量=核心线程数(nThreads)，没有超时策略。


### **CachedThreadPool**
通过Executors的newCachedThreadPool方法来创建
```
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}
```
通过这个方法的实现可以发现，这中线程池没有核心线程，并且最大线程数量是Integer.MAX_VALUE，这就意味着没有线程数量的限制，空闲超时时间为60秒。new SynchronousQueue<Runnable>这个任务队列比较特殊，很多时候可以简单理解为一个无法存储的任务队列。也就是说，当有新的任务到来，如果有空闲的线程，则将任务给空闲的线程处理，否则直接创建一个新的线程来处理任务。这将导致任何任务将被立即执行。从它的特性来看，比较适合用来处理大量的耗时较短的任务。
### **ScheduledThreadPool**
通过Executors的newScheduledThreadPool方法来创建
```
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
    return new ScheduledThreadPoolExecutor(corePoolSize);
}

public ScheduledThreadPoolExecutor(int corePoolSize) {
    super(corePoolSize, Integer.MAX_VALUE,
          DEFAULT_KEEPALIVE_MILLIS, MILLISECONDS,
          new DelayedWorkQueue());
}

private static final long DEFAULT_KEEPALIVE_MILLIS = 10L;
```
通过这个方法的实现，可以看到，指定了核心线程数量，非核心线程数没有限制，非核心线程超时时间为10毫秒（相当于空闲后立即回收）。这类线程池适用于执行定时的任务和固定周期的重复任务。

### **SingleThreadPool**
通过Executors的newSingleThreadExecutor方法来创建
```
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}
```
这种线程池只有一个核心线程，它确保所有任务在一个线程中顺序执行。SingleThreadExecutor的意义在于，统一所有的任务在一个线程中处理，这使得这些任务之间不需要处理线程同步问题。

## IX. 4种线程池的演示例子
```
Runnable command = new Runnable() {
    @Override
    public void run() {
        SystemClock.sleep(1000);
        Log.d(TAG, "command is running");
    }
};

ExecutorService fixedThreadPool = Executors.newFixedThreadPool(2);
fixedThreadPool.execute(command);


ExecutorService cachedThreadPool = Executors.newCachedThreadPool();
cachedThreadPool.execute(command);

ScheduledExecutorService scheduledThreadPool =  Executors.newScheduledThreadPool(4);
//延时两秒后执行
scheduledThreadPool.schedule(command, 2000, java.util.concurrent.TimeUnit.MILLISECONDS);

ExecutorService singleThreadPool = Executors.newSingleThreadExecutor();

singleThreadPool.execute(command);

```
![][5]

## X. 我们在前面说到，AsyncTask使用到了线程池，我们去看看它的线程池是怎么样的

```
private static final int CORE_POOL_SIZE = Math.max(2, Math.min(CPU_COUNT - 1, 4));
private static final int MAXIMUM_POOL_SIZE = CPU_COUNT * 2 + 1;
private static final int KEEP_ALIVE_SECONDS = 30;

private static final ThreadFactory sThreadFactory = new ThreadFactory() {
    private final AtomicInteger mCount = new AtomicInteger(1);

    public Thread newThread(Runnable r) {
        return new Thread(r, "AsyncTask #" + mCount.getAndIncrement());
    }
};

private static final BlockingQueue<Runnable> sPoolWorkQueue =
        new LinkedBlockingQueue<Runnable>(128);

/**
 * An {@link Executor} that can be used to execute tasks in parallel.
 */
public static final Executor THREAD_POOL_EXECUTOR;

static {
    ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
            CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE_SECONDS, TimeUnit.SECONDS,
            sPoolWorkQueue, sThreadFactory);
    threadPoolExecutor.allowCoreThreadTimeOut(true);
    THREAD_POOL_EXECUTOR = threadPoolExecutor;
}
```
①核心线程数：  Math.max(2, Math.min(CPU_COUNT - 1, 4))，其中CPU_COUNT 是CUP的核数，CPU_COUNT = Runtime.getRuntime().availableProcessors();
②最大线程数：CPU_COUNT * 2 + 1
③超时时间：30S

以上源码均来自Android-24


  [1]: https://raw.githubusercontent.com/jdqm/hello-world/master/thread/Image1.png
  [2]: https://raw.githubusercontent.com/jdqm/hello-world/master/thread/Image2.png
  [3]: https://raw.githubusercontent.com/jdqm/hello-world/master/thread/Image3.png
  [4]: https://raw.githubusercontent.com/jdqm/hello-world/master/thread/Image4.png
  [5]: https://raw.githubusercontent.com/jdqm/hello-world/master/thread/Image5.png