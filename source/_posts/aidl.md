---
title: 从一个例子开始分析AIDL原理
date: 2017-10-21 12:18:30
updated: 2017-10-21 12:18:30
tags:
    -原理分析
    -aidl
    -跨进程通信
categories: Android
---
上一个项目（下载中心）使用到了AIDL相关的技术，趁现在项目不是特别繁忙，总结一下。首先第一个问题，AIDL是个啥东西？它的全称叫 Android Interface Definition Language，中文叫做安卓接口定义语言，这里面有两个关键字，“Interface”和“Language"，从这两个关键字来看它是一门用于定义接口的语言，既然是语言那自然就有它的语法与规则，但是本着先实现一个例子再回过头来学习语法的原则，[下一篇文章][3]再详细说明AIDL的语法。坦率讲，即使你不了解AIDL语法，基本上也能看懂，因为它与Java非常相似。下面通过一个例子来展示如何通过AIDL来实现跨进程通信（IPC）。
<!-- more -->
假设这样一个场景：有一个DownloadCenter，它可以向外提供下载服务，其他App有下载需求的话就可以通过它提供的服务来完成下载，这种方式非常类似于C/S模型。很明显DownloadCenter和其他App不是在同一个进程中，不能直接调用，就需要通过IPC机制来完成，而Android下的IPC方式有很多，比如：通过Inten传递数据、文件共享、Messenger、ContentProvider、aidl、Socket等，根据业务需求，选用aidl。

# 一.先写一个例子
我们通常把提供服务的一方叫做服务端，请求服务的一方叫做客户端，这里就把服务端命名为 DownloadCenter，客户端就叫 Client，接下来就开始实现这个例子。

(1) 创建一个普通的Android工程DownloadCenter，接着创建一个包 com.jdqm.downloadcenter.aidl，在这个包下创建一个DownloadTask类。
```
public class DownloadTask implements Parcelable{

    private int id;

    private String url;

    public DownloadTask() {
    }

    public DownloadTask(int id, String url) {
        this.id = id;
        this.url = url;
    }

    protected DownloadTask(Parcel in) {
        id = in.readInt();
        url = in.readString();
    }

    public static final Creator<DownloadTask> CREATOR = new Creator<DownloadTask>() {
        @Override
        public DownloadTask createFromParcel(Parcel in) {
            return new DownloadTask(in);
        }

        @Override
        public DownloadTask[] newArray(int size) {
            return new DownloadTask[size];
        }
    };

    @Override
    public int describeContents() {
        return 0;
    }

    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeInt(id);
        dest.writeString(url);
    }

    public void readFromParcel(Parcel in) {
        id = in.readInt();
        url = in.readString();
    }

    //重写toString方法方便打印
    @Override
    public String toString() {
        return "DownloadTask{" +
                "id=" + id +
                ", url='" + url + '\'' +
                '}';
    }
}

```
这个类代码看起来很多，其实就定义了两个成员变量id和url，并且实现Parcelable接口，其他的代码都是AS生成的。为什么要实现Parcelable接口？这是因为在跨进程通信过程中，涉及到对象数据的序列化和反序列化。关于Android中实现对象的序列化通常可以实现Parcelable或者Serializable接口，为了更好的理解建议还是先熟悉对象的序列化和反序列化相关的知识，这里就不再深入了。

插播一个场景：当初在学对象的序列化和反序列化的时候萌生了这样一个幻想，如今春节车票真的是一票难求，想回家的你费尽了心思也没买到一张二等座。假设有这么一个设备能把一个人（对象）序列化，然后通过网络传输到家里，再反序列化出来，这该是多么美好的一件事！

