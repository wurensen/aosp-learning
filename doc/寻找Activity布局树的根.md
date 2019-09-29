[TOC]

# 寻找Activity布局树的根

## 带着疑问开始

在[《Activity.setContentView()源码分析》](./Activity.setContentView()源码分析.md)一文中，我们已经知道了调用`Activity.setContentView()`函数最终会把开发者传入的布局添加到`DecorView`视图树的`mContentParent`节点上，从而也带来了新的疑问：

1. `DecorView`是什么
2. `DecorView`跟`PhoneWindow`又是什么关系
3. 谁才是布局树真正的根

## 解惑之旅

这次我们从[《Activity.setContentView()源码分析》](./Activity.setContentView()源码分析.md)一文中的一张图开始：

![Activity.setContentView()时序图](./imgs/Activity.setContentView()时序图.jpg)

可以看到`Activity`在被创建后，在执行`attach()`函数时会创建`PhoneWindow`对象，之后又会在`PhoneWindow`的`installDecor()`中创建`DecorView`对象。

### 初识Window与PhoneWindow

如下为两者的类定义：

```java
/**
 * Abstract base class for a top-level window look and behavior policy.  An
 * instance of this class should be used as the top-level view added to the
 * window manager. It provides standard UI policies such as a background, title
 * area, default key processing, etc.
 *
 * <p>The only existing implementation of this abstract class is
 * android.view.PhoneWindow, which you should instantiate when needing a
 * Window.
 */
public abstract class Window {
    ...
}

/**
 * Android-specific Window.
 * <p>
 * todo: need to pull the generic functionality out into a base class
 * in android.widget.
 *
 * @hide
 */
public class PhoneWindow extends Window implements MenuBuilder.Callback {
    ...
}
```

从类说明上来看，只能有个大概的认知，`Window`作为抽象类，表示顶级窗体的样式和行为，该实例应该作为顶级`View`被添加到`window manager`，同时提到了`PhoneWindow`是唯一的实现类。

`PhoneWindow`本身并没有类似于`View`的功能，那如何作为顶级`View`添加到`window manager`的呢？

从上图分析可知道，`PhoneWindow`持有`DecorView`的引用，除了提供创建的函数`installDecor()`以外，`Window`接口还定义了`DecorView`获取函数：

```java
public abstract class Window {
    /**
     * Retrieve the top-level window decor view (containing the standard
     * window frame/decorations and the client's content inside of that), which
     * can be added as a window to the window manager.
     *
     * <p><em>Note that calling this function for the first time "locks in"
     * various window characteristics as described in
     * {@link #setContentView(View, android.view.ViewGroup.LayoutParams)}.</em></p>
     *
     * @return Returns the top-level window decor view.
     */
    public abstract View getDecorView();

    /**
     * Retrieve the current decor view, but only if it has already been created;
     * otherwise returns null.
     *
     * @return Returns the top-level window decor or null.
     * @see #getDecorView
     */
    public abstract View peekDecorView();
}
```

`PhoneWindow`实现了对应函数：

```java
@Override
public final View getDecorView() {
    if (mDecor == null || mForceDecorInstall) {
        installDecor();
    }
    return mDecor;
}

@Override
public final View peekDecorView() {
    return mDecor;
}
```

### 初识DecorView

查看`DecorView`的声明：

```java
public class DecorView extends FrameLayout implements RootViewSurfaceTaker, WindowCallbacks {
	...
}
```

该类没有说明注释，但是通过查找可以发现该类的构造函数只在`PhoneWindow.generateDecor()`中调用。

**可见`DecorView`就是作为`PhoneWindow`的视图存在的**，所以`Window`类的说明“作为顶级`View`添加到`window manager`”，其中的`View`指的应该就是`DecorView`，而`Window`本身只是表示一种抽象的窗体概念。

那`window manager`是指什么？可以知道和`Window`有关，来看下`Window`的几个函数：

