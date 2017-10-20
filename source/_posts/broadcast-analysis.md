---
title: Android广播工作过程分析
date: 2017-09-26 23:17:22
updated: 2017-10-20 23:17:22
tags: 
	-broadcast
categories: Android
---

[上一篇文章][1]已经介绍了广播的类型，如何注册广播，如何发送广播以及使用过程应该注意的一些点。本文将从源码的角度来分析注册、发送、执行广播的过程。
<!-- more -->

# 一.静态广播注册
静态注册指的是在AndroidManifest.xml中注册的广播，这些册信息的维护主要有两个过程：系统启动的时候扫描系统中安装的apk的注册信息，安装、卸载的时候更新注册信息。


当系统启动的时候会启动PackageManagerService，从其main方法开始。

```
public static PackageManagerService main(Context context, Installer installer,
        boolean factoryTest, boolean onlyCore) {
    // Self-check for initial settings.
    PackageManagerServiceCompilerMapping.checkProperties();
    
    //创建了一个PackageManagerService实例
    PackageManagerService m = new PackageManagerService(context, installer,
            factoryTest, onlyCore);
    m.enableSystemUserPackages();
    
    //注册这个Service
    ServiceManager.addService("package", m);
    return m;
}
```
它的main方法主要通过它的构造方法创建了一个实例，并注册了这个Service，它的构造方法实现如下：


```
public PackageManagerService(Context context, Installer installer,
        boolean factoryTest, boolean onlyCore) {

    ......
    synchronized (mPackages) {
        ......
        File dataDir = Environment.getDataDirectory();
        mAppInstallDir = new File(dataDir, "app");
        mAppLib32InstallDir = new File(dataDir, "app-lib");
        mEphemeralInstallDir = new File(dataDir, "app-ephemeral");
        mAsecInternalPath = new File(dataDir, "app-asec").getPath();
        mDrmAppPrivateInstallDir = new File(dataDir, "app-private");
        ......
        // Collect ordinary system packages.
        final File systemAppDir = new File(Environment.getRootDirectory(), "app");
        scanDirTracedLI(systemAppDir, mDefParseFlags
                | PackageParser.PARSE_IS_SYSTEM
                | PackageParser.PARSE_IS_SYSTEM_DIR, scanFlags, 0);
        //扫描其他路径            
}
```
构造方法中的代码比较多，这里只保留了部分关键代码，从代码中的扫描的路径可以了解到Android系统中的应用都安装在哪，这里只分析扫描系统应用的过程，即/system/app这个目录，其他路径类似。这个过程涉及到的方法比较多，先上一张时序序图：

