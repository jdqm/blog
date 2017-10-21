---
title: 探索AIDL定向tag in out inout原理
date: 2017-10-21 12:24:44
updated: 2017-10-21 12:24:44
tags:
    -aidl
    -定向tag
    -in
    -out
    -inout
categories: Android
---
[上一篇文章《从一个例子开始分析AIDL原理》][1]分析了通过AIDL来完成跨进程通信的过程，文章最后抛出了一个问题: aidl语法中参数列表中的定向tag: in、out 和 inout 是什么含义？（阅读这篇文章需要有一定aidl基础，如果你看到云里雾里，可以先看看[上一篇文章][1]）
<!-- more -->
现在假设我不知道（尽管或多或少有一些猜想），通常的做法是谁写的这个东西就找谁，于是去到Android官网搜索aidl关键字，在长长的文章中躺着下面的一段话。
> All non-primitive parameters require a directional tag indicating which way the data goes. Either in, out, or inout. Primitives are in by default, and cannot be otherwise.
**Caution:** You should limit the direction to what is truly needed, because marshalling parameters is expensive.

大致的意思是说所有非基本数据类型的参数在传递的时候都需要指定一个方向tag来指明数据的流向，可以是in、out或者inout。基本数据类型默认是in，还不能修改为其他的。还附带了一个警告，说呀你应该根据实际需要来限制方向，因为参数的编排开销是很大的（毕竟实现需求追求性能是我们的目标）。

官网上也就这么几句话关于定向tag的说明，貌似并不能解释清楚。很迷茫，怎么办？如果你是狄仁杰还能来这么一句：元芳，你怎么看？这时元芳一脸胸有成竹地说：当线索断了，没有了头绪了，那就回归事情的本质，再寻找突破口。好吧，回忆一下使用aidl的过程。

创建aidl文件->Build->生成对应的java文件->实现其内部类Stub->onBind返回一个Stub实现类实例......

注意到一点aidl文件的作用是用来生成java文件，最终参与编译的是这个java文件，这貌似是一个突破口：只要在aidl文件中分别使用in、out和intout分别定义几个方法，看看生成的java文件的区别，从这个这角度似乎能反推出它们的含义？非常激动，根本停不下来，还是使用[上一篇文章][1]的例子:
```
// IDownloadCenter.aidl
package com.jdqm.downloadcenter.aidl;
import com.jdqm.downloadcenter.aidl.DownloadTask;

interface IDownloadCenter {
    //添加下载任务
    void addDownloadTaskIn(in DownloadTask task);
    void addDownloadTaskOut(out DownloadTask task);
    void addDownloadTaskInout(inout DownloadTask task);
}
```
在aidl文件中定义了三个方法，参数分别使用了in、out和inout这三个定向tag修饰，Build一下看看生成的java文件。

# 1、in
```
@Override
public void addDownloadTaskIn(com.jdqm.downloadcenter.aidl.DownloadTask task) throws android.os.RemoteException {
    android.os.Parcel _data = android.os.Parcel.obtain();
    android.os.Parcel _reply = android.os.Parcel.obtain();
    try {
        _data.writeInterfaceToken(DESCRIPTOR);
        if ((task != null)) {
        
            //注意这里参数不是null写入的是1
            _data.writeInt(1);
            
            //将传递的实参序列化到 _data
            task.writeToParcel(_data, 0);
        } else {
        
            //参数为null写入0
            _data.writeInt(0);
        }
        mRemote.transact(Stub.TRANSACTION_addDownloadTaskIn, _data, _reply, 0);
        _reply.readException();
    } finally {
        _reply.recycle();
        _data.recycle();
    }
}
```

可以看到当使用in定向tag时，我们传过来的参数被序列化到_data中，所以服务端的onTransact能收到我们在发起远程调用时传的参数。

```
case TRANSACTION_addDownloadTaskIn: {
    data.enforceInterface(DESCRIPTOR);
    com.jdqm.downloadcenter.aidl.DownloadTask _arg0;
    
    //通过读取tansact中写入的int值来判读是否有参数
    if ((0 != data.readInt())) {
    
        //将参数反序列化
        _arg0 = com.jdqm.downloadcenter.aidl.DownloadTask.CREATOR.createFromParcel(data);
    } else {
        _arg0 = null;
    }
    
    //调用目标方法
    this.addDownloadTaskIn(_arg0);
    reply.writeNoException();
    return true;
}
```
到这里我们得出了一个结论：使用in定向tag来修饰参数，在服务端目标方法可以得到一个与实参值相同的对象（注意不是同一个对象，它们在不同虚拟机实例中）。