```java
/**
 * Set the window manager for use by this Window to, for example,
 * display panels.  This is <em>not</em> used for displaying the
 * Window itself -- that must be done by the client.
 *
 * @param wm The window manager for adding new windows.
 */
public void setWindowManager(WindowManager wm, IBinder appToken, String appName) {
    setWindowManager(wm, appToken, appName, false);
}

/**
 * Set the window manager for use by this Window to, for example,
 * display panels.  This is <em>not</em> used for displaying the
 * Window itself -- that must be done by the client.
 *
 * @param wm The window manager for adding new windows.
 */
public void setWindowManager(WindowManager wm, IBinder appToken, String appName,
        boolean hardwareAccelerated) {
    mAppToken = appToken;
    mAppName = appName;
    mHardwareAccelerated = hardwareAccelerated
            || SystemProperties.getBoolean(PROPERTY_HARDWARE_UI, false);
    if (wm == null) {
        wm = (WindowManager)mContext.getSystemService(Context.WINDOW_SERVICE);
    }
    mWindowManager = ((WindowManagerImpl)wm).createLocalWindowManager(this);
}

/**
 * Return the window manager allowing this Window to display its own
 * windows.
 *
 * @return WindowManager The ViewManager.
 */
public WindowManager getWindowManager() {
    return mWindowManager;
}
```

`Window`中定义了3个与`WindowManager`设置和获取相关的函数，注意到代码`mWindowManager = ((WindowManagerImpl)wm).createLocalWindowManager(this)`，说明每个`Window`都会创建一个对应的`WindowManager`实例。`WindowManagerImpl.createLocalWindowManager()`代码如下：

```java
public WindowManagerImpl createLocalWindowManager(Window parentWindow) {
    return new WindowManagerImpl(mContext, parentWindow);
}
```

同时可以知道，通过该路径创建的`WindowManagerImpl`实例，`mParentWindow`成员变量都是对应的`Window`调用者。

接下来先认识下`WindowManager`和`WindowManagerImpl`。

### 初识WindowManager与WindowManagerImpl

看下`WindowManager`以及父类和子类：

```java
/** Interface to let you add and remove child views to an Activity. To get an instance
  * of this class, call {@link android.content.Context#getSystemService(java.lang.String) Context.getSystemService()}.
  */
public interface ViewManager
{
    /**
     * Assign the passed LayoutParams to the passed View and add the view to the window.
     * <p>Throws {@link android.view.WindowManager.BadTokenException} for certain programming
     * errors, such as adding a second view to a window without removing the first view.
     * <p>Throws {@link android.view.WindowManager.InvalidDisplayException} if the window is on a
     * secondary {@link Display} and the specified display can't be found
     * (see {@link android.app.Presentation}).
     * @param view The view to be added to this window.
     * @param params The LayoutParams to assign to view.
     */
    public void addView(View view, ViewGroup.LayoutParams params);
    public void updateViewLayout(View view, ViewGroup.LayoutParams params);
    public void removeView(View view);
}

/**
 * The interface that apps use to talk to the window manager.
 * </p><p>
 * Each window manager instance is bound to a particular {@link Display}.
 * To obtain a {@link WindowManager} for a different display, use
 * {@link Context#createDisplayContext} to obtain a {@link Context} for that
 * display, then use <code>Context.getSystemService(Context.WINDOW_SERVICE)</code>
 * to get the WindowManager.
 * </p><p>
 * The simplest way to show a window on another display is to create a
 * {@link Presentation}.  The presentation will automatically obtain a
 * {@link WindowManager} and {@link Context} for that display.
 * </p>
 */
@SystemService(Context.WINDOW_SERVICE)
public interface WindowManager extends ViewManager {
    ...
}

/**
 * Provides low-level communication with the system window manager for
 * operations that are bound to a particular context, display or parent window.
 * Instances of this object are sensitive to the compatibility info associated
 * with the running application.
 *
 * This object implements the {@link ViewManager} interface,
 * allowing you to add any View subclass as a top-level window on the screen.
 * Additional window manager specific layout parameters are defined for
 * control over how windows are displayed.  It also implements the {@link WindowManager}
 * interface, allowing you to control the displays attached to the device.
 *
 * <p>Applications will not normally use WindowManager directly, instead relying
 * on the higher-level facilities in {@link android.app.Activity} and
 * {@link android.app.Dialog}.
 *
 * <p>Even for low-level window manager access, it is almost never correct to use
 * this class.  For example, {@link android.app.Activity#getWindowManager}
 * provides a window manager for adding windows that are associated with that
 * activity -- the window manager will not normally allow you to add arbitrary
 * windows that are not associated with an activity.
 *
 * @see WindowManager
 * @see WindowManagerGlobal
 * @hide
 */
public final class WindowManagerImpl implements WindowManager {
    ...
}
```