![静态注册广播时序图](http://upload-images.jianshu.io/upload_images/3631399-459b001ca827c0b7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

构造方法之后，沿着路径往下到 scanPackageLI 方法才有我们比较关心的代码。
```
/**
 *  Scans a package and returns the newly parsed package.
 *  Returns {@code null} in case of errors and the error code is stored in mLastScanError
 */
private PackageParser.Package scanPackageLI(File scanFile, int parseFlags, int scanFlags,
        long currentTime, UserHandle user) throws PackageManagerException {
    if (DEBUG_INSTALL) Slog.d(TAG, "Parsing: " + scanFile);
    PackageParser pp = new PackageParser();
    pp.setSeparateProcesses(mSeparateProcesses);
    pp.setOnlyCoreApps(mOnlyCore);
    pp.setDisplayMetrics(mMetrics);

    if ((scanFlags & SCAN_TRUSTED_OVERLAY) != 0) {
        parseFlags |= PackageParser.PARSE_TRUSTED_OVERLAY;
    }

    Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "parsePackage");
    final PackageParser.Package pkg;
    try {
    
        //这里调用了PackageParser的parsePackage方法来解析apk文件
        pkg = pp.parsePackage(scanFile, parseFlags);
    } catch (PackageParserException e) {
        throw PackageManagerException.from(e);
    } finally {
        Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
    }
    
    //最后又调用一个重载方法
    return scanPackageLI(pkg, scanFile, parseFlags, scanFlags, currentTime, user);
}
```
这个方法里主要调用了两个方法：PackageParser的parsePackage方法和一个重载的scanPackageLI方法，按照顺序，先看parsePackage方法。

```
public Package parsePackage(File packageFile, int flags) throws PackageParserException {
    if (packageFile.isDirectory()) {
        return parseClusterPackage(packageFile, flags);
    } else {
        return parseMonolithicPackage(packageFile, flags);
    }
}
```
这个方法比较简单，区分文件和目录来调用不同的方法，这里研究parseClusterPackage这条路径（路径参考上面的时序图），沿着这条路径往下，直到parseBaseApplication方法，这里有我们所关心的代码。

```
/**
 * Parse the {@code application} XML tree at the current parse location in a
 * <em>base APK</em> manifest.
 * <p>
 * When adding new features, carefully consider if they should also be
 * supported by split APKs.
 */
private boolean parseBaseApplication(Package owner, Resources res,
        XmlResourceParser parser, int flags, String[] outError)
    throws XmlPullParserException, IOException {
    TypedArray sa = res.obtainAttributes(parser,
                com.android.internal.R.styleable.AndroidManifestApplication);
                
    //省略部分非关键代码
    ....
    while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
                && (type != XmlPullParser.END_TAG || parser.getDepth() > innerDepth)) {
            if (type == XmlPullParser.END_TAG || type == XmlPullParser.TEXT) {
                continue;
            }

            String tagName = parser.getName();
            if (tagName.equals("activity")) {
                Activity a = parseActivity(owner, res, parser, flags, outError, false,
                        owner.baseHardwareAccelerated);
                if (a == null) {
                    mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
                    return false;
                }

                owner.activities.add(a);
            
            //解析receiver节点并保存到owner.receivers
            } else if (tagName.equals("receiver")) {
                Activity a = parseActivity(owner, res, parser, flags, outError, true, false);
                if (a == null) {
                    mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
                    return false;
                }

                owner.receivers.add(a);
            }    
            
            //省略解析其他组件分支的代码
            ......
    }        
}        
```
可以看到，原来是解析各个apk文件的AndroidManifest.xml文件中所有注册的receiver，并且其添加到Package的成员变量receivers中，这样我们在PMS中拿到Package对象就能得到这个应用的所用静态注册的广播。接下来就回到了PMS中（通常跟踪源码到返回的时候意味着离目标也已经不远了）。


当返回到PMS后，又调用了一个重载的scanPackageLI方法，沿着路径往下，到scanPackageDirtyLI方法又出现关键代码。

```
private PackageParser.Package scanPackageDirtyLI(PackageParser.Package pkg,
            final int policyFlags, final int scanFlags, long currentTime, UserHandle user)
            throws PackageManagerException {
    ....        
    //省略其他非关键代码
    
    //这个pkg就是刚才PackageParser中解析得到的
    N = pkg.receivers.size();
    r = null;
    for (i=0; i<N; i++) {
        PackageParser.Activity a = pkg.receivers.get(i);
        a.info.processName = fixProcessName(pkg.applicationInfo.processName,
                a.info.processName, pkg.applicationInfo.uid);
        //将所有静态注册的广播添加到mReceivers集合中        
        mReceivers.addActivity(a, "receiver");
        if ((policyFlags&PackageParser.PARSE_CHATTY) != 0) {
            if (r == null) {
                r = new StringBuilder(256);
            } else {
                r.append(' ');
            }
            r.append(a.info.name);
        }
    }
    .....
}        
```
看到这里终于松了一口气，原来所有的静态注册的广播最终是保存在PMS的成员变量mReceivers中，当发送广播时，AMS就会到这里来查询所有匹配的静态广播接收者，进而将广播发送给它们。下面的就是AMS中查询静态广播接受者的代码。


# 二.动态注册广播
动态注册广播我们通常需要这么做
```
IntentFilter filter = new IntentFilter("jdqm.intent.action.TEST");
registerReceiver(receiver, filter);
```
从调用context.registerReceiver方法到注册完成的大致的过程为：

![动态注册广播时序图](http://upload-images.jianshu.io/upload_images/3631399-fa0a479ed313f299.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从ContextWapper的registerReceiver方法开始

```
@Override
public Intent registerReceiver(
    BroadcastReceiver receiver, IntentFilter filter) {
    return mBase.registerReceiver(receiver, filter);
}
```
直接调用了mBase的registerReceiver方法，这个mBase实际上是ContextImp的实例，ContextImp的registerReceiver方法也是将处理逻辑放到了该类的registerReceiverInternal方法中，该方法的实现如下：

```
private Intent registerReceiverInternal(BroadcastReceiver receiver, int userId,
        IntentFilter filter, String broadcastPermission,
        Handler scheduler, Context context) {
    //使用IItentReceiver（这是一个Binder对象）来中转实现跨进程通信    
    IIntentReceiver rd = null;
    if (receiver != null) {
    
        //mPackageInfo是LoadedApk的实例
        if (mPackageInfo != null && context != null) {
            if (scheduler == null) {
                //这个Handler是主线程中mH
                scheduler = mMainThread.getHandler();
            }
            
            //通过getReceiverDispatcher获取对应的IItentReceiver对象
            rd = mPackageInfo.getReceiverDispatcher(
                receiver, context, scheduler,
                mMainThread.getInstrumentation(), true);
        } else {
            if (scheduler == null) {
                scheduler = mMainThread.getHandler();
            }
            rd = new LoadedApk.ReceiverDispatcher(
                    receiver, context, scheduler, null, true).getIIntentReceiver();
        }
    }
    try {
        //通过Binder机制，向AMS发起注册请求，后续的逻辑就到AMS中了
        final Intent intent = ActivityManagerNative.getDefault().registerReceiver(
                mMainThread.getApplicationThread(), mBasePackageName,
                rd, filter, broadcastPermission, userId);
        if (intent != null) {
            intent.setExtrasClassLoader(getClassLoader());
            intent.prepareToEnterProcess();
        }
        return intent;
    } catch (RemoteException e) {
        throw e.rethrowFromSystemServer();
    }
}
```

在这个方法中通过调用getReceiverDispatcher方法，将广播记录在LoadedApk的成员变量mReceivers中。另外 ActivityManagerNative.getDefault()得到的是ActivityManagerService在客户端的一个代理对象，通过它我们就可以调用ActivityManagerService的registerReceiver方法：

```
public Intent registerReceiver(IApplicationThread caller, String callerPackage,
        IIntentReceiver receiver, IntentFilter filter, String permission, int userId) {
    ......
    //省略前面代码
    synchronized (this) {
        if (callerApp != null && (callerApp.thread == null
                || callerApp.thread.asBinder() != caller.asBinder())) {
            // Original caller already died
            return null;
        }
        ReceiverList rl = mRegisteredReceivers.get(receiver.asBinder());
        if (rl == null) {
            rl = new ReceiverList(this, callerApp, callingPid, callingUid,
                    userId, receiver);
            if (rl.app != null) {
                rl.app.receivers.add(rl);
            } else {
                try {
                    receiver.asBinder().linkToDeath(rl, 0);
                } catch (RemoteException e) {
                    return sticky;
                }
                rl.linkedToDeath = true;
            }
            
            //动态注册的广播都保存在这个集合中
            mRegisteredReceivers.put(receiver.asBinder(), rl);
        } else if (rl.uid != callingUid) {
            throw new IllegalArgumentException(
                    "Receiver requested to register for uid " + callingUid
                    + " was previously registered for uid " + rl.uid);
        } else if (rl.pid != callingPid) {
            throw new IllegalArgumentException(
                    "Receiver requested to register for pid " + callingPid
                    + " was previously registered for pid " + rl.pid);
        } else if (rl.userId != userId) {
            throw new IllegalArgumentException(
                    "Receiver requested to register for user " + userId
                    + " was previously registered for user " + rl.userId);
        }
        BroadcastFilter bf = new BroadcastFilter(filter, rl, callerPackage,
                permission, callingUid, userId);
                
        //关联IntentFilter        
        rl.add(bf);
        if (!bf.debugCheck()) {
            Slog.w(TAG, "==> For Dynamic broadcast");
        }
        mReceiverResolver.addFilter(bf);
        //省略后面代码
        ......
    }
}
```

到这里我们得出结论：动态注册的广播存储在AMS的成员变量mRegisteredReceivers中。


# 三.发送广播
发送广播过程的时序图如下：

![发送广播时序图](http://upload-images.jianshu.io/upload_images/3631399-3ac33d85dfefa157.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

首先从ContextWrapper#sendBroadcast(Intent)方法开始.

```
@Override
public void sendBroadcast(Intent intent) {
    //ContextImp: mBase
    mBase.sendBroadcast(intent);
}
```
ContextImp#sendBroadcast(Intent)

```
@Override
public void sendBroadcast(Intent intent) {
    warnIfCallingFromSystemProcess();
    //这个resolvedType实际上是MIME类型
    String resolvedType = intent.resolveTypeIfNeeded(getContentResolver());
    try {
        intent.prepareToLeaveProcess(this);
        
        //通过Binder机制，向AMS发起请求
        ActivityManagerNative.getDefault().broadcastIntent(
                mMainThread.getApplicationThread(), intent, resolvedType, null,
                Activity.RESULT_OK, null, null, null, AppOpsManager.OP_NONE, null, false, false,
                getUserId());
    } catch (RemoteException e) {
        throw e.rethrowFromSystemServer();
    }
}
```
前面提到过ActivityManagerNative.getDefault()返回的是AMS在客户端的代理对象，通过它来调用AMS#broadcastIntent方法。

```
public final int broadcastIntent(IApplicationThread caller,
        Intent intent, String resolvedType, IIntentReceiver resultTo,
        int resultCode, String resultData, Bundle resultExtras,
        String[] requiredPermissions, int appOp, Bundle bOptions,
        boolean serialized, boolean sticky, int userId) {
    enforceNotIsolatedCaller("broadcastIntent");
    synchronized(this) {
        intent = verifyBroadcastLocked(intent);
        final ProcessRecord callerApp = getRecordForAppLocked(caller);
        final int callingPid = Binder.getCallingPid();
        final int callingUid = Binder.getCallingUid();
        final long origId = Binder.clearCallingIdentity();
        
        //进一步调用broadcastIntentLocked来处理
        int res = broadcastIntentLocked(callerApp,
                callerApp != null ? callerApp.info.packageName : null,
                intent, resolvedType, resultTo, resultCode, resultData, resultExtras,
                requiredPermissions, appOp, bOptions, serialized, sticky,
                callingPid, callingUid, userId);
        Binder.restoreCallingIdentity(origId);
        return res;
    }
}
```
可以看到在这个方法内部又进一步调用broadcastIntentLocked来处理，这个方法代码很多，下面只列出部分：

```
//Android3.1之后，默认情况下，Intent会添加了下面这个flag,所以默认情况下不会启动处于停止状态的应用。
// By default broadcasts do not go to stopped apps.
intent.addFlags(Intent.FLAG_EXCLUDE_STOPPED_PACKAGES);
......

// Figure out who all will receive this broadcast.
List receivers = null;
List<BroadcastFilter> registeredReceivers = null;
// Need to resolve the intent to interested receivers...
if ((intent.getFlags()&Intent.FLAG_RECEIVER_REGISTERED_ONLY)
         == 0) {
    //收集匹配静态注册的广播接收者    
    receivers = collectReceiverComponents(intent, resolvedType, callingUid, users);
}
if (intent.getComponent() == null) {
    if (userId == UserHandle.USER_ALL && callingUid == Process.SHELL_UID) {
        // Query one target user at a time, excluding shell-restricted users
        for (int i = 0; i < users.length; i++) {
            if (mUserController.hasUserRestriction(
                    UserManager.DISALLOW_DEBUGGING_FEATURES, users[i])) {
                continue;
            }
            List<BroadcastFilter> registeredReceiversForUser =
                    mReceiverReHHsolver.queryIntent(intent,
                            resolvedType, false, users[i]);
            if (registeredReceivers == null) {
                registeredReceivers = registeredReceiversForUser;
            } else if (registeredReceiversForUser != null) {
                registeredReceivers.addAll(registeredReceiversForUser);
            }
        }
    } else {
        //收集匹配动态注册的广播接受者
        registeredReceivers = mReceiverResolver.queryIntent(intent,
                resolvedType, false, userId);
    }
}

......
//动态注册的无序广播
BroadcastRecord r = new BroadcastRecord(queue, intent, callerApp,
        callerPackage, callingPid, callingUid, resolvedType, requiredPermissions,
        appOp, brOptions, registeredReceivers, resultTo, resultCode, resultData,
        resultExtras, ordered, sticky, false, userId);
if (DEBUG_BROADCAST) Slog.v(TAG_BROADCAST, "Enqueueing parallel broadcast " + r);
final boolean replaced = replacePending && queue.replaceParallelBroadcastLocked(r);
if (!replaced) {
   //将动态注册的无序广播添加的无序广播队列
    queue.enqueueParallelBroadcastLocked(r);
    queue.scheduleBroadcastsLocked();
}
....

//receivers保存的是所有的静态广播，动态注册的有序广播
if ((receivers != null && receivers.size() > 0)
        || resultTo != null) {
    BroadcastQueue queue = broadcastQueueForIntent(intent);
    BroadcastRecord r = new BroadcastRecord(queue, intent, callerApp,
            callerPackage, callingPid, callingUid, resolvedType,
            requiredPermissions, appOp, brOptions, receivers, resultTo, resultCode,
            resultData, resultExtras, ordered, sticky, false, userId);

    if (DEBUG_BROADCAST) Slog.v(TAG_BROADCAST, "Enqueueing ordered broadcast " + r
            + ": prev had " + queue.mOrderedBroadcasts.size());
    if (DEBUG_BROADCAST) Slog.i(TAG_BROADCAST,
            "Enqueueing broadcast " + r.intent.getAction());

    boolean replaced = replacePending && queue.replaceOrderedBroadcastLocked(r);
    if (!replaced) {
        //将构造的BroadcastRecord添加的有序广播的队列
        queue.enqueueOrderedBroadcastLocked(r);
        queue.scheduleBroadcastsLocked();
    }
} else {
    // There was nobody interested in the broadcast, but we still want to record
    // that it happened.
    if (intent.getComponent() == null && intent.getPackage() == null
            && (intent.getFlags()&Intent.FLAG_RECEIVER_REGISTERED_ONLY) == 0) {
        // This was an implicit broadcast... let's record it for posterity.
        addBroadcastStatLocked(intent.getAction(), callerPackage, 0, 0, 0);
    }
}
```
从这方法的实现可以看到，首先收集了所有的符合的动态注册、静态注册的接收者，然后将动态无序广播、所有静态和动态有序包装成两个BroadcastRecord对象，添加到BroadcastQueue的两个集合中，这两个集合是：

```
final ArrayList<BroadcastRecord> mParallelBroadcasts = new ArrayList<>();
final ArrayList<BroadcastRecord> mOrderedBroadcasts = new ArrayList<>();
```
然后调用用BroadcastQueue的scheduleBroadcastsLocked方法进处理这两个集合。scheduleBroadcastsLocked方法的实现如下：

```
public void scheduleBroadcastsLocked() {
    if (DEBUG_BROADCAST) Slog.v(TAG_BROADCAST, "Schedule broadcasts ["
            + mQueueName + "]: current="
            + mBroadcastsScheduled);

    if (mBroadcastsScheduled) {
        return;
    }
    mHandler.sendMessage(mHandler.obtainMessage(BROADCAST_INTENT_MSG, this));
    mBroadcastsScheduled = true;
}
```
这里逻辑也很简单，通过mHandler发送了一条BROADCAST_INTENT_MSG消息，这个Handler的实现如下：

```
final BroadcastHandler mHandler;

private final class BroadcastHandler extends Handler {
    public BroadcastHandler(Looper looper) {
        super(looper, null, true);
    }

    @Override
    public void handleMessage(Message msg) {
        switch (msg.what) {
            case BROADCAST_INTENT_MSG: {
                if (DEBUG_BROADCAST) Slog.v(
                        TAG_BROADCAST, "Received BROADCAST_INTENT_MSG");
                //关键代码
                processNextBroadcast(true);
            } break;
            case BROADCAST_TIMEOUT_MSG: {
                synchronized (mService) {
                    broadcastTimeoutLocked(true);
                }
            } break;
            case SCHEDULE_TEMP_WHITELIST_MSG: {
                DeviceIdleController.LocalService dic = mService.mLocalDeviceIdleController;
                if (dic != null) {
                    dic.addPowerSaveTempWhitelistAppDirect(UserHandle.getAppId(msg.arg1),
                            msg.arg2, true, (String)msg.obj);
                }
            } break;
        }
    }
}

```
可以发现，调用了processNextBroadcast方法来处理：

```
final void processNextBroadcast(boolean fromMsg) {
    synchronized(mService) {
        BroadcastRecord r;
        mService.updateCpuStats();
        if (fromMsg) {
            mBroadcastsScheduled = false;
        }
        
        //首先处理mParallelBroadcasts这个集合，这意味着mParallelBroadcasts集合的广播接受者比mOrderedBroadcasts集合的广播接收者先收到广播
        // First, deliver any non-serialized broadcasts right away.
        while (mParallelBroadcasts.size() > 0) {
        
           //一个广播对应着一个BroadcastRecord实例
            r = mParallelBroadcasts.remove(0);
            r.dispatchTime = SystemClock.uptimeMillis();
            r.dispatchClockTime = System.currentTimeMillis();
            final int N = r.receivers.size();
            if (DEBUG_BROADCAST_LIGHT) Slog.v(TAG_BROADCAST, "Processing parallel broadcast ["
                    + mQueueName + "] " + r);
            for (int i=0; i<N; i++) {
                Object target = r.receivers.get(i);
                if (DEBUG_BROADCAST)  Slog.v(TAG_BROADCAST,
                        "Delivering non-ordered on [" + mQueueName + "] to registered "
                        + target + ": " + r);
                //动态注册的无序广播        
                deliverToRegisteredReceiverLocked(r, (BroadcastFilter)target, false, i);
            }
            
            //记录广播的历史
            addBroadcastToHistoryLocked(r);
            if (DEBUG_BROADCAST_LIGHT) Slog.v(TAG_BROADCAST, "Done with parallel broadcast ["
                    + mQueueName + "] " + r);
        }

        // Now take care of the next serialized one...

        // If we are waiting for a process to come up to handle the next
        // broadcast, then do nothing at this point.  Just in case, we
        // check that the process we're waiting for still exists.
        if (mPendingBroadcast != null) {
            if (DEBUG_BROADCAST_LIGHT) Slog.v(TAG_BROADCAST,
                    "processNextBroadcast [" + mQueueName + "]: waiting for "
                    + mPendingBroadcast.curApp);

            boolean isDead;
            synchronized (mService.mPidsSelfLocked) {
                ProcessRecord proc = mService.mPidsSelfLocked.get(mPendingBroadcast.curApp.pid);
                isDead = proc == null || proc.crashing;
            }
            if (!isDead) {
                // It's still alive, so keep waiting
                return;
            } else {
                Slog.w(TAG, "pending app  ["
                        + mQueueName + "]" + mPendingBroadcast.curApp
                        + " died before responding to broadcast");
                mPendingBroadcast.state = BroadcastRecord.IDLE;
                mPendingBroadcast.nextReceiver = mPendingBroadcastRecvIndex;
                mPendingBroadcast = null;
            }
        }

        boolean looped = false;
        
        do {
            if (mOrderedBroadcasts.size() == 0) {
            ......   
        } while (r == null);
        
        .....
        
        // Is this receiver's application already running?
        if (app != null && app.thread != null) {
            try {
                app.addPackage(info.activityInfo.packageName,
                        info.activityInfo.applicationInfo.versionCode, mService.mProcessStats);
                //当进程跑起来后        
                processCurBroadcastLocked(r, app);
                return;
            } catch (RemoteException e) {
                Slog.w(TAG, "Exception when sending broadcast to "
                      + r.curComponent, e);
            } catch (RuntimeException e) {
                Slog.wtf(TAG, "Failed sending broadcast to "
                        + r.curComponent + " with " + r.intent, e);
                // If some unexpected exception happened, just skip
                // this broadcast.  At this point we are not in the call
                // from a client, so throwing an exception out from here
                // will crash the entire system instead of just whoever
                // sent the broadcast.
                logBroadcastReceiverDiscardLocked(r);
                finishReceiverLocked(r, r.resultCode, r.resultData,
                        r.resultExtras, r.resultAbort, false);
                scheduleBroadcastsLocked();
                // We need to reset the state if we failed to start the receiver.
                r.state = BroadcastRecord.IDLE;
                return;
            }

            // If a dead object exception was thrown -- fall through to
            // restart the application.
        }
}
```
可以发现，先处理了mParallelBroadcasts，然后才处理mOrderedBroadcasts，这两个集合的处理逻辑是有些差异的，按顺序先看mParallelBroadcasts，这个集合是通过deliverToRegisteredReceiverLocked来处理，而这个方法又进一步调用了performReceiveLocked方法：

```
void performReceiveLocked(ProcessRecord app, IIntentReceiver receiver,
        Intent intent, int resultCode, String data, Bundle extras,
        boolean ordered, boolean sticky, int sendingUser) throws RemoteException {
    // Send the intent to the receiver asynchronously using one-way binder calls.
    if (app != null) {
        if (app.thread != null) {
            // If we have an app thread, do the call through that so it is
            // correctly ordered with other one-way calls.
            try {
                app.thread.scheduleRegisteredReceiver(receiver, intent, resultCode,
                        data, extras, ordered, sticky, sendingUser, app.repProcState);
            // TODO: Uncomment this when (b/28322359) is fixed and we aren't getting
            // DeadObjectException when the process isn't actually dead.
            //} catch (DeadObjectException ex) {
            // Failed to call into the process.  It's dying so just let it die and move on.
            //    throw ex;
            } catch (RemoteException ex) {
                // Failed to call into the process. It's either dying or wedged. Kill it gently.
                synchronized (mService) {
                    Slog.w(TAG, "Can't deliver broadcast to " + app.processName
                            + " (pid " + app.pid + "). Crashing it.");
                    app.scheduleCrash("can't deliver broadcast");
                }
                throw ex;
            }
        } else {
            // Application has died. Receiver doesn't exist.
            throw new RemoteException("app.thread must not be null");
        }
    } else {
        receiver.performReceive(intent, resultCode, data, extras, ordered,
                sticky, sendingUser);
    }
}
```
app.thread就是ActivityThread的一个代理对象，通过它就可以调用ActivityThread的scheduleRegisteredReceiver方法：

```
// This function exists to make sure all receiver dispatching is
// correctly ordered, since these are one-way calls and the binder driver
// applies transaction ordering per object for such calls.
public void scheduleRegisteredReceiver(IIntentReceiver receiver, Intent intent,
        int resultCode, String dataStr, Bundle extras, boolean ordered,
        boolean sticky, int sendingUser, int processState) throws RemoteException {
    updateProcessState(processState, false);
    receiver.performReceive(intent, resultCode, dataStr, extras, ordered,
            sticky, sendingUser);
}
```
继续调用IIntentReceiver.performReceive方法来完成后续动作，又将逻辑切回到了我们注册广播时的LoadedApk.ReceiverDispatcher的performReceive中，这就回到了注册广播的主线程了。

```
public void performReceive(Intent intent, int resultCode, String data,
        Bundle extras, boolean ordered, boolean sticky, int sendingUser) {
    //创建了一个Args对象，Args实现了Runable接口    
    final Args args = new Args(intent, resultCode, data, extras, ordered,
            sticky, sendingUser);
    if (intent == null) {
        Log.wtf(TAG, "Null intent received");
    } else {
        if (ActivityThread.DEBUG_BROADCAST) {
            int seq = intent.getIntExtra("seq", -1);
            Slog.i(ActivityThread.TAG, "Enqueueing broadcast " + intent.getAction()
                    + " seq=" + seq + " to " + mReceiver);
        }
    }
    
    //mActivityThread是 ActivityThread#mH
    if (intent == null || !mActivityThread.post(args)) {
        if (mRegistered && ordered) {
            IActivityManager mgr = ActivityManagerNative.getDefault();
            if (ActivityThread.DEBUG_BROADCAST) Slog.i(ActivityThread.TAG,
                    "Finishing sync broadcast to " + mReceiver);
            args.sendFinished(mgr);
        }
    }
}
```
创建了一个Args对象，Args实现了Runnable接口，然后mActivityThread.post(args)将这个Runnable 抛给了mH，Args的run方法实现如下：

```
public void run() {
    ......
    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "broadcastReceiveReg");
    try {
        ClassLoader cl =  mReceiver.getClass().getClassLoader();
        intent.setExtrasClassLoader(cl);
        intent.prepareToEnterProcess();
        setExtrasClassLoader(cl);
        receiver.setPendingResult(this);
        
        //直接通过注册时的引用，调用其onReceive方法
        receiver.onReceive(mContext, intent);
    } catch (Exception e) {
        if (mRegistered && ordered) {
            if (ActivityThread.DEBUG_BROADCAST) Slog.i(ActivityThread.TAG,
                    "Finishing failed broadcast to " + mReceiver);
            sendFinished(mgr);
        }
        if (mInstrumentation == null ||
                !mInstrumentation.onException(mReceiver, e)) {
            Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
            throw new RuntimeException(
                "Error receiving broadcast " + intent
                + " in " + mReceiver, e);
        }
    }
    
    if (receiver.getPendingResult() != null) {
        finish();
    }
    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
}
```
到这里终于看到了广播接受者的onReceive(mContext, intent)方法被执行了。mParallelBroadcasts的处理逻辑分析完了，接下来分析mOrderedBroadcasts的处理逻辑，从前面广播注册的逻辑得知，mOrderedBroadcasts包含所有静态注册的广播和动态注册的有序广播，其中动态注册的有序广播的处理逻辑与前面无序的处理过程类似。现在重点看下静态注册的广播，从processNextBroadcast方法中可以看到是调用了processCurBroadcastLocked来处理mOrderedBroadcasts中静态注册的广播。

```
private final void processCurBroadcastLocked(BroadcastRecord r,
        ProcessRecord app) throws RemoteException {

    r.receiver = app.thread.asBinder();
    r.curApp = app;
    app.curReceiver = r;
    app.forceProcessStateUpTo(ActivityManager.PROCESS_STATE_RECEIVER);
    mService.updateLruProcessLocked(app, false, null);
    mService.updateOomAdjLocked();

    // Tell the application to launch this receiver.
    r.intent.setComponent(r.curComponent);

    boolean started = false;
    try {
        if (DEBUG_BROADCAST_LIGHT) Slog.v(TAG_BROADCAST,
                "Delivering to component " + r.curComponent
                + ": " + r);
        mService.notifyPackageUse(r.intent.getComponent().getPackageName(),
                                  PackageManager.NOTIFY_PACKAGE_USE_BROADCAST_RECEIVER);
        //关键代码
        app.thread.scheduleReceiver(new Intent(r.intent), r.curReceiver,
                mService.compatibilityInfoForPackageLocked(r.curReceiver.applicationInfo),
                r.resultCode, r.resultData, r.resultExtras, r.ordered, r.userId,
                app.repProcState);
        if (DEBUG_BROADCAST)  Slog.v(TAG_BROADCAST,
                "Process cur broadcast " + r + " DELIVERED for app " + app);
        started = true;
    } finally {
        if (!started) {
            if (DEBUG_BROADCAST)  Slog.v(TAG_BROADCAST,
                    "Process cur broadcast " + r + ": NOT STARTED!");
            r.receiver = null;
            r.curApp = null;
            app.curReceiver = null;
        }
    }
}
```
通过ActivityThread的代理对象app.thread调用其scheduleReceiver方法，其实现如下：

```
public final void scheduleReceiver(Intent intent, ActivityInfo info,
        CompatibilityInfo compatInfo, int resultCode, String data, Bundle extras,
        boolean sync, int sendingUser, int processState) {
    updateProcessState(processState, false);
    ReceiverData r = new ReceiverData(intent, resultCode, data, extras,
            sync, false, mAppThread.asBinder(), sendingUser);
    r.info = info;
    r.compatInfo = compatInfo;
    sendMessage(H.RECEIVER, r);
}

private void sendMessage(int what, Object obj) {
        sendMessage(what, obj, 0, 0, false);
}

private void sendMessage(int what, Object obj, int arg1, int arg2, boolean async) {
    if (DEBUG_MESSAGES) Slog.v(
        TAG, "SCHEDULE " + what + " " + mH.codeToString(what)
        + ": " + arg1 + " / " + obj);
    Message msg = Message.obtain();
    msg.what = what;
    msg.obj = obj;
    msg.arg1 = arg1;
    msg.arg2 = arg2;
    if (async) {
        msg.setAsynchronous(true);
    }
    mH.sendMessage(msg);
}

```
最终通过mH发送了一条H.RECEIVER消息，对应what为RECEIVER的处理：

```
case RECEIVER:
    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "broadcastReceiveComp");
    handleReceiver((ReceiverData)msg.obj);
    maybeSnapshot();
    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
    break;
                    
```
紧接着调用handleReceiver来处理：
```
private void handleReceiver(ReceiverData data) {
    // If we are getting ready to gc after going to the background, well
    // we are back active so skip it.
    unscheduleGcIdler();

    String component = data.intent.getComponent().getClassName();

    LoadedApk packageInfo = getPackageInfoNoCheck(
            data.info.applicationInfo, data.compatInfo);

    IActivityManager mgr = ActivityManagerNative.getDefault();

    BroadcastReceiver receiver;
    try {
        java.lang.ClassLoader cl = packageInfo.getClassLoader();
        data.intent.setExtrasClassLoader(cl);
        data.intent.prepareToEnterProcess();
        data.setExtrasClassLoader(cl);
        
        //创建了一个广播接收者实例
        receiver = (BroadcastReceiver)cl.loadClass(component).newInstance();
    } catch (Exception e) {
        if (DEBUG_BROADCAST) Slog.i(TAG,
                "Finishing failed broadcast to " + data.intent.getComponent());
        data.sendFinished(mgr);
        throw new RuntimeException(
            "Unable to instantiate receiver " + component
            + ": " + e.toString(), e);
    }

    try {
        Application app = packageInfo.makeApplication(false, mInstrumentation);

        if (localLOGV) Slog.v(
            TAG, "Performing receive of " + data.intent
            + ": app=" + app
            + ", appName=" + app.getPackageName()
            + ", pkg=" + packageInfo.getPackageName()
            + ", comp=" + data.intent.getComponent().toShortString()
            + ", dir=" + packageInfo.getAppDir());

        ContextImpl context = (ContextImpl)app.getBaseContext();
        sCurrentBroadcastIntent.set(data.intent);
        receiver.setPendingResult(data);
        
        //广播接收者的onReceive方法被执行了
        receiver.onReceive(context.getReceiverRestrictedContext(),
                data.intent);
    } catch (Exception e) {
        if (DEBUG_BROADCAST) Slog.i(TAG,
                "Finishing failed broadcast to " + data.intent.getComponent());
        data.sendFinished(mgr);
        if (!mInstrumentation.onException(receiver, e)) {
            throw new RuntimeException(
                "Unable to start receiver " + component
                + ": " + e.toString(), e);
        }
    } finally {
        sCurrentBroadcastIntent.set(null);
    }

    if (receiver.getPendingResult() != null) {
        data.finish();
    }
}
```
静态注册的广播接收者通常是没有运行的，所以在这里先是创建了一个接收者的实例，然后调用它的onReceive方法。


# 三.总结：
1. 静态注册注册的广播，在PMS启动的时候扫描系统中安装的apk文件，并解析它们的AndroidManifest.xml文件，将所有注册的广播保存在PMS的成员变量mReceivers中；

2. 动态注册的广播存储在AMS的成员变量mRegisteredReceivers中；
3. 发送广播的处理逻辑在AMS中，AMS负责从mReceivers和mRegisteredReceivers这两个集合查询出与IntentFilter匹配的接收者，并将它们添加到BroadcastQueue的mParallelBroadcasts（动态注册的无序广播）或者mOrderedBroadcasts（有序广播和所有的静态广播）中;
4. 在BroadcastQueue先处理mParallelBroadcasts，所以动态注册的无序广播会先收到广播;
6. 当有应用安装，卸载时（实际的逻辑在PMS中），静态注册的广播自然是在PMS中进行更新注册信息，接着再发送广播到AMS中更新动态注册广播；
5. 动态广播的onReceive方法是在LoadedApk#ReceiverDispatcher#Args的run方法中被调用；静态广播的onReceive是在ActivityThread的handleReceiver方法中被调用，而它们都是跑在目标进程的主线程中。

以上源码均来自 Android25。

  [1]: http://www.jianshu.com/p/626245eb80a0
