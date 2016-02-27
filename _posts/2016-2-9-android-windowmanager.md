---
layout: post
title: 探秘Android之WindowManager
---
在Android的世界里，我们可以通过`WindowManager`将一个视图添加到屏幕上。下面就是实现此需求的两条关键语句：

	WindowManager wm = (WindowManager) contex.getSystemService(Context.WINDOW_SERVICE);

	...

	wm.addView(view, layoutParam);

这篇文章将以这两条语句作为切入点，探究一下与`WindowManager`相关的源码（基于5.1.1系统）。

<!-- more -->

### `WindowManager`是什么

根据`WindowManager`的定义

	public interface WindowManager extends ViewManager

可以看出`WindowManager`是一个继承自`ViewManager`的接口。所以`WindowManager`继承了`ViewManager`中的两个主要函数：

	public void addView(View view, ViewGroup.LayoutParams params);
	public void removeView(View view);

从函数名便可推断这两个函数的作用分别是往屏幕添加视图以及移除之前添加的视图。

既然`WindowManager`只是一个接口，那必然有类实现了这个接口。让我们回到这条语句

	WindowManager wm = (WindowManager) contex.getSystemService(Context.WINDOW_SERVICE);

去探究一下通过`contex.getSystemService(Context.WINDOW_SERVICE)`拿到的`WindowManager`到底是什么。

因为与UI紧密相关的`Context`是`Activity`，所以假定这里的`context`是一个`Activity`。

首先看一下`Activity`中`getSystemService`的实现：

	@Override
	5033    public Object getSystemService(@ServiceName @NonNull String name) {
	5034        if (getBaseContext() == null) {
	5035            throw new IllegalStateException(
	5036                    "System services not available to Activities before onCreate()");
	5037        }
	5038
	5039        if (WINDOW_SERVICE.equals(name)) {
	5040            return mWindowManager;
	5041        } else if (SEARCH_SERVICE.equals(name)) {
	5042            ensureSearchManager();
	5043            return mSearchManager;
	5044        }
	5045        return super.getSystemService(name);
	5046    }

通过5039行的`if`语句可以看出，如果参数`name`是`WINDOW_SERVICE`则直接返回`Activity`的成员变量`mWindowManager`. 接下来需要找到`mWindowManager`被初始化的地方。它的初始化紧随`Activity`的初始化。而`Activity`的初始化在`ActivityThread.performLaunchActivity`中进行。这里的`ActivityThread`就是我们通常所说的主线程或者UI线程。摘取`ActivityThread.performLaunchActivity`中的几条关键语句：

	private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
	...
	       Activity activity = null;
	...
	        activity = mInstrumentation.newActivity(
	                    cl, component.getClassName(), r.intent);
	...
	        activity.attach(appContext, this, getInstrumentation(), r.token,
	                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
	                        r.embeddedID, r.lastNonConfigurationInstances, config,
	                        r.referrer, r.voiceInteractor);