(2) 在(1)中的包名右键，New->AIDL->AIDL File 创建一个aild文件 DownloadTask.aidl
```
// DownloadTask.aidl
package com.jdqm.downloadcenter.aidl;

parcelable DownloadTask;
```
当你在(1)中的包名右键新建一个AIDL文件时，项目的目录结构中main下面多了一个aidl的文件夹，并且会帮你创建一个与你右键的地方相同的包名。紧接着在相同的包名的创建IDownloadCenter.aidl
```
// IDownloadCenter.aidl
package com.jdqm.downloadcenter.aidl;

interface IDownloadCenter {
    //添加下载任务
    void addDownloadTask(in DowloadTask task);
    
    //查询所有的添加的下载任务
    List<DownloadTask> getDownloadTask();
}
```
到这里，你需要Build一下项目，一来是检查有没有错误，更重要的是让Android SDK Tool根据aidl文件生成对应的Java文件，如果Build成功，那么会在app/build/generated/source/aidl/debug下生成IDownloadCenter.java这个文件（切换到Project视图，或者直接double shift搜索更快）。

(3) 创建服务 DownloadCenterService

```
public class DownloadCenterService extends Service {

    private List<DownloadTask> tasks;

    private DownloadCenter downloadCenter;

    @Override
    public void onCreate() {
        super.onCreate();
        
        //由于是在Binder线程池中访问这个集合，所以有必要好线程同步。除非你能确保并发情况下不会出现问题
        tasks = Collections.synchronizedList(new ArrayList<DownloadTask>());
        downloadCenter = new DownloadCenter();
    }

    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return downloadCenter;
    }

    /**
     * 完成aidl文件编写后，下次Build时Android SDK tools会根据IDownloadCenter.aidl文件生成
     * IDownloadCenter.java文件，Stub是它的内部类
     */
    private class DownloadCenter extends IDownloadCenter.Stub {

        @Override
        public void addDownloadTask(DownloadTask task) throws RemoteException {
            tasks.add(task);
        }

        @Override
        public List<DownloadTask> getDownloadTask() throws RemoteException {
            return tasks;
        }
    }
}
```
这个Service其实也很简单，首先创建一个IDownloadCenter.Stub的实现类DownloadCenter，并且实现其内部的两个抽象方法（可以看到这两个方法就是在aidl文件中声明的方法），然后在onBind()方法将其返回它的一个实例。

(4) 最后别忘了在AndroidManifest.xml文件中注册这个Service，另外为了让其他应用通过隐式Intent启动，需要给这个Service添加一个intent-filter
```
<service android:name=".DownloadCenterService">
    <intent-filter>
        <action android:name="jdqm.intent.action.LAUNCH"/>
        <category android:name="android.intent.category.DEFAULT"/>
    </intent-filter>
</service>
```

接下来是客户端Client的实现：

(1) 首先创建一个名为Client的Android工程，然后将服务端的所有aidl文件以及aidl文件中使用到的Java类拷贝到Cilent中，注意包名结构也要和服务端一致，如果不一致在反序列化的时候就会出错。

(2) 通过bindService与服务端建立连接，完成服务方法调用
```
public class MainActivity extends AppCompatActivity implements View.OnClickListener {

    private static final String TAG = "MainActivity";

    private Button btnAddTask;
    private Button btnGetTasks;

    private DownloadServiceConn serviceConn;
    private IDownloadCenter downloadCenter;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        initViews();
        serviceConn = new DownloadServiceConn();
        Intent intent = new Intent("jdqm.intent.action.LAUNCH");
        intent.setPackage("com.jdqm.downloadcenter");
        bindService(intent, serviceConn, BIND_AUTO_CREATE);
    }

    private void initViews() {
        btnAddTask = findViewById(R.id.btnAddTask);
        btnGetTasks = findViewById(R.id.btnGetTasks);
        btnAddTask.setOnClickListener(this);
        btnGetTasks.setOnClickListener(this);
        btnAddTask.setEnabled(false);
        btnGetTasks.setEnabled(false);
    }

    @Override
    public void onClick(View v) {
        switch (v.getId()) {
            case R.id.btnAddTask:
                try {
                    Random random = new Random();
                    int id = random.nextInt(10);
                    DownloadTask task = new DownloadTask(id, "http://test.jdqm.com/test.aidl");
                    
                    //在客户端直接调用IBinder接口的方法，最终服务端对应的方法会调用，这似乎是在同一个进程中一般
                    //这是因为Binder机制已经把底层的实现隐藏掉了
                    downloadCenter.addDownloadTask(task);
                    Log.d(TAG, "添加任务成功: " + task);
                } catch (RemoteException e) {
                    e.printStackTrace();
                    Log.e(TAG, "添加任务失败");
                }
                break;
            case R.id.btnGetTasks:
                try {
                    List<DownloadTask> tasks = downloadCenter.getDownloadTask();
                    Log.d(TAG, "从服务中获取到已经添加的任务列表: " + tasks);
                } catch (RemoteException e) {
                    e.printStackTrace();
                }
                break;
            default:
                break;
        }
    }

    private class DownloadServiceConn implements ServiceConnection {

        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            Log.d(TAG, "服务已连接");

            //将绑定服务的返回的IBinder对象转换为目标接口类型
            downloadCenter = IDownloadCenter.Stub.asInterface(service);
            btnAddTask.setEnabled(true);
            btnGetTasks.setEnabled(true);
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            Log.d(TAG, "服务已断开");
            btnAddTask.setEnabled(false);
            btnGetTasks.setEnabled(false);
        }
    }

    @Override
    protected void onDestroy() {
        unbindService(serviceConn);
        super.onDestroy();
    }
}
```

