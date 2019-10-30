[TOC]

# 应用主线程ActivityThread

在关于[Android进程](Android进程学习资料汇总.md)的一文中，不管是系统进程还是应用进程，最终都会执行`ActivityThread`相关的方法。

## 类定义

```java
/**
 * This manages the execution of the main thread in an
 * application process, scheduling and executing activities,
 * broadcasts, and other operations on it as the activity
 * manager requests.
 *
 * {@hide}
 */
public final class ActivityThread extends ClientTransactionHandler {
}
```

可见该类并不是`Thread`的子类，那为什么会命名为`ActivityThread`呢？

## 入口函数main()

在应用进程的创建中，最终会通过`RuntimeInit.applicationInit()`函数中反射调用`ActivityThread.main()`函数：

```java
public static void main(String[] args) {
    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ActivityThreadMain");

    // CloseGuard defaults to true and can be quite spammy.  We
    // disable it here, but selectively enable it later (via
    // StrictMode) on debug builds, but using DropBox, not logs.
    CloseGuard.setEnabled(false);

    Environment.initForCurrentUser();

    // Set the reporter for event logging in libcore
    EventLogger.setReporter(new EventLoggingReporter());

    // Make sure TrustedCertificateStore looks in the right place for CA certificates
    final File configDir = Environment.getUserConfigDirectory(UserHandle.myUserId());
    TrustedCertificateStore.setDefaultUserDirectory(configDir);

    Process.setArgV0("<pre-initialized>");
    // 创建MainLooper
    Looper.prepareMainLooper();

    // Find the value for {@link #PROC_START_SEQ_IDENT} if provided on the command line.
    // It will be in the format "seq=114"
    long startSeq = 0;
    if (args != null) {
        for (int i = args.length - 1; i >= 0; --i) {
            if (args[i] != null && args[i].startsWith(PROC_START_SEQ_IDENT)) {
                startSeq = Long.parseLong(
                        args[i].substring(PROC_START_SEQ_IDENT.length()));
            }
        }
    }
    // 创建对象，并执行attch函数
    ActivityThread thread = new ActivityThread();
    thread.attach(false, startSeq);

    // 给sMainThreadHandler赋值
    if (sMainThreadHandler == null) {
        sMainThreadHandler = thread.getHandler();
    }

    if (false) {
        Looper.myLooper().setMessageLogging(new
                LogPrinter(Log.DEBUG, "ActivityThread"));
    }

    // End of event ActivityThreadMain.
    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
    // 开启消息队列轮询
    Looper.loop();

    throw new RuntimeException("Main thread loop unexpectedly exited");
}
```

关键内容：

1. 先调用`Looper.prepareMainLooper()`创建主线程`looper`
2. 创建`ActivityThread`对象，并调用`attach()`
3. 调用`thread.getHandler()`给`sMainThreadHandler`赋值
4. 调用`Looper.loop()`，当前线程开启消息队列轮询

由第4点可知，线程将不会结束，也就意味着进程也将持续运行；虽然该类没有继承`Thread`，但实现却是一个一直轮询的单线程消息队列模型。

## 与ActivityManagerService进行attach

```java
private void attach(boolean system, long startSeq) {
    sCurrentActivityThread = this;
    mSystemThread = system;
    if (!system) {
        ViewRootImpl.addFirstDrawHandler(new Runnable() {
            @Override
            public void run() {
                ensureJitEnabled();
            }
        });
        android.ddm.DdmHandleAppName.setAppName("<pre-initialized>",
                                                UserHandle.myUserId());
        RuntimeInit.setApplicationObject(mAppThread.asBinder());
        final IActivityManager mgr = ActivityManager.getService();
        try {
            // mgr就是AMS在当前进程的代理对象，调用attachApplication进行关联
            mgr.attachApplication(mAppThread, startSeq);
        } catch (RemoteException ex) {
            throw ex.rethrowFromSystemServer();
        }
        // Watch for getting close to heap limit.
        BinderInternal.addGcWatcher(new Runnable() {
            @Override public void run() {
                if (!mSomeActivitiesChanged) {
                    return;
                }
                Runtime runtime = Runtime.getRuntime();
                long dalvikMax = runtime.maxMemory();
                long dalvikUsed = runtime.totalMemory() - runtime.freeMemory();
                if (dalvikUsed > ((3*dalvikMax)/4)) {
                    if (DEBUG_MEMORY_TRIM) Slog.d(TAG, "Dalvik max=" + (dalvikMax/1024)
                            + " total=" + (runtime.totalMemory()/1024)
                            + " used=" + (dalvikUsed/1024));
                    mSomeActivitiesChanged = false;
                    try {
                        mgr.releaseSomeActivities(mAppThread);
                    } catch (RemoteException e) {
                        throw e.rethrowFromSystemServer();
                    }
                }
            }
        });
    } else {
        // Don't set application object here -- if the system crashes,
        // we can't display an alert, we just want to die die die.
        android.ddm.DdmHandleAppName.setAppName("system_process",
                UserHandle.myUserId());
        try {
            mInstrumentation = new Instrumentation();
            mInstrumentation.basicInit(this);
            ContextImpl context = ContextImpl.createAppContext(
                    this, getSystemContext().mPackageInfo);
            mInitialApplication = context.mPackageInfo.makeApplication(true, null);
            mInitialApplication.onCreate();
        } catch (Exception e) {
            throw new RuntimeException(
                    "Unable to instantiate Application():" + e.toString(), e);
        }
    }

    // add dropbox logging to libcore
    DropBox.setReporter(new DropBoxReporter());

    ViewRootImpl.ConfigChangedCallback configChangedCallback
            = (Configuration globalConfig) -> {
        synchronized (mResourcesManager) {
            // We need to apply this change to the resources immediately, because upon returning
            // the view hierarchy will be informed about it.
            if (mResourcesManager.applyConfigurationToResourcesLocked(globalConfig,
                    null /* compat */)) {
                updateLocaleListFromAppContext(mInitialApplication.getApplicationContext(),
                        mResourcesManager.getConfiguration().getLocales());

                // This actually changed the resources! Tell everyone about it.
                if (mPendingConfiguration == null
                        || mPendingConfiguration.isOtherSeqNewer(globalConfig)) {
                    mPendingConfiguration = globalConfig;
                    sendMessage(H.CONFIGURATION_CHANGED, globalConfig);
                }
            }
        }
    };
    ViewRootImpl.addConfigCallback(configChangedCallback);
}
```