`ViewManager`接口：定义视图具有增、删、改的行为

`WindowManager`接口：继承自`ViewManager`，说明具有`ViewManager`定义的行为，同时类说明上说到该接口是`app`用来与`window manager`通讯时用到的（所以自己并不是真正的`window manager`，应该只是定义了应用端与真正`window manager`通讯的接口）

`WindowManagerImpl`最终类：作为`WindowManager`的唯一实现类，类的第一段说明提到”该类给绑定到特定的上下文、显示器、或父窗体操作，提供了和`system window manager`（所以真正的`window manager`是系统级别的）的`low-level`通讯“，再看第二段说明”该对象实现`ViewManager`接口，让你能够把任意`View`的子类作为顶级窗体添加到屏幕上“，第三段提到“应用通常不会直接使用`WindowManager`，而是使用`Activity`和`Dialog`来代替”

> 有悬浮窗开发经验的，或者对`Dialog`、`Toast`源码有去研究，对`WindowManager`应该会比较熟悉，但可能只是知道用法，但还是无法准确说出来它到底是什么

### 寻找布局树的根

大概认识了上面提到的几个关键类，接下来可以去分析`DecorView`到底是怎样被添加到系统的`window manager`了。

如何找到对应的函数调用栈？可以从`WindowManagerImpl.addView()`开始，发现调用该函数的地方比较多，可以通过打断点的方式来找到准确的入口。通过在编译后运行的模拟器的上随意打开一个`app`，如`Calendar`，可以找到调用的地方：`ActivityThread.handleResumeActivity()`，代码如下：

```java
@Override
public void handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward,
        String reason) {
    // If we are getting ready to gc after going to the background, well
    // we are back active so skip it.
    unscheduleGcIdler();
    mSomeActivitiesChanged = true;

    // TODO Push resumeArgs into the activity for consideration
    final ActivityClientRecord r = performResumeActivity(token, finalStateRequest, reason);
    if (r == null) {
        // We didn't actually resume the activity, so skipping any follow-up actions.
        return;
    }

    final Activity a = r.activity;

    if (localLOGV) {
        Slog.v(TAG, "Resume " + r + " started activity: " + a.mStartedActivity
                + ", hideForNow: " + r.hideForNow + ", finished: " + a.mFinished);
    }

    final int forwardBit = isForward
            ? WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION : 0;

    // If the window hasn't yet been added to the window manager,
    // and this guy didn't finish itself or start another activity,
    // then go ahead and add the window.
    boolean willBeVisible = !a.mStartedActivity;
    if (!willBeVisible) {
        try {
            willBeVisible = ActivityManager.getService().willActivityBeVisible(
                    a.getActivityToken());
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
    if (r.window == null && !a.mFinished && willBeVisible) {
        r.window = r.activity.getWindow(); // 获取window，其实就是PhoneWindow
        View decor = r.window.getDecorView(); // 获取decor，其实就是DecorView
        decor.setVisibility(View.INVISIBLE); // 先设置decor不可见
        ViewManager wm = a.getWindowManager(); // wm其实就是WindowManagerImpl
        WindowManager.LayoutParams l = r.window.getAttributes();
        a.mDecor = decor;
        l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION; // 设置type参数
        l.softInputMode |= forwardBit;
        if (r.mPreserveWindow) {
            a.mWindowAdded = true;
            r.mPreserveWindow = false;
            // Normally the ViewRoot sets up callbacks with the Activity
            // in addView->ViewRootImpl#setView. If we are instead reusing
            // the decor view we have to notify the view root that the
            // callbacks may have changed.
            ViewRootImpl impl = decor.getViewRootImpl();
            if (impl != null) {
                impl.notifyChildRebuilt();
            }
        }
        if (a.mVisibleFromClient) {
            if (!a.mWindowAdded) {
                a.mWindowAdded = true;
                // 在这里调用了WindowManagerImpl的addView函数
                wm.addView(decor, l);
            } else {
                // The activity will get a callback for this {@link LayoutParams} change
                // earlier. However, at that time the decor will not be set (this is set
                // in this method), so no action will be taken. This call ensures the
                // callback occurs with the decor set.
                a.onWindowAttributesChanged(l);
            }
        }

        // If the window has already been added, but during resume
        // we started another activity, then don't yet make the
        // window visible.
    } else if (!willBeVisible) {
        if (localLOGV) Slog.v(TAG, "Launch " + r + " mStartedActivity set");
        r.hideForNow = true;
    }

    // Get rid of anything left hanging around.
    cleanUpPendingRemoveWindows(r, false /* force */);

    // The window is now visible if it has been added, we are not
    // simply finishing, and we are not starting another activity.
    if (!r.activity.mFinished && willBeVisible && r.activity.mDecor != null && !r.hideForNow) {
        if (r.newConfig != null) {
            performConfigurationChangedForActivity(r, r.newConfig);
            if (DEBUG_CONFIGURATION) {
                Slog.v(TAG, "Resuming activity " + r.activityInfo.name + " with newConfig "
                        + r.activity.mCurrentConfig);
            }
            r.newConfig = null;
        }
        if (localLOGV) Slog.v(TAG, "Resuming " + r + " with isForward=" + isForward);
        WindowManager.LayoutParams l = r.window.getAttributes();
        if ((l.softInputMode
                & WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION)
                != forwardBit) {
            l.softInputMode = (l.softInputMode
                    & (~WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION))
                    | forwardBit;
            if (r.activity.mVisibleFromClient) {
                ViewManager wm = a.getWindowManager();
                View decor = r.window.getDecorView();
                wm.updateViewLayout(decor, l);
            }
        }

        r.activity.mVisibleFromServer = true;
        mNumVisibleActivities++;
        // 在这里调用activity.makeVisible，把DecorView设置成可见
        if (r.activity.mVisibleFromClient) {
            r.activity.makeVisible();
        }
    }

    r.nextIdle = mNewActivities;
    mNewActivities = r;
    if (localLOGV) Slog.v(TAG, "Scheduling idle handler for " + r);
    Looper.myQueue().addIdleHandler(new Idler());
}
```