# 2、out
```
@Override
public void addDownloadTaskOut(com.jdqm.downloadcenter.aidl.DownloadTask task) throws android.os.RemoteException {
    android.os.Parcel _data = android.os.Parcel.obtain();
    android.os.Parcel _reply = android.os.Parcel.obtain();
    try {
        _data.writeInterfaceToken(DESCRIPTOR);
        
        //这里根本没有将我们传过来的task写入 _data
        mRemote.transact(Stub.TRANSACTION_addDownloadTaskOut, _data, _reply, 0);
        _reply.readException();
        
        //_reply.readInt()这个值由服务端写入
        if ((0 != _reply.readInt())) {
        
            //从 _reply 中反序列化出服务端 onTransact 写入 reply 的值,所以我们传的实参被修改了
            task.readFromParcel(_reply);
        }
    } finally {
        _reply.recycle();
        _data.recycle();
    }
}
```
可以发现，使用out定向tag，我们传的实参不会传递到服务端，并且，如果 _reply.readInt()这个值不为0，我们传的参数还会被反序列化修改掉。出现疑问，既然参数不传到服务端，那服务端调用目标方法时的参数哪里来的？看看服务端onTransat:

```
case TRANSACTION_addDownloadTaskOut: {
    data.enforceInterface(DESCRIPTOR);
    com.jdqm.downloadcenter.aidl.DownloadTask _arg0;
    
    //没有从 data 中读取参数的过程，而是直接new了一个新的对象
    _arg0 = new com.jdqm.downloadcenter.aidl.DownloadTask();
    
    //将new出来的对象直接传给目标方法
    this.addDownloadTaskOut(_arg0);
    reply.writeNoException();
    if ((_arg0 != null)) {
    
        //有参数回写，标记1
        reply.writeInt(1);
        _arg0.writeToParcel(reply, android.os.Parcelable.PARCELABLE_WRITE_RETURN_VALUE);
    } else {
    
        //有参数回写，标记0
        reply.writeInt(0);
    }
    return true;
}
```
到这里刚才的疑问得到了解答，原来呀在服务端直接通过new关键字新建了一个对象，然后将这个新建的对象传给了目标方法，目标方法返回后将参数序列化到reply中返回。

又有得出了一些结论：使用定向 tag out 的参数，在发起远程调用时传入的实参不会传到到服务端，而是在服务端新建一个对象传给目标方法，待目标方法返回后将这个对象传回客户端。另外还发现一点，服务端是调用无参的构造方法来新建对象，这意味着你的参数所对应的类必须有有无参的构造方法。

# 3、inout
可能看完in、out这两个定向tag后，你已经开始猜想inout可能是它们的并集，到底是不是呢，接着往下看。

```
@Override
public void addDownloadTaskInout(com.jdqm.downloadcenter.aidl.DownloadTask task) throws android.os.RemoteException {
    android.os.Parcel _data = android.os.Parcel.obtain();
    android.os.Parcel _reply = android.os.Parcel.obtain();
    try {
        _data.writeInterfaceToken(DESCRIPTOR);
        if ((task != null)) {
            _data.writeInt(1);
            
            //将参数写入_data，这点与定向tag in一致
            task.writeToParcel(_data, 0);
        } else {
            _data.writeInt(0);
        }
        mRemote.transact(Stub.TRANSACTION_addDownloadTaskInout, _data, _reply, 0);
        _reply.readException();
        if ((0 != _reply.readInt())) {
            //将服务端写入reply的值反序列化，修改可客户端对象
            task.readFromParcel(_reply);
        }
    } finally {
        _reply.recycle();
        _data.recycle();
    }
}
```
可以看到，客户端确实是跟我们猜想的一样，将参数写进了_data（这点与in相同），调用task.readFromParcel(_reply)修改了我们传的参数（这点与out相同），再看看服务端。


```
case TRANSACTION_addDownloadTaskInout: {
    data.enforceInterface(DESCRIPTOR);
    com.jdqm.downloadcenter.aidl.DownloadTask _arg0;
    if ((0 != data.readInt())) {
    
        //从data中反序列化了客户端传过来的参数
        _arg0 = com.jdqm.downloadcenter.aidl.DownloadTask.CREATOR.createFromParcel(data);
    } else {
        _arg0 = null;
    }
    this.addDownloadTaskInout(_arg0);
    reply.writeNoException();
    if ((_arg0 != null)) {
        reply.writeInt(1);
        
        //将_arg0序列化到repley，客户端将反序列化出来
        _arg0.writeToParcel(reply, android.os.Parcelable.PARCELABLE_WRITE_RETURN_VALUE);
    } else {
        reply.writeInt(0);
    }
    return true;
}
```
服务端既读取了客户端传过来的数据（没有new对象，不具备out这点特性），并将参数回写给客户端。到这基本上你已经有了自己对这三个定向tag的结论，剩下就是该总结一下：

1. 定向tag in修饰的的参数，经序列化后传递服务端，服务端反序列化得到一个与之值相同的新的对象；
2. 定向tag out修饰的参数，客户端不会序列化该参数，而是服务端调用无参构造方法新建了一个对象，待目标方法返回后，将参数写入reply返回给客户端；
3. 定向tag inout基本上算是in、out的并集，为什么说基本上，因为out会在服务端通过new关键字来新建一个对象，而inout已经通过反序列化客户端传过来的数据得到一个新的对象，就没有别要再new一个了。