以上是完整的客户端实现，首先bindService，通过ServiceConnection完成绑定时的回调方法onServiceConnected拿到服务端返回的IBinder对象，通过IDownloadCenter.Stub.asInterface()方法将IBinder转换为目标IBinder类型IDownloadCenter，通过它来完成IDownloadCenter.Stub实现类方法的调用。

现在将DownloadCenter和Client安装到同一个Android设备中，打开客户端，点击界面里的按钮，下面是输出的log信息：
```
/com.jdqm.client D/MainActivity: 服务已连接
/com.jdqm.client D/MainActivity: 添加任务成功: DownloadTask{id=6, url='http://test.jdqm.com/test.aidl'}
/com.jdqm.client D/MainActivity: 从服务中获取到已经添加的任务列表: [DownloadTask{id=6, url='http://test.jdqm.com/test.aidl'}]
/com.jdqm.client D/MainActivity: 添加任务成功: DownloadTask{id=5, url='http://test.jdqm.com/test.aidl'}
/com.jdqm.client D/MainActivity: 从服务中获取到已经添加的任务列表: [DownloadTask{id=6, url='http://test.jdqm.com/test.aidl'}, DownloadTask{id=5, url='http://test.jdqm.com/test.aidl'}]
```

到这里，我们完成了客户端与服务端的跨进程通信。虽然没有像唐僧西天取经那样历经九九81难（何苦呢），但也可以回头看看我们走过路了。
# 二.回头看看

先来总结一下步骤：

1. 在src/main/aidl/下创建了两个 aidl文件：DownloadTask.aidl、IDownloadCenter.aidl;
2. 创建一个IDownloadCenter.Stub实现类，并在onBind中返回一个实例: DownloadCenter extends IDownloadCenter.Stub;
3. 在客户端实现一个ServiceConnection;
4. Context.bindService(), 在回调onServiceConnected()中接收IBinder实例， 并且调用IDownloadCenter.Stub.asInterface(service)将返回值转换为IDownloadCenter类型。
5. 调用IDownloadCenter接口中定义的方法来完成跨进程调用；
6. 调用Context.unbindService()断开连接。

