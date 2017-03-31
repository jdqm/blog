---
title: Handler消息机制
date: 2017-02-20 17:13:32
tags:
	- 消息机制
	- Handler
categories:
---

>消息机制主要包含三个元素：Handler、MessageQueue、Looper

#### 工作原理
Hander被创建后，通过Handler的post方法将一个Runable投递到Handler内部的Looper中去处理，或者通过Handler的send方法发送一个消息到Handler内部的Looper中处理，其中post方法最终也是通过send方法实现的。具体的过程是：当Handler的send方法被调用发出一个消息，MessageQueue就会将这个消息插入消息队列中，Looper发现有新的消息（MessageQueue的next方法），就会处理这个消息。最终这个消息的Runable或者Handler的handleMessage方法就会被调用。Looper是运行在创建Handler的线程中，这样一来Handler中的业务逻辑就会切换到创建Handler的线程中。
<!-- more -->

#### 1、ThreadLocal
 >ThreadLocal是线程内部的数据存储类，可以通过它在指定的线程中存储数据，数据存储后，只能指定的线程能访问，别的线程都不能访问。应用场景：
 
- 当某个数据以线程为作用域且对于不同的线程需要不同的副本时可以选用；
- 复杂逻辑下的对象传参。
```
private ThreadLocal<Boolean> mBooleanThreadLocal = new ThreadLocal<>();
```
```
mBooleanThreadLocal.set(true);
Log.d(TAG, "MainThread: " + mBooleanThreadLocal.get());
```
```
new Thread(new Runnable() {
    @Override
    public void run() {
        mBooleanThreadLocal.set(false);
        Log.d(TAG, "Thread1: " + mBooleanThreadLocal.get());
    }
}).start();
```
```
new Thread(new Runnable() {
     @Override
     public void run() {
         Log.d(TAG, "Thread2: " + mBooleanThreadLocal.get());
     }
 }).start();
```
![这里写图片描述](http://img.blog.csdn.net/20161205112842050)

上面的例子可以看出，不同的线程访问同一个成员变量mBooleanThreadLocal ，MainThread设置为 true, 调用mBooleanThreadLocal.get()得到的是true；而Thread1设置为false，访问到的是false；Thread2没有设置，调用mBooleanThreadLocal.get()得到是null。不同的线程访问同一个成员变量得到的值不一样，这是为什么？看看set和get方法的实现。

①先看看set方法的实现
```
/**
 * Sets the current thread's copy of this thread-local variable
 * to the specified value.  Most subclasses will have no need to
 * override this method, relying solely on the {@link #initialValue}
 * method to set the values of thread-locals.
 *
 * @param value the value to be stored in the current thread's copy of
 *        this thread-local.
 */
 public void set(T value) {
     Thread t = Thread.currentThread();
     ThreadLocalMap map = getMap(t);
     if (map != null)
         map.set(this, value);
     else
         createMap(t, value);
 }
```
可以看到，这个方法从当前线程获取了一个ThreadLocalMap对象，然后把值set进去，如果获取的ThreadLocalMap对象为空，则创建一个，再把值放进去。

![这里写图片描述](http://img.blog.csdn.net/20161205113615000)

接下来就该看看map.set(this, value);是如何把值放进去的
```
/**
 * Set the value associated with key.
 *
 * @param key the thread local object
 * @param value the value to be set
 */
private void set(ThreadLocal key, Object value) {

    // We don't use a fast path as with get() because it is at
    // least as common to use set() to create new entries as
    // it is to replace existing ones, in which case, a fast
    // path would fail more often than not.

    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);

    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        ThreadLocal k = e.get();

        if (k == key) {
            e.value = value;
            return;
        }

        if (k == null) {
            replaceStaleEntry(key, value, i);
            return;
        }
    }

    tab[i] = new Entry(key, value);
    int sz = ++size;
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}
```
可以看到通过 int i = key.threadLocalHashCode & (len-1); 获取一个索引，当然不同的key通过Hash散列后可能会有冲突，在这个方法中也处理了冲突的情况，也就是说存入的位置不一定是int i = key.threadLocalHashCode & (len-1);得到的i；然后将value存入ThreadLocalMap中的table数组中。



②再看一下mBooleanThreadLocal.get()这个get()方法的实现。
```
/**
 * Returns the value in the current thread's copy of this
 * thread-local variable.  If the variable has no value for the
 * current thread, it is first initialized to the value returned
 * by an invocation of the {@link #initialValue} method.
 *
 * @return the current thread's value of this thread-local
 */
 public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null)
            return (T)e.value;
    }
    return setInitialValue();
 }
```
可以看到，get方法就是从ThreadLocalMap中的table数组中取出通过set方法放进去的值，如果找不到则调用setInitialValue()初始化并返回，这个初始化默认是初始化为null。

再看看setInitialValue()其实就是为当前线程中的一个ThreadLocalMap（就是threadLocals）设置了一个值，如果是上面的例子就是：map.set( mBooleanThreadLocal, null); ，所以我们在线程2中得到null。

结论：通过mBooleanThreadLocal.get()方法取出的是各个线程中ThreadLocalMap中的table数组中的某个索引的值，当然就不一样啦。


下面的代码
```
new Thread(new Runnable() {
	@Override
	public void run() {
	    Log.d(TAG, "Thread2: " + mBooleanThreadLocal.get());
	}
}).start();
```
没有先调用set方法，直接取值，返回的是setInitialValue()这个方法的返回值，即为null，我们能不能实现一个返回一个默认的值，比如我没有调set方法，但是我希望返回的false，而不是null。我们看看setInitialValue()的实现
```
/**
 * Variant of set() to establish initialValue. Used instead
 * of set() in case user has overridden the set() method.
 *
 * @return the initial value
 */
private T setInitialValue() {
    T value = initialValue();
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
    return value;
}

protected T initialValue() {
    return null;
}
```



好了，看到这里我们就明白了为什么是null，而且这个方法是protected修饰的，并且返回的是泛型，我们可以重写它来实现我们想返回的值。示例：

①我们需要写一个类继承ThreadLocal并重写initalValue()方法
```
class MyThreadLocal extends ThreadLocal{
        @Override
        protected Object initialValue() {
            return false;
        }
    }
```
②创建一个实例，直接调get方法，看看结果是null，还是false
```
private MyThreadLocal myThreadLocal = new MyThreadLocal();
Log.d(TAG, "MainThread: " + myThreadLocal.get());
```
③跑起来看看logcat

![这里写图片描述](http://img.blog.csdn.net/20161205114417099)

#### 2、MessageQueue

>MessageQueue即消息队列，这是一个链式的队列。为什么采用链式而不采用顺序存储（比如数组）。这就取决于消息队列的特点：经常做的操作是插入，取出删除。

顺序存储 VS 链式存储

| 存储方式 | 查找效率 | 插入、删除效率 |
| ------------- |-------------|-----|
| 顺序存储 | 高，可直接根据索引查找 | 低，插入删除操作都有可能需要移动大量的数据 |
| 链式存储 | 低，需要从头开始查找 | 高 |

MessageQueue 主要涉及两个方法：①boolean enqueueMessage(Message msg, long when)，这个方法就是将消息插入消息队列中；②Message next()，这个方法作用是从消息队列中取出一条消息，并且将这条消息从消息队列中删除。
```
 Message next() {
        // Return here if the message loop has already quit and been disposed.
        // This can happen if the application tries to restart a looper after quit
        // which is not supported.
        final long ptr = mPtr;
        if (ptr == 0) {
            return null;
        }
        int pendingIdleHandlerCount = -1; // -1 only during first iteration
        int nextPollTimeoutMillis = 0;
        for (;;) {
            if (nextPollTimeoutMillis != 0) {
                Binder.flushPendingCommands();
            }
            nativePollOnce(ptr, nextPollTimeoutMillis);
            synchronized (this) {
                // Try to retrieve the next message.  Return if found.
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                Message msg = mMessages;
                if (msg != null && msg.target == null) {
                    // Stalled by a barrier.  Find the next asynchronous message in the queue.
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                if (msg != null) {
                    if (now < msg.when) {
                        // Next message is not ready.  Set a timeout to wake up when it is ready.
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        // Got a message.
                        mBlocked = false;
                        if (prevMsg != null) {
                            prevMsg.next = msg.next;
                        } else {
                            mMessages = msg.next;
                        }
                        msg.next = null;
                        if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                        msg.markInUse();
                        return msg;
                    }
                } else {
                    // No more messages.
                    nextPollTimeoutMillis = -1;
                }
                // Process the quit message now that all pending messages have been handled.
                if (mQuitting) {
                    dispose();
                    return null;
                }
                // If first time idle, then get the number of idlers to run.
                // Idle handles only run if the queue is empty or if the first message
                // in the queue (possibly a barrier) is due to be handled in the future.
                if (pendingIdleHandlerCount < 0
                        && (mMessages == null || now < mMessages.when)) {
                    pendingIdleHandlerCount = mIdleHandlers.size();
                }
                if (pendingIdleHandlerCount <= 0) {
                    // No idle handlers to run.  Loop and wait some more.
                    mBlocked = true;
                    continue;
                }
                if (mPendingIdleHandlers == null) {
                    mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
                }
                mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
            }
            // Run the idle handlers.
            // We only ever reach this code block during the first iteration.
            for (int i = 0; i < pendingIdleHandlerCount; i++) {
                final IdleHandler idler = mPendingIdleHandlers[i];
                mPendingIdleHandlers[i] = null; // release the reference to the handler
                boolean keep = false;
                try {
                    keep = idler.queueIdle();
                } catch (Throwable t) {
                    Log.wtf(TAG, "IdleHandler threw exception", t);
                }
                if (!keep) {
                    synchronized (this) {
                        mIdleHandlers.remove(idler);
                    }
                }
            }
            // Reset the idle handler count to 0 so we do not run them again.
            pendingIdleHandlerCount = 0;
            // While calling an idle handler, a new message could have been delivered
            // so go back and look again for a pending message without waiting.
            nextPollTimeoutMillis = 0;
        }
    }
```
可以看到这个方法是个无限循环，如果没有新的消息，next方法就会阻塞在这里，如果有新的消息到来时，next方法就会返回这个消息并在单链表中将其删除。要退出这个无限循环可以调Loopper的quit()或者Looper的quitSafely()，前者是直接退出，后者是处理完现有消息再退出。对应next方法中的
```
// Process the quit message now that all pending messages have been handled.
if (mQuitting) {
     dispose();
     return null;
 }
 这段代码
```

#### 3、Looper

>Looper在Android消息机制中扮演消息循环的角色，具体来说就是不停的从MessageQueue中查看是否有新的消息，如果有新的消息则立即处理，否则一则阻塞在那里。

我们知道Handler的工作需要Looper，比如：
```
 new Thread("Thread#3"){
    @Override
     public void run() {
          handler = new Handler();
     }
 }.start();
```
其中的handler是Activity的一个成员属性，子线程默认是没有Looper的，这样就会报如下异常
```
java.lang.RuntimeException: Can't create handler inside thread that has not called Looper.prepare()
```
没有Looper那就自己创建，异常信息也告诉我们调Looper.prepare() 就可以，好吧，那我们就改一下
```
new Thread("Thread#3"){
    @Override
    public void run() {
        Looper.prepare();
        handler = new Handler(){

            @Override
            public void handleMessage(Message msg) {
                super.handleMessage(msg);
                Log.d(TAG, "handleMessage: 收到主线程发过来的消息啦 " + msg.what);
            }
        };
    }
}.start();
```
这个时候我们在Activity增加一个Button去点击发送消息到这个子线程中的handler
```
public void sendMessage(View view) {
        int what = 0;
        handler.sendEmptyMessage(what);
}
```
结果是根本不会执行handleMessage(Message msg)方法，原因是我们没有调用Looper.loop()方法来开启消息循环，也就是消息发出去了，但是Looper并没有去取消息。将代码改为如下即可：
```
new Thread("Thread#3"){

    @Override
    public void run() {
        Looper.prepare();
        handler = new Handler(){

            @Override
            public void handleMessage(Message msg) {
                super.handleMessage(msg);

                Log.d(TAG, "handleMessage: 收到主线程发过来的消息啦 " + msg.what);
            }
        };
        Looper.loop();
    }
}.start();
```
![这里写图片描述](http://img.blog.csdn.net/20161205122606399)

为什么这个要增加一个点击事件来发消息而不是直接发送，主要是这个handler在子线程中创建，不知道什么时候它会执行完，所以我是等一会再点发送。如果调用handler.getLooper().quit()或者handler.getLooper().quitSafely()，这个线程就会终止，所以不能再发消息到这个handler否则会报如下异常：

![这里写图片描述](http://img.blog.csdn.net/20161205120441330)

Looper.loop()方法也是个无限循环，只有MessageQueue的next方法返回null的时候才退出，也就是调用Looper的quit方法或者quitSafely()方法时，MessageQueue的next方法会返回null，这个时候消息循环就终止了。


#### 4、Handler
 >Handler的主要工作是发送消息和接收消息，消息发送可以通过post的一系列方法以及send的一系列方法来实现，post的一系列方法的最终是通过send的一系列方法实现的。发一条消息的典型过程如下：
```
public final boolean sendMessage(Message msg){
    return sendMessageDelayed(msg, 0);
};

public final boolean sendMessageDelayed(Message msg, long delayMillis)
{
    if (delayMillis < 0) {
        delayMillis = 0;
    }
    return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
}


public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
    MessageQueue queue = mQueue;
    if (queue == null) {
        RuntimeException e = new RuntimeException(
                this + " sendMessageAtTime() called with no mQueue");
        Log.w("Looper", e.getMessage(), e);
        return false;
    }
    return enqueueMessage(queue, msg, uptimeMillis);
}

private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
    msg.target = this;
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}
```
可以发现Handler发送消息，仅仅是往消息队列中插入一条消息，MessageQueue的next方法会返回这个消息给Looper，Looper收到消息后就开始处理了，最终由Looper交由Handler处理，即Handler的dispathMessage方法被调用，这时Handler就进入了消息处理阶段，dispathMessage方法的实现如下：
```
public void dispatchMessage(Message msg) {
    if (msg.callback != null) {
        handleCallback(msg);
    } else {
        if (mCallback != null) {
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        handleMessage(msg);
    }
}
```
①首先检查 msg.callback对象，这是个 Runnable对象，即通过post方法所传递地参数，如果这个参数不为null，则调用这个参数的run方法（handleCallback(msg)这个调用就是调了Runable的run方法）。示例：
```
mHandler.post(new Runnable() {
    @Override
    public void run() {
        Log.d(TAG, "msg.callback != null ");
    }
});
```
②其次检查mCallback是否为空，不为空则调mCallback的handleMessage(msg)方法，这个mCallback是一个接口
```
/**
 * Callback interface you can use when instantiating a Handler to avoid
 * having to implement your own subclass of Handler.
 *
 * @param msg A {@link android.os.Message Message} object
 * @return True if no further handling is desired
 */
public interface Callback {
    public boolean handleMessage(Message msg);
}
```
示例：
```
private Handler handler = new Handler(new Handler.Callback(){
    @Override
    public boolean handleMessage(Message msg) {
        Log.d(TAG, "mCallback != null ");
        return true;
    }
});

handler.sendEmptyMessage(0);
```
![这里写图片描述](http://img.blog.csdn.net/20161205122729691)

![这里写图片描述](http://img.blog.csdn.net/20161205122807098)

- 创建Handler的几种方式：（其中①②都是属于派生子类，常用②③的方式）
①派生子类
```
private MyHandler myHandler = new MyHandler();

class MyHandler extends Handler{
    @Override
    public void handleMessage(Message msg) {
        super.handleMessage(msg);
    }
}
```

②匿名内部类
```
private Handler mHandler = new Handler(){
   @Override
    public void handleMessage(Message msg) {
        super.handleMessage(msg);
    }
};
```
③使用callback接口
```
private Handler handler = new Handler(new Handler.Callback(){
    @Override
    public boolean handleMessage(Message msg) {
        return false;
    }
});

```
- 另外Handler还有一个比较特殊的构造方法：
```
/**
 * Use the provided {@link Looper} instead of the default one.
 *
 * @param looper The looper, must not be null.
 */
public Handler(Looper looper) {
    this(looper, null, false);
}
```
这个构造方法可以指定Looper，我们知道，Looper是跟线程绑定的，换句话说就是Handler可以工作在任意一个线程，也就是不一定是主线程，当不在主线程时就不能用来更新UI。例子：（这样写就是使用子线程的Looper）
```
new Thread("Thread#4"){

    @Override
    public void run() {
        Looper.prepare();
        handlerInThread = new Handler(){
            @Override
            public void handleMessage(Message msg) {
                super.handleMessage(msg);
                textView.setText("子线程Handler更新UI");
            }
        };
        Looper.loop();
    }
}.start();

```

![这里写图片描述](http://img.blog.csdn.net/20161205122859869)

我们可以使用public Handler(Looper looper) 这个构造方法来指定Handler的Looper
```
new Thread("Thread#4"){

    @Override
    public void run() {
        Looper.prepare();
        handlerInThread = new Handler(getMainLooper()){
            @Override
            public void handleMessage(Message msg) {
                super.handleMessage(msg);
                //虽然这个Handler在子线程中创建，但其Looper是主线程的Looper,也就是这个方法是在主线程中执行的，当然可以更新UI
                textView.setText("子线程Handler更新UI");
            }
        };
        Looper.loop();
    }
}.start();
```
这样写也就意味着这个Handler是工作在主线程的，当然可以更新UI。所以，如果你的Handler是用来更新UI的，那么推荐使用new Handler(getMainLooper())这种方式来实现，不管你在哪里new这个对象，都可以用来更新UI，有时候这样可以减少不必要的麻烦。典型的例子是，我们在使用AsynTask时，必须在主线程去做类似这样的操作new MyAsycTask().execute();原因是在AsynTask内部会创建一个Handler，更新UI相关的操作都会借助这个Handler来切换回主线程，所以必须保证Handler的Looper是UI线程的。当然在较高的Android版本已经没有这个限制了。主要原因请看下面的源码：
```
在AsycTask内部有一个 private static InternalHandler sHandler; ，我们看一下InternalHandler的实现。
```
```
private static class InternalHandler extends Handler {
    public InternalHandler() {
        super(Looper.getMainLooper());
    }

    @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
    @Override
    public void handleMessage(Message msg) {
        AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
        switch (msg.what) {
            case MESSAGE_POST_RESULT:
                // There is only one result
                result.mTask.finish(result.mData[0]);
                break;
            case MESSAGE_POST_PROGRESS:
                result.mTask.onProgressUpdate(result.mData);
                break;
        }
    }
}
```
可以看到，创建这个sHandler的时候已经使用new Handler(getMainLooper())这种方式将主线程的Looper传进去，不管你在哪个线程创建，最终都会切换回到主线程去执行handleMessage方法。Android-22及更高版本是可以在子线程中new MyAsycTask().execute();的，Android-21版本及以下的应该都不可以。Android-21的InternalHandler如下：
```
private static class InternalHandler extends Handler {
    @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
    @Override
    public void handleMessage(Message msg) {
        AsyncTaskResult result = (AsyncTaskResult) msg.obj;
        switch (msg.what) {
            case MESSAGE_POST_RESULT:
                // There is only one result
                result.mTask.finish(result.mData[0]);
                break;
            case MESSAGE_POST_PROGRESS:
                result.mTask.onProgressUpdate(result.mData);
                break;
        }
    }
}
```
这个版本的源码使用的是默认的Handler的构造方法，也就是Looper是创建它的线程，所以要求在主线程中创建。

- 以上源码大部分来自 Android-24.