得到几个相关的关键点：

1. 从`activity`中获取`PhoneWindow`、`DecorView`以及`WindowManagerImpl`
2. 然后调用`WindowManagerImpl.addView()`函数把`DecorView`添加进去

知道了是在何处调用的，那接下来就看`WindowManagerImpl.addView()`是如何把`DecorView`添加的：

```java
private final WindowManagerGlobal mGlobal = WindowManagerGlobal.getInstance();

@Override
public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
    applyDefaultToken(params);
    mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
}
```

可以看到转发给了`mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow)`执行，可以看到`mGlobal`为`WindowManagerGlobal`实例，并且进入`WindowManagerGlobal.getInstance()`查看可以知道该实例是个单例，也就是，该实例在整个进程中唯一。

### 初识WindowManagerGlobal

接下来看`mGlobal.addView()`：

```java
...
private final ArrayList<View> mViews = new ArrayList<View>();
private final ArrayList<ViewRootImpl> mRoots = new ArrayList<ViewRootImpl>();
private final ArrayList<WindowManager.LayoutParams> mParams =
        new ArrayList<WindowManager.LayoutParams>();
private final ArraySet<View> mDyingViews = new ArraySet<View>();
...
    
public void addView(View view, ViewGroup.LayoutParams params,
        Display display, Window parentWindow) {
    ...

    final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams) params;
    if (parentWindow != null) {
        // activity调用的路径，parentWindow不为null
        parentWindow.adjustLayoutParamsForSubWindow(wparams);
    } else {
        // If there's no parent, then hardware acceleration for this view is
        // set from the application's hardware acceleration setting.
        final Context context = view.getContext();
        if (context != null
                && (context.getApplicationInfo().flags
                        & ApplicationInfo.FLAG_HARDWARE_ACCELERATED) != 0) {
            wparams.flags |= WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED;
        }
    }

    ViewRootImpl root;
    View panelParentView = null;

    synchronized (mLock) {
        // Start watching for system property changes.
        ...

        int index = findViewLocked(view, false);
        if (index >= 0) {
            if (mDyingViews.contains(view)) {
                // Don't wait for MSG_DIE to make it's way through root's queue.
                mRoots.get(index).doDie();
            } else {
                throw new IllegalStateException("View " + view
                        + " has already been added to the window manager.");
            }
            // The previous removeView() had not completed executing. Now it has.
        }

        // If this is a panel window, then find the window it is being
        // attached to for future reference.
        if (wparams.type >= WindowManager.LayoutParams.FIRST_SUB_WINDOW &&
                wparams.type <= WindowManager.LayoutParams.LAST_SUB_WINDOW) {
            final int count = mViews.size();
            for (int i = 0; i < count; i++) {
                if (mRoots.get(i).mWindow.asBinder() == wparams.token) {
                    panelParentView = mViews.get(i);
                }
            }
        }
        // 创建ViewRootImpl对象
        root = new ViewRootImpl(view.getContext(), display);

        view.setLayoutParams(wparams);

        mViews.add(view); // 添加DecorView到集合
        mRoots.add(root); // 添加ViewRootImpl到集合
        mParams.add(wparams); // 添加布局参数到集合

        // do this last because it fires off messages to start doing things
        try {
            // 最后再把DecorView设置给了root，即ViewRootImpl
            root.setView(view, wparams, panelParentView);
        } catch (RuntimeException e) {
            // BadTokenException or InvalidDisplayException, clean up.
            if (index >= 0) {
                removeViewLocked(index, true);
            }
            throw e;
        }
    }
}
```

