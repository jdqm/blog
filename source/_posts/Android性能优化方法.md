---
title: Android性能优化方法
date: 2017-03-31 16:54:16
tags:
	- Android
	- 性能优化
categories: Android
---

前言

## Q：为什么要进行性能优化？
 Android作为移动平台，不管是内存或者cpu的性能都受到了一定的限制？过多的使用内存会导致OOM，过多的使用cpu资源，一般指做大量的耗时任务，将会是设备变得卡顿甚至出现ANR（应用程序无响应）异常。

**优化的方式**
<!-- more -->
## I.  布局优化
布局优化的思想：尽量减少布局的层级，减少绘制界面时的工作量。具体如何做：
**方式一：**
①去除一些无用的布局，View;
②有选择性的使用布局：比如能使用RelativeLayout也可以使用使用LinearLayout的地方使用后者，尽量少使用性能低的ViewGroup;
③减少层级优先级高于对ViewGroup性能的考虑。


**方式二：**使用```<include>```、```<merge>```标签和```<ViewStub>```，```<include>```标签用于布局重用，```<merge>```标签一般和```<include>```标签配合使用，它可以减少布局的层级，而```<ViewStub>```提供了按需加载的功能，当需要使用时再加载布局到内存，这可以提高应用程序的初始化效率。但ViewStub也不是万能的，下面总结下ViewStub能做的事儿和什么时候该用ViewStub，什么时候该用View可见性的控制。

首先来说说ViewStub的一些特点：

1、ViewStub只能Inflate一次，之后ViewStub对象会被置为空。按句话说，某个被ViewStub指定的布局被Inflate后，就不会够再通过ViewStub来控制它了。
2、ViewStub只能用来Inflate一个布局文件，而不是某个具体的View，当然也可以把View写在某个布局文件中。

基于以上的特点，那么可以考虑使用ViewStub的情况有：

1、在程序的运行期间，某个布局在Inflate后，就不会有变化，除非重新启动。

因为ViewStub只能Inflate一次，之后会被置空，所以无法指望后面接着使用ViewStub来控制布局。所以当需要在运行时不止一次的显示和隐藏某个布局，那么ViewStub是做不到的。这时就只能使用View的可见性来控制了。

2、想要控制显示与隐藏的是一个布局文件，而非某个View。
因为设置给ViewStub的只能是某个布局文件的Id，所以无法让它来控制某个View。
所以，如果想要控制某个View(如Button或TextView)的显示与隐藏，或者想要在运行时不断的显示与隐藏某个布局或View，只能使用View的可见性来控制。

## II.绘制优化
    绘制优化指的是避免在View的onDraw方法中执行大量的操作。主要体现在两个方面：
①在onDraw方法中不要创建新的对象，因为onDraw方法会频繁调用，这会产生 大量的临时对象，不仅占用内存高而且可能导致频繁的gc而影响程序的执行效率。
②避免在onDraw方法中执行耗时操作或者成千上万的循环操作，大量的循环会抢占cpu时间片，这将导致View的绘制出现卡顿现象，根据Google官方给出的性能优化典范中的标准，view的绘制帧率保证60fps是最佳，这就要求每帧的绘制时间为16ms(16=1000/60)，虽然程序很难做到这点，但是降低onDraw方法的复杂度确是切实有效的。

## III.内存泄漏优化
内存泄漏在开发中是需要重视的一个问题，但是内存泄漏问题对开发人员的经验和开发意识有较高的要求。内存泄漏优化主要分为两个方面：
①避免写出内存泄漏的代码；
②通过一些分析工具如MAT来找出内存泄漏的代码继而解决；

## IV.响应速度优化和ANR日志分析

响应速度优化的核心思想是避免在主线程中做耗时操作，但是有时候确实有很多耗时操作，可以考虑放到子线程中执行，就是采用异步的方式去处理耗时操作。响应速度过慢更多的体现在Activity的启动上，如果在主线程中做过多的操作，可能会导致Activity启动过程出现黑屏，甚至出现ANR异常。Android规定activity5秒内不能响应输入事件就会出现ANR，而BroadcastReceiver 10 秒内还没执行完操作也会出现ANR异常。ANR异常很难再代码中发现，如何定位？当出现ANR异常时，系统会在data/anr目录下创建一个trace.txt文件，通过分析这个文件来定位分析ANR的原因。

## V. ListView和Bitmap优化
 ListView主要分三个方面：
①采用ViewHolder并避免在getView方法中做耗时操作；
②要根据列表的滑动状态来控制任务的执行频率，比如当列表快速滑动是显然不适合开启大量的异步任务；
③尝试开启硬件加速来优化来是ListView的滑动更加顺畅。

## VI.线程优化
 线程优化的核心思想是使用线程池，避免程序中存在大量的线程，通过线程池可以重用线程，减少由于线程的创建销毁带来的系统消耗，同时线程池也可以有效地控制线程池中最大的并发数，避免大量线程因互相抢占系统资源而导致阻塞。

## VII. 一些性能优化建议
  ①避免过多的创建兑现；
  ②不要过多使用枚举，枚举占用的内存比整形大；
  ③常量请使用 static final 来修饰；
  ④使用一些Android特有的数据结构，如SparesArray和Pair等，他们都具有更好的性能；
  ⑤适当使用软引用和弱引用；
  ⑥采用内存缓存和磁盘缓存；
  ⑦尽量采用静态内部类，这样可以避免潜在的由于内部类引起的内存泄漏。


