通过`ActivityManager.getService()`获取`ActivityManagerService`在当前进程的代理对象，然后调用`AMS`的`attachApplication`进行关联，参数`mAppThread`为`ApplicationThread`实例。

## 内部类ApplicationThread

先看类声明：

```java
private class ApplicationThread extends IApplicationThread.Stub {
  ...
}
```

该类继承自`IApplicationThread.Stub`，显然为`IApplicationThread`的远程实现；可见是通过`Binder`通讯，让`AMS`能够通知应用做一些事情。

## getHandler获取主线程Handler

```java
final H mH = new H();

final Handler getHandler() {
    return mH;
}

class H extends Handler {
    public static final int BIND_APPLICATION        = 110;
    public static final int EXIT_APPLICATION        = 111;
    public static final int RECEIVER                = 113;
    public static final int CREATE_SERVICE          = 114;
    public static final int SERVICE_ARGS            = 115;
    public static final int STOP_SERVICE            = 116;

    public static final int CONFIGURATION_CHANGED   = 118;
    public static final int CLEAN_UP_CONTEXT        = 119;
    public static final int GC_WHEN_IDLE            = 120;
    public static final int BIND_SERVICE            = 121;
    public static final int UNBIND_SERVICE          = 122;
    public static final int DUMP_SERVICE            = 123;
    public static final int LOW_MEMORY              = 124;
    public static final int PROFILER_CONTROL        = 127;
    public static final int CREATE_BACKUP_AGENT     = 128;
    public static final int DESTROY_BACKUP_AGENT    = 129;
    public static final int SUICIDE                 = 130;
    public static final int REMOVE_PROVIDER         = 131;
    public static final int ENABLE_JIT              = 132;
    public static final int DISPATCH_PACKAGE_BROADCAST = 133;
    public static final int SCHEDULE_CRASH          = 134;
    public static final int DUMP_HEAP               = 135;
    public static final int DUMP_ACTIVITY           = 136;
    public static final int SLEEPING                = 137;
    public static final int SET_CORE_SETTINGS       = 138;
    public static final int UPDATE_PACKAGE_COMPATIBILITY_INFO = 139;
    public static final int DUMP_PROVIDER           = 141;
    public static final int UNSTABLE_PROVIDER_DIED  = 142;
    public static final int REQUEST_ASSIST_CONTEXT_EXTRAS = 143;
    public static final int TRANSLUCENT_CONVERSION_COMPLETE = 144;
    public static final int INSTALL_PROVIDER        = 145;
    public static final int ON_NEW_ACTIVITY_OPTIONS = 146;
    public static final int ENTER_ANIMATION_COMPLETE = 149;
    public static final int START_BINDER_TRACKING = 150;
    public static final int STOP_BINDER_TRACKING_AND_DUMP = 151;
    public static final int LOCAL_VOICE_INTERACTION_STARTED = 154;
    public static final int ATTACH_AGENT = 155;
    public static final int APPLICATION_INFO_CHANGED = 156;
    public static final int RUN_ISOLATED_ENTRY_POINT = 158;
    public static final int EXECUTE_TRANSACTION = 159;
    public static final int RELAUNCH_ACTIVITY = 160;
    public static final int PURGE_RESOURCES = 161;

    ...
    public void handleMessage(Message msg) {
        if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
        switch (msg.what) {
            case BIND_APPLICATION:
                Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "bindApplication");
                AppBindData data = (AppBindData)msg.obj;
                handleBindApplication(data);
                Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                break;
            ...
        }
        ...
    }
}
```

可以见到定义了很多常量，同时通过查找可以发现，定义的这些消息类型的发送都是在`ApplicationThread`的函数中，由于`Binder`通讯时函数的调用是在`Binder`线程上，所以就需要这样一个`Handler`对象把任务都转发到主线程上处理。

## 总结

`ActivityThread`通过`ApplicationThread`和`AMS`进行进程间通讯，`AMS`以进程间通信的方式完成`ActivityThread`的请求后会回调`ApplicationThread`中的`Binder`方法，然后`ApplicationThread`会向`H`发送消息，`H`收到消息后会将`ApplicationThread`中的逻辑切换到`ActivityThread`中去执行，即切换到主线程中去执行。

关于`ActivityThread`的细节，需要后续结合`ActivityManagerService`来一起分析。