![两个项目结构](http://upload-images.jianshu.io/upload_images/3631399-dd27b706648a0742.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

粗略地总结好像并没什么意思，毕竟大多数真谛是隐藏在细节中的。就好比让你回忆初恋的过程，可能你更想回忆起接吻时的感觉！ 首先从IDownloadCenter.aidl生成的IDownloadCenter.java开始。

```
public interface IDownloadCenter extends android.os.IInterface {

    public static abstract class Stub extends android.os.Binder implements com.jdqm.downloadcenter.aidl.IDownloadCenter {
       //省略器内部实现
    }
    
    //添加下载任务
    public void addDownloadTask(com.jdqm.downloadcenter.aidl.DownloadTask task) throws android.os.RemoteException;
    
    //查询所有的添加的下载任务
    public java.util.List<com.jdqm.downloadcenter.aidl.DownloadTask> getDownloadTask() throws android.os.RemoteException;
}
```
打开这个文件代码看起来是真的有点多（130多行），而且格式不太好阅读，所以最好是先格式化一下。所幸的是它的结构很清晰，首先它是一个接口，内部定义了两个接口方法和一个内部类Stub，显然这两个方法就是在aidl文件中定义的方法，连注释都给你搬过来了，可见这个生成工具是如此地用心。重点就是这个内部类Stub，它是一个抽象类，并且实现了IDownloadCenter接口，但它并没有实现接口中的两个方法。还记得在Service中IDownloadCenter.Stub的实现类吗？
```
private class DownloadCenter extends IDownloadCenter.Stub {

    @Override
    public void addDownloadTask(DownloadTask task) throws RemoteException {
        tasks.add(task);
    }

    @Override
    public List<DownloadTask> getDownloadTask() throws RemoteException {
        return tasks;
    }
}
```

接着我们在Service的onBind方法返回了它的一个实例，创建这个Binder的过程做了什么？那就得窥探窥探Stub的真容了。
```
public static abstract class Stub extends android.os.Binder implements com.jdqm.downloadcenter.aidl.IDownloadCenter {
        private static final java.lang.String DESCRIPTOR = "com.jdqm.downloadcenter.aidl.IDownloadCenter";

        public Stub() {
            this.attachInterface(this, DESCRIPTOR);
        }

        public static com.jdqm.downloadcenter.aidl.IDownloadCenter asInterface(android.os.IBinder obj) {
            if ((obj == null)) {
                return null;
            }
            android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
            if (((iin != null) && (iin instanceof com.jdqm.downloadcenter.aidl.IDownloadCenter))) {
                return ((com.jdqm.downloadcenter.aidl.IDownloadCenter) iin);
            }
            return new com.jdqm.downloadcenter.aidl.IDownloadCenter.Stub.Proxy(obj);
        }

        @Override
        public android.os.IBinder asBinder() {
            return this;
        }

        @Override
        public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
            //省略内部实现
        }

        private static class Proxy implements com.jdqm.downloadcenter.aidl.IDownloadCenter {
            //省略内部实现
        }

        static final int TRANSACTION_addDownloadTask = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
        static final int TRANSACTION_getDownloadTask = (android.os.IBinder.FIRST_CALL_TRANSACTION + 1);
    }
```

可以看到，在Stub的构造方法中调了this.attachInterface(this, DESCRIPTOR)方法，这个DESCRIPTOR就是IDownloadCenter的完整名称。

Binder#attachInterface
```
public void attachInterface(IInterface owner, String descriptor) {
    mOwner = owner;
    mDescriptor = descriptor;
}
```
可以看到把当前对象保存到Binder的mOwner，将接口的完整名称保存到Binder的mDescriptor。至此onBind返回了这个对象，交由底层的Binder机制来处理，底层原理之复杂度犹如乾坤大挪移一般深噢，将逻辑转到了客户端。

```
@Override
public void onServiceConnected(ComponentName name, IBinder service) {
    Log.d(TAG, "服务已连接" );
    Log.d(TAG, "ComponentName: " + name.getClassName()); //com.jdqm.downloadcenter.DownloadCenterService
    Log.d(TAG, "service: " + service.getClass().getName()); //android.os.BinderProxy
    //将绑定服务的返回的IBinder对象转换为目标接口类型
    downloadCenter = IDownloadCenter.Stub.asInterface(service);
    btnAddTask.setEnabled(true);
    btnGetTasks.setEnabled(true);
}
```
这个方法是在客户端与服务端建立连接完成时回调，这个service到底是什么，是Service中onBind返回的DownloadCenter吗？通过 service.getClass().getName()发现它是一个BinderProxy，接着调用IDownloadCenter.Stub.asInterface(service)，它的实现如下：

```
public static com.jdqm.downloadcenter.aidl.IDownloadCenter asInterface(android.os.IBinder obj) {
    if ((obj == null)) {
        return null;
    }
    
    //这个obj是BinderProxy
    android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR); //返回null
    if (((iin != null) && (iin instanceof com.jdqm.downloadcenter.aidl.IDownloadCenter))) {
        return ((com.jdqm.downloadcenter.aidl.IDownloadCenter) iin);
    }
    return new com.jdqm.downloadcenter.aidl.IDownloadCenter.Stub.Proxy(obj);
}
```
首先通过调用了BinderProxy.queryLocalInterface

```

final class BinderProxy implements IBinder {

    public IInterface queryLocalInterface(String descriptor) {
        return null;
    }
}
```
它的返回值是null，即 iin 为null，所以 IDownloadCenter.Stub.asInterface(service)得到的是IDownloadCenter.Stub.Proxy的实例。

通过asInterface方法可以得出一个结论：

- 如果服务端与客户端在同一个进程，那么在onServiceConnected方法得到的service就是我们在onBind中返回的类型IDownloadCenter.Stub；
- 如果不是同一个进程，那得到的就是IDownloadCenter.Stub.Proxy类型；

拿到了IDownloadCenter.Stub.Proxy，那就可以调用它的方法了 downloadCenter.addDownloadTask(task)，下面是IDownloadCenter.Stub.Proxy的实现：
```
public static abstract class Stub extends android.os.Binder implements com.jdqm.downloadcenter.aidl.IDownloadCenter {
        
        //省略前面代码
        @Override
        public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
            switch (code) {
                case INTERFACE_TRANSACTION: {
                    reply.writeString(DESCRIPTOR);
                    return true;
                }
                case TRANSACTION_addDownloadTask: {
                    data.enforceInterface(DESCRIPTOR);
                    com.jdqm.downloadcenter.aidl.DownloadTask _arg0;
                    if ((0 != data.readInt())) {
                        _arg0 = com.jdqm.downloadcenter.aidl.DownloadTask.CREATOR.createFromParcel(data);
                    } else {
                        _arg0 = null;
                    }
                    this.addDownloadTask(_arg0);
                    reply.writeNoException();
                    return true;
                }
                case TRANSACTION_getDownloadTask: {
                    data.enforceInterface(DESCRIPTOR);
                    java.util.List<com.jdqm.downloadcenter.aidl.DownloadTask> _result = this.getDownloadTask();
                    reply.writeNoException();
                    reply.writeTypedList(_result);
                    return true;
                }
            }
            return super.onTransact(code, data, reply, flags);
        }

        private static class Proxy implements com.jdqm.downloadcenter.aidl.IDownloadCenter {
            private android.os.IBinder mRemote;

            Proxy(android.os.IBinder remote) {
                mRemote = remote;
            }

            @Override
            public android.os.IBinder asBinder() {
                return mRemote;
            }

            public java.lang.String getInterfaceDescriptor() {
                return DESCRIPTOR;
            }
            
            //添加下载任务
            @Override
            public void addDownloadTask(com.jdqm.downloadcenter.aidl.DownloadTask task) throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    if ((task != null)) {
                    
                        //注意这里写入的是1
                        _data.writeInt(1);
                        
                        //序列化我们传入的实参，到这里你知道为什么跨进程通信的对象需要实现Parcelable接口了吧
                        task.writeToParcel(_data, 0);
                    } else {
                        _data.writeInt(0);
                    }
                    
                    //这个mRemote就是是BinderProxy
                    mRemote.transact(Stub.TRANSACTION_addDownloadTask, _data, _reply, 0);
                    _reply.readException();
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
            }
            
            //查询所有的添加的下载任务
            @Override
            public java.util.List<com.jdqm.downloadcenter.aidl.DownloadTask> getDownloadTask() throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                java.util.List<com.jdqm.downloadcenter.aidl.DownloadTask> _result;
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    mRemote.transact(Stub.TRANSACTION_getDownloadTask, _data, _reply, 0);
                    _reply.readException();
                    _result = _reply.createTypedArrayList(com.jdqm.downloadcenter.aidl.DownloadTask.CREATOR);
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
                return _result;
            }
        }

        static final int TRANSACTION_addDownloadTask = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
        static final int TRANSACTION_getDownloadTask = (android.os.IBinder.FIRST_CALL_TRANSACTION + 1);
    }

```
从Proxy#addDownloadTask方法可以看到，通过 task.writeToParcel(_data, 0)，将我们传入的DownloadTask对象进行了序列化，终于知道为啥前面写的DownloadTask要实现Parcelable接口了。接着调用 mRemote.transact(Stub.TRANSACTION_addDownloadTask, _data, _reply, 0)方法，通过前面创建Proxy对象的过程，我们知道这个mRemote是BinderProxy，这样逻辑就交给了底层的Binder机制，服务端的onTransact就会被调用。接下来就看看onTransact方法中 TRANSACTION_addDownloadTask 这个case的实现：

```
public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {

    switch (code) {
        case TRANSACTION_addDownloadTask: {
            data.enforceInterface(DESCRIPTOR);
            com.jdqm.downloadcenter.aidl.DownloadTask _arg0;
            if ((0 != data.readInt())) {
                _arg0 = com.jdqm.downloadcenter.aidl.DownloadTask.CREATOR.createFromParcel(data);
            } else {
                _arg0 = null;
            }
            this.addDownloadTask(_arg0);
            reply.writeNoException();
            return true;
        }
    }
    //省略其他内容
}            
```

首先 if ((0 != data.readInt())) 这个条件是true，因为我们在Proxy#addDownloadTask中写入的是1。但是这里我有一个疑问，为啥这个if语句多了一层圆括号？你知道吗？然后通过DownloadTask.CREATOR.createFromParcel(data)将数据反序列化并赋值给 _arg0，接着调用 this.addDownloadTask(_arg0)，这个this不就是我们在onBind返回的Stub实例吗！就这样服务端方法被调用了。

另外一个方法getDownloadTask()方法的调用过程也是类似，有区别的是它有返回值，所以onTransact()方法时多了 reply.writeTypedList(_result)，回到transact()方法时将结果反序列化 _result = _reply.createTypedArrayList(com.jdqm.downloadcenter.aidl.DownloadTask.CREATOR)。下面给出一张粗略的工作流程图辅助理解这个过程：

![Binder](http://upload-images.jianshu.io/upload_images/3631399-3dee016fcfb2cb52.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 三.最后

- 在这个例子中，我们是在主线程中调用服务方法downloadCenter.addDownloadTask(task)，在执行mRemote.transact(Stub.TRANSACTION_getDownloadTask, _data, _reply, 0)时调用线程会被挂起，所以如果不能确保远程服务方法的是否耗时时，应该避免在主线程中调用。当然了，aidl还提供了一个关键字 oneway，定义方法时加上这个关键字就不用等待服务端方法返回了，如果这满足你的业务不需要等待结果，那也是一个不错的选择。
```
//添加下载任务
oneway void addDownloadTask(in DownloadTask task);
```
- 另外一点就是服务端方法是运行在Binder线程池中的，要考虑好线程同步。


最后并不代表结束了，可能又是一个新的开始。细心的读者可能会留意到前面我们写的aidl文件：
```
// IDownloadCenter.aidl
package com.jdqm.downloadcenter.aidl;

import com.jdqm.downloadcenter.aidl.DownloadTask;

interface IDownloadCenter {
    //添加下载任务
    void addDownloadTask(in DownloadTask task);

    //查询所有的添加的下载任务
    List<DownloadTask> getDownloadTask();
}
```

其中 addDownloadTask 这个方法的参数前面有一个in，这个 in 是什么？这是一个定向tag，事实上定向tag有3个in、out、inout，它们又是什么含义？[下一篇文章][3]将从零开始探索这3个定向tag的含义以及用法。

最后，如果你有不同的见解，欢迎留下您的足迹，谢谢大家！

[客户端源码][1] 
[服务端源码][2]

[1]: https://github.com/jdqm/blog-code/tree/master/Client
[2]: https://github.com/jdqm/blog-code/tree/master/DownloadCenter
[3]: http://www.jianshu.com/p/382633129b53