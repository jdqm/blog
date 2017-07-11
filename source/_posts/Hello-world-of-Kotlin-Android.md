---
title: Hello world of Kotlin-Android
date: 2017-07-11 23:15:25
updated: 2017-07-11 23:15:25
tags:
   -Kotlin
   -Android-Kotlin
categories: Kotlin
---

# 1.安装Kotlin插件

- windows : 打开Android Studio-->File-->settings-->Install JetBrains plugin，在搜索框中输入Kotlin，点击右边的Install安装，然后重启Android Studio即可。
<!-- more -->
![Settings](http://upload-images.jianshu.io/upload_images/3631399-83f19cfef7e1306c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- Mac: 打开Android Studio-->Android Studio-->preferences-->Plugins-->Install JetBrains plugin，在搜索框中输入Kotlin，然后在搜索结果中选择Kotlin插件安装，安装完成后重启Android studio 即可。


![Preferences](http://upload-images.jianshu.io/upload_images/3631399-2710a700c98419ea.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![Install Kotlin](http://upload-images.jianshu.io/upload_images/3631399-55a4354fcaf7b190.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 2.开始创建工程

创建一个普通的Android Project，但是要注意minSdkVersion至少是15，因为Kotlin有的库最低只兼容到15。

# 3.配置文件
- 修改project的build.grade 文件，参照下面的例子

```
buildscript {
    ext.support_version = '25.3.1'
    ext.kotlin_version = '1.1.2-5'
    ext.anko_version = '0.10.0'
    repositories {
        jcenter()
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:2.3.3'
        classpath "org.jetbrains.kotlin:kotlin-gradle-plugin:$kotlin_version"
    }
}

allprojects {
    repositories {
        jcenter()
    }
}
```

- 在你的主module的build.grade文件中加入如下内容

```
apply plugin: 'kotlin-android'
apply plugin: 'kotlin-android-extensions'
```
```
dependencies {
    compile "org.jetbrains.kotlin:kotlin-stdlib:$kotlin_version"
    compile "org.jetbrains.anko:anko-common:$anko_version"
}

```

tip:sync过程需要下载一些资源，速度取决与您的网络状态，耐心等待gradle执行完成。

到这里我就当你已经build成功，那好现在我们先看看MainActivity.java的代码

```
package com.jdqm.firstkotlin;

import android.app.Activity;
import android.os.Bundle;

public class MainActivity extends Activity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
    }
}
```

布局文件的内容

```
<?xml version="1.0" encoding="utf-8"?>
<android.support.constraint.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context="com.jdqm.firstkotlin.MainActivity">

    <TextView
        android:id="@+id/message"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Hello World!"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintLeft_toLeftOf="parent"
        app:layout_constraintRight_toRightOf="parent"
        app:layout_constraintTop_toTopOf="parent" />

</android.support.constraint.ConstraintLayout>

```

接下来我们执行一个神奇的操作。打开MainActivity.java，选择菜单栏的Code-->Convert java File to Kotlin File。这个时候你会神奇地发现MainActivity.java变成了MainActivity.kt，并且内容变成了下面的样子

```
package com.jdqm.firstkotlin

import android.app.Activity
import android.os.Bundle

class MainActivity : Activity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
    }
}
```

你会发现标记行结束的分号都不见了。接下来将这个文件修改一下，我们的HelloWorld of Kotlin就要大功告成了。修改结果如下：

```
package com.jdqm.firstkotlin

import android.app.Activity
import android.os.Bundle
import kotlinx.android.synthetic.main.activity_main.*

class MainActivity : Activity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        message.text = "Hello world of Kotlin!"
    }
}

```
其实就是添加了两行代码

```
import kotlinx.android.synthetic.main.activity_main.*
message.text = "Hello world of Kotlin!"
```

**Now, is time to run it.**


![demo效果](http://upload-images.jianshu.io/upload_images/3631399-3bc9a058a2c65d01.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Congratulations!

代码地址: [https://github.com/jdqm/FirstKotlin][2]


[1]: http://blog.shengyang.me
[2]: https://github.com/jdqm/FirstKotlin