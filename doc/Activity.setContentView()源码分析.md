[TOC]

# Activity.setContentView()源码分析

## 带着疑问开始

都知道给`Activity`设置布局是在`onCreate()`中调用`setContentView(R.layout.xxx)`，但是调用该函数会发生什么？

## 着手分析

> 源码版本：（AOSP）android-9.0.0-r40
>
> 所需知识：对`LayoutInflater`的解析已经有一定的理解

### Activity.setContentView()

> 关于`setContentView()`多个重载函数，实现都类似，就选取使用频率最高的来分析

首先从`Activity.setContentView()`函数：

```java
public void setContentView(@LayoutRes int layoutResID) {
    getWindow().setContentView(layoutResID);
    initWindowDecorActionBar();
}

public Window getWindow() {
    return mWindow;
}
```

可看到，实际调用了`getWindow().setContentView(layoutResID)`函数，那看下`getWindow()`函数，返回了`mWindow`对象，显然该过程是交给`mWindow`对象来处理，那`mWindow`对象是什么？看下定义的地方：

```java
private Window mWindow;
```

> 查看`Window`源码可知它是一个抽象类，同时有唯一的子类`PhoneWindow`，关于`Window`更多内容不在这展开，先有个模糊的印象，后续会有文章详细的说明。

既然需要`mWindow`对象，那搜索下该对象的创建，很容易找到唯一的地方，`attach()`函数：

```java
final void attach(Context context, ActivityThread aThread,
        Instrumentation instr, IBinder token, int ident,
        Application application, Intent intent, ActivityInfo info,
        CharSequence title, Activity parent, String id,
        NonConfigurationInstances lastNonConfigurationInstances,
        Configuration config, String referrer, IVoiceInteractor voiceInteractor,
        Window window, ActivityConfigCallback activityConfigCallback) {
    attachBaseContext(context);

    mFragments.attachHost(null /*parent*/);
	// 在此创建window对象
    mWindow = new PhoneWindow(this, window, activityConfigCallback);
    mWindow.setWindowControllerCallback(this);
    mWindow.setCallback(this);
    mWindow.setOnWindowDismissedCallback(this);
    mWindow.getLayoutInflater().setPrivateFactory(this);
    if (info.softInputMode != WindowManager.LayoutParams.SOFT_INPUT_STATE_UNSPECIFIED) {
        mWindow.setSoftInputMode(info.softInputMode);
    }
    if (info.uiOptions != 0) {
        mWindow.setUiOptions(info.uiOptions);
    }
    mUiThread = Thread.currentThread();

    mMainThread = aThread;
    mInstrumentation = instr;
    mToken = token;
    mIdent = ident;
    mApplication = application;
    mIntent = intent;
    mReferrer = referrer;
    mComponent = intent.getComponent();
    mActivityInfo = info;
    mTitle = title;
    mParent = parent;
    mEmbeddedID = id;
    mLastNonConfigurationInstances = lastNonConfigurationInstances;
    if (voiceInteractor != null) {
        if (lastNonConfigurationInstances != null) {
            mVoiceInteractor = lastNonConfigurationInstances.voiceInteractor;
        } else {
            mVoiceInteractor = new VoiceInteractor(voiceInteractor, this, this,
                    Looper.myLooper());
        }
    }

    mWindow.setWindowManager(
            (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
            mToken, mComponent.flattenToString(),
            (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
    if (mParent != null) {
        mWindow.setContainer(mParent.getWindow());
    }
    mWindowManager = mWindow.getWindowManager();
    mCurrentConfig = config;

    mWindow.setColorMode(info.colorMode);

    setAutofillCompatibilityEnabled(application.isAutofillCompatibilityEnabled());
}
```

那`mWindow`对象有了，接下来就是进入`PhoneWindow.setContentView()`函数了。

### PhoneWindow.setContentView()

先贴出代码：

```java
@Override
public void setContentView(int layoutResID) {
    // Note: FEATURE_CONTENT_TRANSITIONS may be set in the process of installing the window
    // decor, when theme attributes and the like are crystalized. Do not check the feature
    // before this happens.
    if (mContentParent == null) {
        // 创建mDecor
        installDecor();
    } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        mContentParent.removeAllViews();
    }

    if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                getContext());
        transitionTo(newScene);
    } else {
        // 把传入的布局添加到mContentParent
        mLayoutInflater.inflate(layoutResID, mContentParent);
    }
    mContentParent.requestApplyInsets();
    final Callback cb = getCallback();
    if (cb != null && !isDestroyed()) {
        cb.onContentChanged();
    }
    mContentParentExplicitlySet = true;
}
```

看第6行，判断`mContentParent == null`，假设该条件满足，那会执行`installDecor()`函数；第19行，会通过`mLayoutInflater.inflate(layoutResID, mContentParent)`把传入的布局解析并添加到`mContentParent`上，已经知道传入的布局被添加到何处了，那接下来只要知道`mContentParent`是如何被创建的就可以了。

> 因为是被添加到`mContentParent`，也难怪`Activity`的接口会取名为`setContentView`了。

先看`installDecor()`函数调用：

```java
private void installDecor() {
    mForceDecorInstall = false;
    if (mDecor == null) {
        mDecor = generateDecor(-1);
        mDecor.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
        mDecor.setIsRootNamespace(true);
        if (!mInvalidatePanelMenuPosted && mInvalidatePanelMenuFeatures != 0) {
            mDecor.postOnAnimation(mInvalidatePanelMenuRunnable);
        }
    } else {
        mDecor.setWindow(this);
    }
    if (mContentParent == null) {
        mContentParent = generateLayout(mDecor);
        ...
    }
}
```