下面试图通过一些例子来验证上面的结论：

```
//aidl接口方法
void addDownloadTaskIn(in DownloadTask task);

//服务端实现
@Override
public void addDownloadTaskIn(DownloadTask task) throws RemoteException {
    //打印结果：{id=1, url='url of directional tag in'}
    //可以看到目标方法的参数确实与传的参数值是一样的
    Log.d(TAG, "addDownloadTaskIn: " + task);
    
    //修改参数的id字段
    task.setId(110);
}

//客户端调用
DownloadTask taskIn = new DownloadTask(1, "url of directional tag in");
downloadCenter.addDownloadTaskIn(taskIn);

//打印结果：{id=1, url='url of directional tag in'}
//服务端修改了id为110, 但客户端对象并没有被修改
Log.d(TAG, taskIn.toString());
```

```
//aidl接口方法
void addDownloadTaskOut(out DownloadTask task);

//服务端实现
@Override
public void addDownloadTaskOut(DownloadTask task) throws RemoteException {
    //打印结果：{id=0, url='null'}
    //可以看到，并不是调用方法是传的值，事实上是调用无参构造方法得到的新对象
    Log.d(TAG, "addDownloadTaskOut: " + task);

    //将task的两个字段
    task.setId(119);
    task.setUrl("change by service");
}

//客户端调用
DownloadTask taskOut = new DownloadTask(2, "url of directional tag out");
downloadCenter.addDownloadTaskOut(taskOut);

//打印结果：{id=119, url='change by service'}
//服务端修改了参数，客户端参数也被修改了
Log.d(TAG, taskOut.toString());

```

```
//aidl接口方法
void addDownloadTaskInout(inout DownloadTask task);

//服务端实现
@Override
public void addDownloadTaskInout(DownloadTask task) throws RemoteException {
    //打印结果：{id=3, url='url of directional tag inout'}
    //可以看到目标方法的参数确实与传的参数值是一样的
    Log.d(TAG, "addDownloadTaskInout: " + task);

    //将task的两个字段
    task.setId(120);
    task.setUrl("change by service");
}

//客户端调用
DownloadTask taskInout = new DownloadTask(3, "url of directional tag inout");
downloadCenter.addDownloadTaskInout(taskInout);
//打印结果：{id=120, url='change by service'}
//服务端修改了参数，客户端参数也被修改了
Log.d(TAG,  taskInout.toString());
```

实验结果证明，前面的分析是正确的。了解了这三个定向tag后，就可以根据自己的业务需求选择合适的tag了，其实大部分情况都是用的in。但是我还是有一个疑问，[上一篇文章][1]中提到，如果在方法前面上关键字 oneway，当客户端调用transact后，线程不会被挂起等待服务端返回，这似乎和定向tag out、inout矛盾了。举一个栗子：

```
oneway void addDownloadTaskOut(out DownloadTask task);
```
Build后直接报错了，inout也是一样不能再使用oneway。这样看来oneway指的是in这个way了。等等，如果方法有返回值呢？

```
oneway int addDownloadTaskIn(in DownloadTask task);
```
Build后依然报错。虽然你是in，但是有返回值也不行，所以这个oneway还得附加一个条件：返回值类型要是void。

>**注意：**以上这些结论也好例子也罢，都是建立在参数正确实现Parcelable接口的基础上的。


# AIDl有哪些语法
1. 有哪几种aidl文件？
一种是非默认支持的数据类型对应的aidl文件，如DownlaodTask.aidl，另外一种是定义接口的aidl。例如：

```
// DownloadTask.aidl
package com.jdqm.downloadcenter.aidl;

parcelable DownloadTask;
```

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
如果你使用的是默认支持的数据类型，那么第一类aild文件就不需要，所以你知道为啥叫“Interface Language”了吧。

2. 默认支持的数据类型有哪些？

（1）Java的所有基本数据类型，只支持定向tag in；
（2）String，只支持定向tag in；
（3）CharSequence，只支持定向tag in；
（4）List，其中元素必须是aidl支持的数据类型，包括（1）（2）（3）和其他aidl文件定义的parcelable；另外还有一点，接收方得到的总是ArrayList；
（5）Map，其中元素必须是aidl支持的数据类型，包括（1）（2）（3）和其他aidl文件定义的parcelable；另外还有一点，接收方得到的总是HashMap。

3. 必须使用Java语言来编写，同时也提供了一些关键字，如parcelable、onway、in、out、inout，这些关键字的含义以及用法在前面已经提到过。

4. 在使用自定义Parcelable类型时，必须使用import语句进行导入，即使是在同一个包中。

5. aidl接口中只能包含方法的定义，不能通过它来暴露一些静态字段。


最后，如果您有不同的见解，欢迎留下您的足迹，谢谢大家！




[1]: http://www.jianshu.com/p/b1ff8c0f3aec