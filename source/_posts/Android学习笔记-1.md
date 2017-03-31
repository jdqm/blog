---
title: Android学习笔记(1)
date: 2017-01-31 16:59:47
tags:
	- Android学习笔记
	- Activity启动模式
	- intent-filter
categories:
---

### 一、Activity的生命周期
#### 1、正常的生命周期

-  **onCreate()：**
> 表示Activity正在被创建，是生命周期中的第一个方法，可以在这个方法中做一些初始化工作，如调用setContentView()去加载布局资源，初始化Activity所需的数据，当然如果加载的数据比较多或者比较耗时的操作建议采用异步加载的方式，加快界面的出现，待异步加载完成再去设置view的属性。

<!--more-->

-  **onRestart():**
> 表示Activity正在被重新创建，一般情况下当Activity由stop状态重新变为可见时，onRestart被调用。

- **onStart()：**
> 表示Activity正在启动，即将开始，这个时候Activity已经可见，只是还没出现在前台，所以看不见。可见与看见还是有区别的。

- **onResume()：**
> 表示Activity已经可见，获取焦点（在前台）可与用户交互了。
- **onPause()：**
> 表示Activity正在失去焦点。应用场景：游戏暂停时在该方法保存游戏数据，在onResume中恢复。

- **onStop()：**
> 表示Activity正在被停止，这个时候可以做一些稍重量级的操作，同样不能太耗时。

- **onDestroy()：**
> 表示Activity即将销毁，是生命周期中最后的一个回调方法，我们可以在这里做一些回收工作和最终的资源释放。

- **完整的生命周期**：
> onCreate()-->onStart()-->onResume()-->onPause()-->onStop()-->onDestroy()

- **可见周期：**
> onStart()-->onResume()-->onPause()-->onStop()期间

- **前台（获得焦点）周期：**
> onResume()-->onPause()期间

- 有一个比较特殊的方法onRestart()会在Activity有不可见变为可见是调用，即当一个Activity处于stop状态时，再次变为可见时：onRestart()-->onStart()-->onResume()

- 当前Activity onPause()之后启动的Activity才能onResume,所以不要再在onPause方法做一些重量级的操作。

#### 2、异常情况下的生命周期

 > 到底什么情况下会出现异常？

-  ①资源相关的系统配置发生变化导致Activity被销毁重建，如横竖屏切换（在不处理的情况下会销毁重建）；
- ②系统资源内存不足，导致较低优先级的Activity被暂时回收；
 
- 这种情况下的生命周期为：
> onSaveInstanceState()-->onDestroy()
  onCreate()-->onRestoreInstanceState()

当Activity被异常终止的情况下，在onStop()之前会调用onSaveInstanceState()来保存数据，这些数据会以Bundle的形式传到onCreate()和onRestoreInstanceState(),onRestoreInstanceState()的调用时机在onStart()之后。当Activity被异常终止的时候系统会自动保存一些数据，当然我们有时也需要自己保存一些数据，具体系统会自动保存和恢复哪些数据，这要看看具体的View的onSaveInstanceState()和onRestoreInstanceState()。

- 既然在重建时onCreate()和onRestoreInstanceState()都能接受到参数，有什么区别？
> 区别是在onCreate(Bundle saveInstanceState)需要判断saveInstanceState是否为null，而onRestoreInstanceState(Bundle saveInstanceState)一旦被调用saveInstanceState就不为null,当然，官方推荐在onRestoreInstanceState()中恢复数据。
#### 二、Activity启动模式
#### 1、standard
>默认的标准模式，每次启动都会创建一个新的实例并置于栈顶，任务栈中可存在多个实例；新启动的Activity属于启动它的Activity所在的任务栈，这就意味着必须通过Activity的Context来启动standard模式的Activity，假设通过ApplicationContext来启动则会抛出异常，因为ApplicationContext并没有所属的任务栈，这个时候可以通过FLAG_ACTIVITY_NEW_TASK标记位来解决这个问题，但是这样实际就是以singleTask模式来启动了。

#### 2、singleTop
>栈顶服用模式，当栈顶的实例是要启动的Activity的实例时，不会重新初建，但是onNewIntent方法会被调用，可以通过这个方法的参数获得当前请求的信息。

#### 3、singleTask
>栈内复用模式，这是一种单实例模式，当请求启动一个该模式的Activity时，①如果所需要的任务栈不存在，创建任务栈，新建实例入栈；②如果任务在存在但无该Activity实例，新建实例入栈；③如果任务栈存在，实例也存在，clearTop,将该实例之上的实例全部出栈，让该实例暴露在栈顶。

#### 4、singleInstance
>单实例模式，是加强的singleTask,除了栈内复用，在启动singleInstance模式的Activity时会新创建一个任务栈，并且这个任务栈中只有该实例，后续都不会再创建该Activity的实例，除非这个任务栈由于某种原因被回收了。

-  以上几种模式，在发出新的启动请求时，如果不创建新的实例，都会调用onNewIntent方法，可以通过这个回调方法的参数来获取每次请求的参数。


### 三、IntentFilter的匹配规则
>隐式调用需要Intent匹配目标组件IntentFilter所设置的过滤信息，IntentFilter中包含的过滤信息有：action、category、data,Intent匹配过程需要同时匹配这三个过滤信息，否则就会匹配失败。这三种过滤信息的匹配规则是有差异的。

#### 1、action的匹配规则
>action是一个字符串，目标组件中的action可以有多个，隐式Intent中指定的action必须和目标组件的action列表中的其中一个完全相同（区分大小写），否则匹配失败，另外Intent中是必须指定action的。

#### 2、catetory匹配规则
>category是一个字符串，在写隐式Intent时可以不包含category,如果有一个或者多个，那么所有的都需要和目标组件中的category列表中的任意一个相同，否则匹配失败。在调用startActivity()或者startActivityForResult()时Intent会默认添加上android.intent.category.DEFAULT,这就是为什么在写隐式Inent时可以不包含category的原因，但是这也就意味着我们在定义IntentFilter时如果没有加上
```
<category android.name="android.intent.category.DEFAULT"/>
```
这个过滤信息，这个组件就不能被隐式Intent匹配。

#### 3、data匹配规则
>data的语法是相对复杂的。data的匹配规则和action的匹配规则类似，Intent中必须包含data,并且要和过滤器中的其中一个data相同。

excample:
```
activity android:name=".ShareActivity">
    <intent-filter>
        <action android:name="com.test.action1"/>
        <action android:name="com.test.action2"/>
        <category android:name="android.intent.category.DEFAULT"/>
        <category android:name="com.test.category1"/>
        <category android:name="com.test.category2"/>
        <data android:mimeType="text/plain"/>
        <data android:mimeType="image/*"/>
        <data android:mimeType="video/*"/>
    </intent-filter>
</activity>
```
可以匹配的两个隐式Intent
```
Intent intent = new Intent();
intent.setAction("com.test.action1");
intent.addCategory("com.test.category1");
intent.setType("text/plain");
startActivity(intent);
```

```
Intent intent = new Intent();
intent.setAction("com.test.action1");
intent.setType("text/plain");
intent.setType("image/*");
startActivity(intent);
```
上面这个Intent会默认添加
```
intent.addCategory("android.intent.category.DEFAULT");
```
若去掉IntentFilter中的
```<category android:name="android.intent.category.DEFAULT"/>```
任何隐式Intent都不能匹配这个组件，原因已在category匹配规则说明。