该函数主要做两件事：

1. 通过`generateDecor()`创建`mDecor`
2. 通过`generateLayout()`创建`mContentParent`

查看定义的地方，可知`mDecor`为`DecorView`，`mContentParent`为`ViewGroup`；

先看`generateDecor()`函数：

```java
protected DecorView generateDecor(int featureId) {
    // System process doesn't have application context and in that case we need to directly use
    // the context we have. Otherwise we want the application context, so we don't cling to the
    // activity.
    Context context;
    if (mUseDecorContext) {
        Context applicationContext = getContext().getApplicationContext();
        if (applicationContext == null) {
            context = getContext();
        } else {
            context = new DecorContext(applicationContext, getContext());
            if (mTheme != -1) {
                context.setTheme(mTheme);
            }
        }
    } else {
        context = getContext();
    }
    return new DecorView(context, featureId, this, getAttributes());
}
```

就是创建`DecorView`对象并返回；

接下来看`generateLayout()`函数：

```java
protected ViewGroup generateLayout(DecorView decor) {
    // Apply data from current theme.

    TypedArray a = getWindowStyle();

    ...

    // Inflate the window decor.

    int layoutResource;
    int features = getLocalFeatures();
    // System.out.println("Features: 0x" + Integer.toHexString(features));
    if ((features & (1 << FEATURE_SWIPE_TO_DISMISS)) != 0) {
        layoutResource = R.layout.screen_swipe_dismiss;
        setCloseOnSwipeEnabled(true);
    } else if ((features & ((1 << FEATURE_LEFT_ICON) | (1 << FEATURE_RIGHT_ICON))) != 0) {
        if (mIsFloating) {
            TypedValue res = new TypedValue();
            getContext().getTheme().resolveAttribute(
                    R.attr.dialogTitleIconsDecorLayout, res, true);
            layoutResource = res.resourceId;
        } else {
            layoutResource = R.layout.screen_title_icons;
        }
        // XXX Remove this once action bar supports these features.
        removeFeature(FEATURE_ACTION_BAR);
        // System.out.println("Title Icons!");
    } else if ((features & ((1 << FEATURE_PROGRESS) | (1 << FEATURE_INDETERMINATE_PROGRESS))) != 0
            && (features & (1 << FEATURE_ACTION_BAR)) == 0) {
        // Special case for a window with only a progress bar (and title).
        // XXX Need to have a no-title version of embedded windows.
        layoutResource = R.layout.screen_progress;
        // System.out.println("Progress!");
    } else if ((features & (1 << FEATURE_CUSTOM_TITLE)) != 0) {
        // Special case for a window with a custom title.
        // If the window is floating, we need a dialog layout
        if (mIsFloating) {
            TypedValue res = new TypedValue();
            getContext().getTheme().resolveAttribute(
                    R.attr.dialogCustomTitleDecorLayout, res, true);
            layoutResource = res.resourceId;
        } else {
            layoutResource = R.layout.screen_custom_title;
        }
        // XXX Remove this once action bar supports these features.
        removeFeature(FEATURE_ACTION_BAR);
    } else if ((features & (1 << FEATURE_NO_TITLE)) == 0) {
        // If no other features and not embedded, only need a title.
        // If the window is floating, we need a dialog layout
        if (mIsFloating) {
            TypedValue res = new TypedValue();
            getContext().getTheme().resolveAttribute(
                    R.attr.dialogTitleDecorLayout, res, true);
            layoutResource = res.resourceId;
        } else if ((features & (1 << FEATURE_ACTION_BAR)) != 0) {
            layoutResource = a.getResourceId(
                    R.styleable.Window_windowActionBarFullscreenDecorLayout,
                    R.layout.screen_action_bar);
        } else {
            layoutResource = R.layout.screen_title;
        }
        // System.out.println("Title!");
    } else if ((features & (1 << FEATURE_ACTION_MODE_OVERLAY)) != 0) {
        layoutResource = R.layout.screen_simple_overlay_action_mode;
    } else {
        // Embedded, so no decoration is needed.
        layoutResource = R.layout.screen_simple;
        // System.out.println("Simple!");
    }

    mDecor.startChanging();
    // 从确定的布局解析得到DecorView的root，并添加到DecorView上
    mDecor.onResourcesLoaded(mLayoutInflater, layoutResource);
    // 从DecorView中查找id为ID_ANDROID_CONTENT（com.android.internal.R.id.content）的控件
    ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
    if (contentParent == null) {
        throw new RuntimeException("Window couldn't find content container view");
    }

    ...

    mDecor.finishChanging();

    return contentParent;
}
```

函数内容比较多，但忽略掉省略的代码部分，剩余的核心就是：

1. 确定`layoutResource`并解析，解析得到的视图添加到`DecorView`
2. 从`DecorView`查找得到`contentParent`并返回

到此，`mContentParent`的获取过程也分析完成了。

根据该分析过程，可以梳理出对应的函数调用时序图：

![Activity.setContentView()时序图](./imgs/Activity.setContentView()时序图.jpg)

## 总结

由上节分析可知，`Activity.setContentView()`实质是通过`PhoneWindow.setContentView()`来实现的，每个`PhoneWindow`又有一个`DecorView`，该`DecorView`根据不同的配置解析不同的布局，并且布局中包含了id为`com.android.internal.R.id.content`的`ViewGroup`（一般都是`FrameLayout`），最终把开发者传入的布局添加到该视图上。