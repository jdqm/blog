---
title: Android Studio 里的Cannot resolve symbol XXX
date: 2018-05-06 18:21:39
updated: 2018-05-06 18:21:39
tags:
    - error
    - Android Studio
    - 解决方法
categories: Android
---
## I.背景
事情是这样，一个同事更新替换了一个第三方合作商的aar包，并且push到了服务器，我pull下来后可以正常编译打包出apk，似乎一切都是正常的，但是当我打开用到这个aar包里的某些类的类时发现有些地方红色的，鼠标放上去提示：“Cannot resolve symbol XXX”。
<!-- more -->
## II.现象
Android Studio 无法识别同一个 package 里的其他类，或者无法识别某些方法，将其显示为红色，但是 compile 没有问题。鼠标放上去后显示 “Cannot resolve symbol XXX”，重启 Android Studio，重新 sync ，Clean build 都没有用。

## III.原因
1. 一个原因可能是因为 Android Studio 之前发生了错误，某些 setting 出了问题；
2. 替换的aar包没有更新版本号，导致AS识别到的是本地缓存的版本（我遇到的就是这个场景），但是 compile 的时候使用的是正确的版本，所以就出现了有些类或者方法提示无法识别但还是可以正常编译打包。
> 打开这个类提示 .class 文件与 .java 文件不匹配。

## IV.解决方法
这看起来是缓存引起的错误，那就清空缓存试试呗。

1. 点击菜单中的 “File” -> “Invalidate Caches / Restart”，然后点击对话框中的 “Invalidate and Restart”，清空 cache 并且重启。等待重启后基本上刚才的错误就没有了。

2. 如果还是没有解决，那可以尝试将依赖先去掉，sync gradle，接着再将依赖添加回来，sycn gradle，这个时候多半就正常了，我的环境就需要第二步才正常。

## V.本地缓存
我们知道，当通过gradle构建项目时，会将一些远程依赖缓存到本地，默认的缓存目录如下：
1. Mac OS: /Users/username/.gradle/caches
2. Windows: C:\Users\username\.gradle\caches

如果你不想用上面的解决方法，也可以尝试直接到文件系统删除相应的缓存文件，只不过这种方式我没有实践过。