看来`WindowManagerGlobal`也只是负责创建`ViewRootImpl`，并管理起来，因为`WindowManagerGlobal`是单例，所以它管理着当前进程所有的`ViewRootImpl`、`DecorView`（当前分析的场景，传入的`View`参数就是`DecorView`）以及对应的`WindowManager.LayoutParams`，最终还会把传入的`View`设置给`ViewRootImpl`。

那`ViewRootImpl`又是什么？

### 初识ViewRootImpl

照旧先看下类定义：

```java
/**
 * The top of a view hierarchy, implementing the needed protocol between View
 * and the WindowManager.  This is for the most part an internal implementation
 * detail of {@link WindowManagerGlobal}.
 *
 * {@hide}
 */
@SuppressWarnings({"EmptyCatchBlock", "PointlessBooleanExpression"})
public final class ViewRootImpl implements ViewParent,
        View.AttachInfo.Callbacks, ThreadedRenderer.DrawCallbacks {
        ...
}
```

大概翻译下：作为视图树的最顶层，即根，实现了`View`和`WindowManager`之间必须的协议。

还是不好理解，那先看`setView()`函数：

```java
/**
 * We have one child
 */
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
    synchronized (this) {
        if (mView == null) {
            mView = view; // 保存传入的View

            mAttachInfo.mDisplayState = mDisplay.getState();
            mDisplayManager.registerDisplayListener(mDisplayListener, mHandler);

            mViewLayoutDirectionInitial = mView.getRawLayoutDirection();
            mFallbackEventHandler.setView(view);
            mWindowAttributes.copyFrom(attrs);
            if (mWindowAttributes.packageName == null) {
                mWindowAttributes.packageName = mBasePackageName;
            }
            attrs = mWindowAttributes;
            setTag();

            ...
            // Keep track of the actual window flags supplied by the client.
            mClientWindowLayoutFlags = attrs.flags;

            setAccessibilityFocus(null, null);

            ...

            // Compute surface insets required to draw at specified Z value.
            // TODO: Use real shadow insets for a constant max Z.
            if (!attrs.hasManualSurfaceInsets) {
                attrs.setSurfaceInsets(view, false /*manual*/, true /*preservePrevious*/);
            }

            CompatibilityInfo compatibilityInfo =
                    mDisplay.getDisplayAdjustments().getCompatibilityInfo();
            mTranslator = compatibilityInfo.getTranslator();

            ...

            boolean restore = false;
            if (mTranslator != null) {
                mSurface.setCompatibilityTranslator(mTranslator);
                restore = true;
                attrs.backup();
                mTranslator.translateWindowLayout(attrs);
            }
            if (DEBUG_LAYOUT) Log.d(mTag, "WindowLayout in setView:" + attrs);

            if (!compatibilityInfo.supportsScreen()) {
                attrs.privateFlags |= WindowManager.LayoutParams.PRIVATE_FLAG_COMPATIBLE_WINDOW;
                mLastInCompatMode = true;
            }

            mSoftInputMode = attrs.softInputMode;
            mWindowAttributesChanged = true;
            mWindowAttributesChangesFlag = WindowManager.LayoutParams.EVERYTHING_CHANGED;
            mAttachInfo.mRootView = view;
            mAttachInfo.mScalingRequired = mTranslator != null;
            mAttachInfo.mApplicationScale =
                    mTranslator == null ? 1.0f : mTranslator.applicationScale;
            if (panelParentView != null) {
                mAttachInfo.mPanelParentWindowToken
                        = panelParentView.getApplicationWindowToken();
            }
            mAdded = true;
            int res; /* = WindowManagerImpl.ADD_OKAY; */

            // Schedule the first layout -before- adding to the window
            // manager, to make sure we do the relayout before receiving
            // any other events from the system.
            requestLayout();
            if ((mWindowAttributes.inputFeatures
                    & WindowManager.LayoutParams.INPUT_FEATURE_NO_INPUT_CHANNEL) == 0) {
                mInputChannel = new InputChannel();
            }
            mForceDecorViewVisibility = (mWindowAttributes.privateFlags
                    & PRIVATE_FLAG_FORCE_DECOR_VIEW_VISIBILITY) != 0;
            try {
                mOrigWindowType = mWindowAttributes.type;
                mAttachInfo.mRecomputeGlobalAttributes = true;
                collectViewAttributes();
                // 通过Session添加到display并返回结果
                res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                        getHostVisibility(), mDisplay.getDisplayId(), mWinFrame,
                        mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                        mAttachInfo.mOutsets, mAttachInfo.mDisplayCutout, mInputChannel);
            } catch (RemoteException e) {
                mAdded = false;
                mView = null;
                mAttachInfo.mRootView = null;
                mInputChannel = null;
                mFallbackEventHandler.setView(null);
                unscheduleTraversals();
                setAccessibilityFocus(null, null);
                throw new RuntimeException("Adding window failed", e);
            } finally {
                if (restore) {
                    attrs.restore();
                }
            }

            if (mTranslator != null) {
                mTranslator.translateRectInScreenToAppWindow(mAttachInfo.mContentInsets);
            }
            mPendingOverscanInsets.set(0, 0, 0, 0);
            mPendingContentInsets.set(mAttachInfo.mContentInsets);
            mPendingStableInsets.set(mAttachInfo.mStableInsets);
            mPendingDisplayCutout.set(mAttachInfo.mDisplayCutout);
            mPendingVisibleInsets.set(0, 0, 0, 0);
            mAttachInfo.mAlwaysConsumeNavBar =
                    (res & WindowManagerGlobal.ADD_FLAG_ALWAYS_CONSUME_NAV_BAR) != 0;
            mPendingAlwaysConsumeNavBar = mAttachInfo.mAlwaysConsumeNavBar;
            if (DEBUG_LAYOUT) Log.v(mTag, "Added window " + mWindow);
            // 根据返回结果判断是否添加成功
            if (res < WindowManagerGlobal.ADD_OKAY) {
                mAttachInfo.mRootView = null;
                mAdded = false;
                mFallbackEventHandler.setView(null);
                unscheduleTraversals();
                setAccessibilityFocus(null, null);
                // 以下各种崩溃相信大家或多或少都见过
                switch (res) {
                    case WindowManagerGlobal.ADD_BAD_APP_TOKEN:
                    case WindowManagerGlobal.ADD_BAD_SUBWINDOW_TOKEN:
                        throw new WindowManager.BadTokenException(
                                "Unable to add window -- token " + attrs.token
                                + " is not valid; is your activity running?");
                    case WindowManagerGlobal.ADD_NOT_APP_TOKEN:
                        throw new WindowManager.BadTokenException(
                                "Unable to add window -- token " + attrs.token
                                + " is not for an application");
                    case WindowManagerGlobal.ADD_APP_EXITING:
                        throw new WindowManager.BadTokenException(
                                "Unable to add window -- app for token " + attrs.token
                                + " is exiting");
                    case WindowManagerGlobal.ADD_DUPLICATE_ADD:
                        throw new WindowManager.BadTokenException(
                                "Unable to add window -- window " + mWindow
                                + " has already been added");
                    case WindowManagerGlobal.ADD_STARTING_NOT_NEEDED:
                        // Silently ignore -- we would have just removed it
                        // right away, anyway.
                        return;
                    case WindowManagerGlobal.ADD_MULTIPLE_SINGLETON:
                        throw new WindowManager.BadTokenException("Unable to add window "
                                + mWindow + " -- another window of type "
                                + mWindowAttributes.type + " already exists");
                    case WindowManagerGlobal.ADD_PERMISSION_DENIED:
                        throw new WindowManager.BadTokenException("Unable to add window "
                                + mWindow + " -- permission denied for window type "
                                + mWindowAttributes.type);
                    case WindowManagerGlobal.ADD_INVALID_DISPLAY:
                        throw new WindowManager.InvalidDisplayException("Unable to add window "
                                + mWindow + " -- the specified display can not be found");
                    case WindowManagerGlobal.ADD_INVALID_TYPE:
                        throw new WindowManager.InvalidDisplayException("Unable to add window "
                                + mWindow + " -- the specified window type "
                                + mWindowAttributes.type + " is not valid");
                }
                throw new RuntimeException(
                        "Unable to add window -- unknown error code " + res);
            }

            if (view instanceof RootViewSurfaceTaker) {
                mInputQueueCallback =
                    ((RootViewSurfaceTaker)view).willYouTakeTheInputQueue();
            }
            // 创建输入事件接收器
            if (mInputChannel != null) {
                if (mInputQueueCallback != null) {
                    mInputQueue = new InputQueue();
                    mInputQueueCallback.onInputQueueCreated(mInputQueue);
                }
                mInputEventReceiver = new WindowInputEventReceiver(mInputChannel,
                        Looper.myLooper());
            }

            view.assignParent(this);
            mAddedTouchMode = (res & WindowManagerGlobal.ADD_FLAG_IN_TOUCH_MODE) != 0;
            mAppVisible = (res & WindowManagerGlobal.ADD_FLAG_APP_VISIBLE) != 0;

            if (mAccessibilityManager.isEnabled()) {
                mAccessibilityInteractionConnectionManager.ensureConnection();
            }

            if (view.getImportantForAccessibility() == View.IMPORTANT_FOR_ACCESSIBILITY_AUTO) {
                view.setImportantForAccessibility(View.IMPORTANT_FOR_ACCESSIBILITY_YES);
            }

            // Set up the input pipeline.
            CharSequence counterSuffix = attrs.getTitle();
            mSyntheticInputStage = new SyntheticInputStage();
            InputStage viewPostImeStage = new ViewPostImeInputStage(mSyntheticInputStage);
            InputStage nativePostImeStage = new NativePostImeInputStage(viewPostImeStage,
                    "aq:native-post-ime:" + counterSuffix);
            InputStage earlyPostImeStage = new EarlyPostImeInputStage(nativePostImeStage);
            InputStage imeStage = new ImeInputStage(earlyPostImeStage,
                    "aq:ime:" + counterSuffix);
            InputStage viewPreImeStage = new ViewPreImeInputStage(imeStage);
            InputStage nativePreImeStage = new NativePreImeInputStage(viewPreImeStage,
                    "aq:native-pre-ime:" + counterSuffix);

            mFirstInputStage = nativePreImeStage;
            mFirstPostImeInputStage = earlyPostImeStage;
            mPendingInputEventQueueLengthCounterName = "aq:pending:" + counterSuffix;
        }
    }
}
```

可以看到传入的`View`到此保存起来之后，就未再添加到其它的视图上了。所以也正如类注释所说，该类作为视图树的根存在。

继续深入去看该类的源码，可以发现该类并不是`View`，而是作为视图树的根，负责调度所有视图的测量、布局、绘制流程，以及负责分发事件：实体按键事件和触摸事件。

## 得到答案

经过以上的分析，可以知道`Window`表示抽象的窗体，而`DecorView`作为`PhoneWindow`的视图形式存在，并通过在创建`PhoneWindow`时创建的对应`WindowManagerImpl`实例，调用到进程唯一管理实例`WindowManagerGlobal`，最终把`DecorView`添加到新创建的`ViewRootImpl`上，即`Activity`视图树的根。

> 上节关于跨进程的部分暂时不去分析，等理解了`IBinder`跨进程通讯机制再回过头分析

## 新的疑问

1. 除了`Activity`，`Dialog`、`PopupWindow`、`Toast`的窗口机制又是如何的
2. 应该如何正确使用`WindowManager`来实现悬浮窗需求，以免发生`BadTokenException`崩溃