当`activity`被创建出来后其`attach`方法即被调用。

	5922    final void attach(Context context, ActivityThread aThread,
	5923            Instrumentation instr, IBinder token, int ident,
	5924            Application application, Intent intent, ActivityInfo info,
	5925            CharSequence title, Activity parent, String id,
	5926            NonConfigurationInstances lastNonConfigurationInstances,
	5927            Configuration config, String referrer, IVoiceInteractor voiceInteractor) {

	           ...      

	5932        mWindow = PolicyManager.makeNewWindow(this);

	           ...

	5966        mWindow.setWindowManager(
	5967                (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
	5968                mToken, mComponent.flattenToString(),
	5969                (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
	5970        if (mParent != null) {
	5971            mWindow.setContainer(mParent.getWindow());
	5972        }
	5973        mWindowManager = mWindow.getWindowManager();
	5974        mCurrentConfig = config;
	5975    }

在第5932行，`Activity`的成员变量`mWindow`是一个`Window`类对象。`Window`是对`Activity`或者`Dialog`的视图的一种上层抽象。`Window`是一个抽象类，而`PhoneWindow`继承了`Window`并且实现了其中的一些关键方法。`PhoneWindow`中有一个类型为`DecorView`的成员变量`mDecor`，表示的是`Activity`对应的视图的最外层容器。`PolicyManager.makeNewWindow(this)`正好返回了一个`PhoneWindow`对象。

接下来在5966行，`Window.setWindowManager`被调用。该函数需要四个参数。首先我们看第一个参数`(WindowManager)context.getSystemService(Context.WINDOW_SERVICE)`. 在这里`context.getSystemService(Context.WINDOW_SERVICE)`又一次被调用并且返回一个`WindowManager`对象。不过这里的`context`不是某个`Activity`对象，而是一个`ContextImpl`对象。接下来看一下`ContextImpl.getSystemService`的实现。

	@Override
	public Object getSystemService(String name) {
	    ServiceFetcher fetcher = SYSTEM_SERVICE_MAP.get(name);
	    return fetcher == null ? null : fetcher.getService(this);
	}

可以看到`getSystemService`会根据参数`name`从`SYSTEM_SERVICE_MAP`中拿到对应的`ServiceFetcher`对象，然后通过`ServiceFetcher.getService`返回具体的对象。在`ContextImpl`中有一个函数的作用是往`SYSTEM_SERVICE_MAP`中放入`(name, ServiceFetcher)`映射。

	private static void registerService(String serviceName, ServiceFetcher fetcher) {
	    if (!(fetcher instanceof StaticServiceFetcher)) {
	        fetcher.mContextCacheIndex = sNextPerContextServiceCacheIndex++;
	    }
	    SYSTEM_SERVICE_MAP.put(serviceName, fetcher);
	}

另外在`ContextImpl`中有一个static语句块，里面通过多次调用`registerService`方法将所有可能的`(name, ServiceFetcher)`映射放入了`SYSTEM_SERVICE_MAP`. 这里我们特别关注一下`WINDOW_SERVICE`:

	634     registerService(WINDOW_SERVICE, new ServiceFetcher() {
	635             Display mDefaultDisplay;
	636             public Object getService(ContextImpl ctx) {
	637                 Display display = ctx.mDisplay;
	638                 if (display == null) {
	639                     if (mDefaultDisplay == null) {
	640                         DisplayManager dm = (DisplayManager)ctx.getOuterContext().getSystemService(Context.DISPLAY_SERVICE);
	642                         mDefaultDisplay = dm.getDisplay(Display.DEFAULT_DISPLAY);
	643                     }
	644                     display = mDefaultDisplay;
	645                 }
	646                 return new WindowManagerImpl(display);
	647             }});

可以看到`getService`返回了一个`WindowManagerImpl`对象。之前提到`WindowManager`只是一个接口，而这里的`WindowManagerImpl`正好实现了`WindowManager`. 

	public final class WindowManagerImpl implements WindowManager {
	    private final WindowManagerGlobal mGlobal = WindowManagerGlobal.getInstance();
	    private final Display mDisplay;
	    private final Window mParentWindow;

	    private IBinder mDefaultToken;

	    public WindowManagerImpl(Display display) {
	        this(display, null);
	    }

	    private WindowManagerImpl(Display display, Window parentWindow) {
	        mDisplay = display;
	        mParentWindow = parentWindow;
	    }

	    ...
	}

有一点需要注意：这里用到了`WindowManagerImpl`的只需一个参数的构造函数。通过上面的类的定义可以看到，`WindowManagerImpl`还有一个需要两个参数的构造函数，这个构造函数的调用接下来就会看到。还有我们可以看到`WindowManagerImpl`中有一个类型为`WindowManagerGlobal`的成员变量`mGlobal`，使用了单例模式，它的作用将在下节说明。

现在回到`Activity.attach`方法的5966行，我们已经知道第一个参数是一个`WindowManagerImpl`对象。接着就看一下`Window.setWindowManager`的具体实现。

	539     public void setWindowManager(WindowManager wm, IBinder appToken, String appName,
	540             boolean hardwareAccelerated) {
	541         mAppToken = appToken;
	542         mAppName = appName;
	543         mHardwareAccelerated = hardwareAccelerated
	544                 || SystemProperties.getBoolean(PROPERTY_HARDWARE_UI, false);
	545         if (wm == null) {
	546             wm = (WindowManager)mContext.getSystemService(Context.WINDOW_SERVICE);
	547         }
	548         mWindowManager = ((WindowManagerImpl)wm).createLocalWindowManager(this);
	549     }

这里有两个比较关键的点：

- 第541行，将参数`appToken`赋给了`Window`的成员变量`mAppToken`. `appToken`是一个`Binder`对象，在`ActivityManagerService`, `WindowManagerService`等系统服务中通过它来标识`Activity`. 让`Window`的成员变量`mAppToken`指向`appToken`，就好比这个`Window`得到了与之对应的`Activity`的身份通行证。这样一来，`WindowManagerService`就能知道这个`Window`是属于哪个`Activity`了。

- 第548行，调用`WindowManagerImpl.createLocalWindowManager`创建了一个新的`WindowManagerImpl`对象，并赋给了成员变量`mWindowManager`. `WindowManagerImpl.createLocalWindowManager`的实现如下：

		public WindowManagerImpl createLocalWindowManager(Window parentWindow) {
		        return new WindowManagerImpl(mDisplay, parentWindow);
		}

  可以看到这里用到了`WindowManagerImpl`中需要两个参数的构造函数，第二个参数是对应的`Window`.

再次回到`Activity.attach`方法。第5973行将`Window`的成员变量`mWindowManager`指向的`WindowManagerImpl`对象赋给了`Activity`的成员变量`mWindowManager`. 至此，我们就知道了通过`Activity.getSystemService(Context.WINDOW_SERVICE);`拿到的是一个`WindowManager`的实现类`WindowManagerImpl`的对象。对于非`Activity`的`Context`，调用它们的`getSystemService`方法实际上会调用`ContextImpl.getSystemService`. 这个方法返回的也是一个`WindowManagerImpl`的对象。只不过这个`WindowManagerImpl`的对象的成员变量`mParentWindow`为`null`，也就是没有关联任何`Window`对象。

### `WindowManager.addView`做了什么

接下来探究一下`WindowManager.addView`到底做了什么。

通过上面的分析，我们已经知道实现`WindowManager`这个接口的是`WindowManagerImpl`类，因此直接看一下`WindowManagerImpl.addView`的实现：

	@Override
	public void addView(@NonNull View view, 
	@NonNull ViewGroup.LayoutParams params) {
	    applyDefaultToken(params);
	    mGlobal.addView(view, params, mDisplay, mParentWindow);
	}

函数就只有两句话，我们看关键的第二句。这里的`mGlobal`就是之前提到过的`WindowManagerGlobal`对象。它是一个单例，也就意味着在一个应用进程里，虽然每一个`Activity`会对应不同的`WindowManagerImpl`对象，但是它们的视图是由唯一的一个`WindowManagerGlobal`对象统一管理。

在看`WindowManagerGlobal.addView`的实现之前，有必要先说明一下`WindowManagerGlobal`中有三个比较重要的`ArrayList`:

	private final ArrayList<View> mViews = new ArrayList<View>();
	private final ArrayList<ViewRootImpl> mRoots = new ArrayList<ViewRootImpl>();
	private final ArrayList<WindowManager.LayoutParams> mParams = new ArrayList<WindowManager.LayoutParams>();

这三个`ArrayList`分别保存了`View`, `ViewRootImpl`和`WindowManager.LayoutParams`.

接下来就具体分析一下`WindowManagerGlobal.addView`.

	204    public void addView(View view, ViewGroup.LayoutParams params,
	205            Display display, Window parentWindow) {
	206        if (view == null) {
	207            throw new IllegalArgumentException("view must not be null");
	208        }
	209        if (display == null) {
	210            throw new IllegalArgumentException("display must not be null");
	211        }
	212        if (!(params instanceof WindowManager.LayoutParams)) {
	213            throw new IllegalArgumentException("Params must be WindowManager.LayoutParams");
	214        }
	215
	216        final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams)params;
	217        if (parentWindow != null) {
	218            parentWindow.adjustLayoutParamsForSubWindow(wparams);
	219        } else {
	220            // If there's no parent and we're running on L or above (or in the
	221            // system context), assume we want hardware acceleration.
	222            final Context context = view.getContext();
	223            if (context != null
	224                    && context.getApplicationInfo().targetSdkVersion >= Build.VERSION_CODES.LOLLIPOP) {
	225                wparams.flags |= WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED;
	226            }
	227        }
	228
	229        ViewRootImpl root;
	230        View panelParentView = null;
	231
	232        synchronized (mLock) {
	233            // Start watching for system property changes.
	234            if (mSystemPropertyUpdater == null) {
	235                mSystemPropertyUpdater = new Runnable() {
	236                    @Override public void More ...run() {
	237                        synchronized (mLock) {
	238                            for (int i = mRoots.size() - 1; i >= 0; --i) {
	239                                mRoots.get(i).loadSystemProperties();
	240                            }
	241                        }
	242                    }
	243                };
	244                SystemProperties.addChangeCallback(mSystemPropertyUpdater);
	245            }
	246
	247            int index = findViewLocked(view, false);
	248            if (index >= 0) {
	249                if (mDyingViews.contains(view)) {
	250                    // Don't wait for MSG_DIE to make it's way through root's queue.
	251                    mRoots.get(index).doDie();
	252                } else {
	253                    throw new IllegalStateException("View " + view
	254                            + " has already been added to the window manager.");
	255                }
	256                // The previous removeView() had not completed executing. Now it has.
	257            }
	258
	259            // If this is a panel window, then find the window it is being
	260            // attached to for future reference.
	261            if (wparams.type >= WindowManager.LayoutParams.FIRST_SUB_WINDOW &&
	262                    wparams.type <= WindowManager.LayoutParams.LAST_SUB_WINDOW) {
	263                final int count = mViews.size();
	264                for (int i = 0; i < count; i++) {
	265                    if (mRoots.get(i).mWindow.asBinder() == wparams.token) {
	266                        panelParentView = mViews.get(i);
	267                    }
	268                }
	269            }
	270
	271            root = new ViewRootImpl(view.getContext(), display);
	272
	273            view.setLayoutParams(wparams);
	274
	275            mViews.add(view);
	276            mRoots.add(root);
	277            mParams.add(wparams);
	278        }
	279
	280        // do this last because it fires off messages to start doing things
	281        try {
	282            root.setView(view, wparams, panelParentView);
	283        } catch (RuntimeException e) {
	284            // BadTokenException or InvalidDisplayException, clean up.
	285            synchronized (mLock) {
	286                final int index = findViewLocked(view, false);
	287                if (index >= 0) {
	288                    removeViewLocked(index, true);
	289                }
	290            }
	291            throw e;
	292        }
	293    }

206-214行是对参数的正确性检查。

然后217行有一个`if`语句：如果`parentWindow`不为`null`则调用其`adjustLayoutParamsForSubWindow`方法。这里的`parentWindow`实际上指向的是`WindowManagerImpl`的成员变量`mParentWindow`. 如果我们仍然假定当前的`Context`是一个`Activity`, 那么`parentWindow`就非空。因此`Window.adjustLayoutParamsForSubWindow`被调用：

	551     void adjustLayoutParamsForSubWindow(WindowManager.LayoutParams wp) {
	552         CharSequence curTitle = wp.getTitle();
	553         if (wp.type >= WindowManager.LayoutParams.FIRST_SUB_WINDOW &&
	554             wp.type <= WindowManager.LayoutParams.LAST_SUB_WINDOW) {
	                ...
	581         } else {
	582             if (wp.token == null) {
	583                 wp.token = mContainer == null ? mAppToken : mContainer.mAppToken;
	584             }
	585             if ((curTitle == null || curTitle.length() == 0)
	586                     && mAppName != null) {
	587                 wp.setTitle(mAppName);
	588             }
	589         }
	            ...
	596     }

这里最关键的是第583行，将`Window`里保存的`Activity`的标识`mAppToken`赋给了传进来的`WindowManager.LayoutParams`的`token`. 由于这个`WindowManager.LayoutParams`最后会传给`WindowManagerService`, 因此`WindowManagerService`可以通过这个`token`知道是哪个`Activity`要添加视图。

回到`WindowManagerGlobal.addView`, 248-257行会对要添加的`view`进行重复添加的检查。如果发现是重复添加则抛出异常。

如果当前要添加的`view`从属于某个之前已经添加的`view`，261-269行就会去找出那个已经添加的`view`.

接着在271行，一个`ViewRootImpl`对象被创建出来。`ViewRootImpl`在应用进程这边管理视图的过程中担任了重要的角色。像`Activity`对应的视图的measure, layout, draw等过程都是从`ViewRootImpl`开始。另外`ViewRootImpl`还负责应用进程和`WindowManagerService`进程之间的通信。

275-277行将`view`, `root`和`wparams`分别加入各自对应的`ArrayList`. `View`, `ViewRootImpl`和`WindowManager.LayoutParams`这三者是通过`ArrayList`的下标一一对应的。

最后282行调用了`ViewRootImpl.setView`, 在这个方法中`ViewRootImpl`会去通知`WindowManagerService`将新的视图添加到屏幕上。

### 总结

这篇文章所讲的内容可以浓缩成下面这张图。

![WindowManager]({{ site.url }}/lib/images/windowmanager.png)

一个`Activity`会有一个成员变量指向一个`WindowManager`的具体实现类`WindowManagerImpl`对象。该`WindowManagerImpl`对象会保存该`Activity`对应的一个`PhoneWindow`对象的引用。一个`PhoneWindow`又会引用一个`DecorView`对象。在一个应用进程中，会有一个`WindowManagerGlobal`的单例管理应用中所有Activity对应的视图。`WindowManagerGlobal`中有三个关键的`ArrayList`，分别保存了`View`, `ViewRootImpl`和`WindowManager.LayoutParams`.

上文所讲的这些内容都是发生在应用进程里的事情。在系统里真正负责视图管理的是`WindowManagerService`进程。其中，`ViewRootImpl`担任了沟通应用进程和`WindowManagerService`进程的角色。
