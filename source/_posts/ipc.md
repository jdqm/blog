---
title: IPC基础
date: 2017-10-21 12:12:39
updated: 2017-10-21 12:12:39
tags:
    -IPC
    -跨进程通信
categories: Android
---

# 1.IPC
Inter-Process Communication，即进程间通信或者跨进程通信。
<!-- more -->
# 2.进程与线程
进程与线程是不同的概念，按照操作系统中的描述，线程是CUP调度的最小单元，而进程一般是指一个执行单元，对于PC和移动设备来说，通常指一个程序或者应用。一个进程可以包含多个线程，即进程与线程是包含与被包含的关系。
# 3.什么情况下会出现多进程
①一个应用存在多进程；②多个应用之间构成多进程。
在Android中使用多进程常规的方法只有一种，那就是在注册四大组件时在AndroidManifest.xml中为四大组件声明属性 android:process属性来指定该组件在哪个进程中工作。例如本例中创建了3个Activity，它的AndroidManifest.xml文件如下：
```
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.jdqm.ipcdemo">

    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/AppTheme">
        <activity android:name=".MainActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />

                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>

        <activity android:name=".SecondActivity"
            android:process=":remote" />

        <activity android:name=".ThirdActivity"
            android:process="com.jdqm.ipcdemo.remote"/>
    </application>

</manifest>
```
其中为SecondActivity指定在新的进程 “包名:remote“中工作，ThirdActivity在 “com.jdqm.ipcdemo.remote”进程中工作，启动应用，并依此启动这个三个Activity，通过菜单Tools-->Android-->Android Devices monitor来查看当前设备的工作进程。

![Android Devices monitor查看系统进程信息](http://upload-images.jianshu.io/upload_images/3631399-d86467fc5e140878.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

8600、8603、8614这三个进程就是该应用新建的进程。除了以上方式外，我们还可以通过命令行的方式去查看进程信息： adb shell ps | grep com.jdqm.ipcdemo，其中com.jdqm.ipcdemo 是包名。

![命令行方式查看系统进程信息](http://upload-images.jianshu.io/upload_images/3631399-208b4455ef16bd0e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

注：另外还有一种比较特殊的方式去创建多进程，那就是在jni层去fork新的进程。

在上面的例子中，android:process=":remote"与android:process="com.jdqm.ipcdemo.remote"有什么不同？
①前者会在冒号前添加当前应用的包名，即完整进程名为：com.jdqm.ipcdeo:remote；
②前者是当前应用的私有进程，其他的应用不能访问，而后者可以通过SharedUID方式与它跑在同一个进程中。

# 4.使用多进程会带来什么问题？
①静态成员和单例模式失效（每个进程有独立的虚拟机）；
②线程同步机制失效（既然不是同一块内存，不管是锁对象还是锁全局类都无法保证线程的同步）；
④SharedPreferences可靠新下降（并发读写都有可能出现问题）；
④Application会多次创建（系统创建一个新的进程，就会分配一个新的虚拟机，这个过程就是应用启动的过程）；

# 5.进程间通信有哪些方式？

Intent传递数据，共享文件和SharedPreferences，基于Binder的Messenger和AIDL以及Socket等。

# 6.Android实现对象的序列化和反序列化的两种方式
①实现erializable接口
```
public class User implements Serializable {

    private static final long serialVersionUID = 1L;

    private String name;
   
    private int age;

    public User(String name, int age) {
        this.name = name;
        this.age = age;
    }

    @Override
    public String toString() {
        return "User{" +
                "name='" + name + '\'' +
                ", age=" + age +
                '}';
    }
}
```
```
ObjectOutputStream out = new ObjectOutputStream(new FileOutputStream("/mnt/sdcard/cache.txt"));
User u = new User("张三", 18);
out.writeObject(u);
out.close();
```
```
ObjectInputStream in = new ObjectInputStream(new FileInputStream("/mnt/sdcard/cache.txt"));

User u = (User) in.readObject();
in.close();
```
通过Serrializable接口实现序列化和反序列化比较简单，只要该类实现Serializable接口即可，那其中的 “private static final long serialVersionUID = 1L”指定的这个值有什么用？serialVersionUID 的工作机制是这样的：在序列化时会把这个serialVersionUID写入到序列化文件中（也可能是其他中介），当反序列化时系统会去检查这个值与当前类的serialVersionUID是否相等，若相等则说明序列化时该类的版本与当前的版本是一致的，这个时候是能成功被反序列化的；否则说明当前类的结构发生了某些变换，比如类的成员变量的数量、类型发生了改变，这个时候是无法正常被反序列化的。当然这个值如果不指定的话，系统会根据类的结构计算一个Hash值，一旦这个类发送任何改变都会引起这个值得改变。有时候虽然我们的类发生了改变，但是我们仍可以利用反序列化来恢复原来的数据，这个时候必须要手动声明serialVersionUID。例如：假设序列化的时候类的结构是这样子的:
```
public class Student implements Serializable{

    private static final long serialVersionUID = 1L;
    
    private String name;
    private int age;

    public Student(String name, int age) {
        this.name = name;
        this.age = age;
    }
}
```
通过序列化写入文件
```
ObjectOutputStream out = new ObjectOutputStream(new FileOutputStream("/mnt/sdcard/student.txt"));
Student stu = new Student("小明", 18);
out.writeObject(stu);
out.close();
```
这个时候我们去改变类的结构，增加一个字段 "private float score;"，这个时候类的结构发生了改变，但是我们指定serialVersionUID 不变，“欺骗”系统。
```
public class Student implements Serializable{

    private static final long serialVersionUID = 1L;

    private String name;

    private int age;
   
     private float score;
   
     public Student(String name, int age) {
        this.name = name;
        this.age = age;
    }

    @Override
    public String toString() {
        return "Student{" +
                "name='" + name + '\'' +
                ", age=" + age +
                ", score=" + score +
                '}';
    }
}
```
在这种情况下去反序列化
```
ObjectInputStream in = new ObjectInputStream(new FileInputStream("/mnt/sdcard/student.txt"));

Student stu2 = (Student) in.readObject();
//name='小明', age=18, score=0.0
Log.d(TAG, "onCreate: " + stu2.toString()); 
in.close();
```
可以看到原来的两个字段被正确的反序列化，而增加的字段则是一个默认值。假设序列化时的serialVersionUID与反序列化时的不一致，则会报如下异常：
```
java.io.InvalidClassException: com.jdqm.ipcdemo.Student; Incompatible class (SUID): com.jdqm.ipcdemo.Student: static final long serialVersionUID =1L; but expected com.jdqm.ipcdeom.Student: static final long serialVersionUID =2L;
```

②Parcelable
只要实现这个接口的类的对象就可以通过Intent和Binder来传递。
```
public class Person implements Parcelable {

    private String name;

    private int age;

    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }

    /**
     *返回当前对象的内容描述
     * @return 0或者1，当含有文件描述符时返回1，其他情况0，几乎所有的情况都是返回0
     */
    @Override
    public int describeContents() {
        return 0;
    }

    /**
     *
     * @param dest
     * @param flags 这个标记为可以是0或者1，为1时标示当前对象需要作为返回值返回，不能立即释放。几乎所有情况都是0
     */
    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeString(name);
        dest.writeInt(age);
    }

    public static Parcelable.Creator<Person> CREATOR = new Parcelable.Creator<Person>(){


        @Override
        public Person createFromParcel(Parcel source) {
            return new Person(source);
        }

        @Override
        public Person[] newArray(int size) {
            return new Person[size];
        }
    };

    private Person(Parcel source) {
        name = source.readString();
        age = source.readInt();
    }

    @Override
    public String toString() {
        return "Person{" +
                "name='" + name + '\'' +
                ", age=" + age +
                '}';
    }
}
```
# 7.Serializable与Parcelable有什么区别？
Serializable是java的序列化接口，使用起来简单但是开销大，会伴随着大量的I/O操作，而Parcelable是Android中的序列化方式，稍显复杂但是效率更高。Parcelable主要用于内存序列化上，通过Parcelable将对象序列化到存储设备上或者序列化后通过网络传输也是可以的，但是过程会稍显复杂，这两种情况下建议使用Serializable。

#8.Binder
Binder是一个类，实现了IBinder接口。从IPC的角度来说，Binder是Android的一种跨进程通信方式。Binder还可以理解为一种虚拟的物理设备，它的设备驱动是/dev/binder。

#9.Binder很重要的两个方法linkToDeath和unlinkToDeath
当我们发起RPC的过程中，由于某种原因服务端由于异常终止了，这个时候客户端到服务端的Binder连接就断了，这样就导致远程调用失败，如果我们不知道连接已经断了，那就会影响客户端。为了解决这个问题，Binder提供一配对的方法linkToDeath和unlinkToDeath，通过linkToDeath我们可以给Binder设置死亡代理，当Binder死亡时，就会收到通知，这个时候就可以选择重连或者别的操作。那么问题来了，如何给Binder设置死亡代理？

1）声明一个DeathRecipient对象
```
private IBinder.DeathRecipient mDeathRecipient = new IBinder.DeathRecipient() {
    @Override
    public void binderDied() {
        if (bookManager == null) {
            return;
        }
        Log.d(TAG, "binderDied: ");
        bookManager.asBinder().unlinkToDeath(mDeathRecipient, 0);
        bookManager = null;
        Intent intent = new Intent(MainActivity.this, BookManagerService.class);
        bindService(intent, connection,BIND_AUTO_CREATE);
    }
};
```

2）在客户端绑定服务成功后，给放回的Binder设置死亡代理

```
@Override
public void onServiceConnected(ComponentName name, IBinder service) {
    Log.d(TAG, "onServiceConnected: ");
    bookManager = IBookManage.Stub.asInterface(service);
    try {
        bookManager.getBookList();
    } catch (RemoteException e) {
        e.printStackTrace();
    }
    try {
        service.linkToDeath(mDeathRecipient, 0);
    } catch (RemoteException e) {
        e.printStackTrace();
    }
}
```
另外，可以通过binder的isBinderAlive方法查看binder时候还“活着”。