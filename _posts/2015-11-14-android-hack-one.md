---
layout: post
title: Android Hack(1)－－制造crash：java.lang.SecurityException：Unable to find app for caller android.app.ApplicationThreadProxy@xxxxxxxx (pid=xxxx) when getting content provider XXXXXX
---

最近触宝电话的crash日志中出现了如下较多的一个crash：

	java.lang.SecurityException: Unable to find app for caller android.app.ApplicationThreadProxy@xxxxxxxx (pid=xxxx) when getting content provider XXXXXX
	at android.os.Parcel.readException(Parcel.java:1472)
	at android.os.Parcel.readException(Parcel.java:1426)
	at android.app.ActivityManagerProxy.getContentProvider(ActivityManagerNative.java:2865)
	at android.app.ActivityThread.acquireProvider(ActivityThread.java:4445)
	at android.app.ContextImpl$ApplicationContentResolver.acquireProvider(ContextImpl.java:2221)
	at android.content.ContentResolver.acquireProvider(ContentResolver.java:1409)
	at android.provider.Settings$NameValueCache.lazyGetProvider(Settings.java:890)
	at android.provider.Settings$NameValueCache.getStringForUser(Settings.java:937)
	at android.provider.Settings$System.getStringForUser(Settings.java:1142)
	at android.provider.Settings$System.getIntForUser(Settings.java:1212)
	at android.provider.Settings$System.getInt(Settings.java:1207)
	at android.media.AudioManager.querySoundEffectsEnabled(AudioManager.java:1859)
	at android.media.AudioManager.playSoundEffect(AudioManager.java:1811)
	at android.view.ViewRootImpl.playSoundEffect(ViewRootImpl.java:5164)
	at android.view.View.playSoundEffect(View.java:16954)
	at android.view.View.performClick(View.java:4443)
	at android.view.View$PerformClick.run(View.java:18457)
	at android.os.Handler.handleCallback(Handler.java:733)
	at android.os.Handler.dispatchMessage(Handler.java:95)
	at android.os.Looper.loop(Looper.java:136)
	at android.app.ActivityThread.main(ActivityThread.java:5049)
	at java.lang.reflect.Method.invokeNative(Native Method)
	at java.lang.reflect.Method.invoke(Method.java:515)
	at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:793)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:609)
	at dalvik.system.NativeStart.main(Native Method)

<!-- more -->
发生crash的手机系统版本以4.4.4居多。根据stack,程序是在与`ActivityManagerService`通信的时候crash了，而且挂在`ActivityManagerService`的`getContentProvider`中。Ok，查看源码，发现异常是在`getContentProviderImpl`中被抛出：

	7514    private final ContentProviderHolder getContentProviderImpl(IApplicationThread caller,
	7515             String name, IBinder token, boolean stable, int userId) {
	7516         ContentProviderRecord cpr;
	7517         ContentProviderConnection conn = null;
	7518         ProviderInfo cpi = null;
	7519 
	7520         synchronized(this) {
	7521             ProcessRecord r = null;
	7522             if (caller != null) {
	7523                 r = getRecordForAppLocked(caller);
	7524                 if (r == null) {
	7525                     throw new SecurityException(
	7526                             "Unable to find app for caller " + caller
	7527                           + " (pid=" + Binder.getCallingPid()
	7528                           + ") when getting content provider " + name);
	7529                 }
	7530             }

从中可以看出当函数`getRecordForAppLocked`返回`null`时，异常被抛出。`getRecordForAppLocked`是这样查找`ProcessRecord`的：

	3654     final ProcessRecord getRecordForAppLocked(
	3655             IApplicationThread thread) {
	3656         if (thread == null) {
	3657             return null;
	3658         }
	3659 
	3660         int appIndex = getLRURecordIndexForAppLocked(thread);
	3661         return appIndex >= 0 ? mLruProcesses.get(appIndex) : null;
	3662     }

如果传入的参数`thread`为`null`则直接返回`null`。考虑到crash是由点击某个`View`导致，因此`thread`不会为`null`. 所以应该是函数`getLRURecordIndexForAppLocked`返回了小于0的值。继续看`getLRURecordIndexForAppLocked`的实现：

	3642     private final int getLRURecordIndexForAppLocked(IApplicationThread thread) {
	3643         IBinder threadBinder = thread.asBinder();
	3644         // Find the application record.
	3645         for (int i=mLruProcesses.size()-1; i>=0; i--) {
	3646             ProcessRecord rec = mLruProcesses.get(i);
	3647             if (rec.thread != null && rec.thread.asBinder() == threadBinder) {
	3648                 return i;
	3649             }
	3650         }
	3651         return -1;
	3652     }

`ActivityManagerService`用一个LRU列表保存当前与每一个应用关联的所有进程的`ProcessRecord`。而`ProcessRecord`中的成员`thread`是`ActivityManagerService`用来与应用进程通信的`Binder`对象。函数返回-1有两种情况：要么我们的应用进程对应的`ProcessRecord`已经不在LRU列表中；要么进程对应的`ProcessRecord`在列表中但是`thread`改变了。不管是哪种情况，都有些匪夷所思。

解决问题的方法之一是先制造出相同的问题。所以我要尝试在一个demo应用里重现这个crash.

首先在stackoverflow上看看有没有人遇到同样的问题。不出所料，还是有不少人被此crash困扰。[这篇post](http://stackoverflow.com/questions/12691149/when-i-send-hashmap-with-size-more-than-20-in-android-4-1-i-get-crash)对制造这个crash提供了线索。因此我在demo应用的`MainActivity`中加入一个视图`view1`并设置其`onClickListener`如下：

	findViewById(R.id.view1).setOnClickListener(new View.OnClickListener() {
	    @Override
	    public void onClick(View v) {
	        Intent intent = new Intent(MainActivity.this, SecondActivity.class);
	        HashMap<String, Bitmap> map = new HashMap<String, Bitmap>();
	        for (int i = 0; i < 20; i++) {
	            Bitmap bitmap = BitmapFactory.decodeResource(getResources(), R.mipmap.ic_launcher);
	            map.put(String.valueOf(i), bitmap);
	        }
	        intent.putExtra("test", map);
	        startActivity(intent);
	    }
	});

运行程序，在点击`view1`后程序因为`TransactionTooLargeException`而闪退。不过这时一个很有思议的现象出现了，原先的进程并没有被杀死而且一个具有同样包名的新进程产生了。使用命令

	adb shell cat /proc/(pid)/stat

发现两个进程都处于活跃状态。既然一开始的进程并没有被杀死，我想如果在原先的进程中一直查询数据库，在点击`view1`之后会发生什么呢？于是在`MainActivity`启动之后，我开启一个线程不停地查询数据库：

	new Thread() {
            public void run() {
                while (true) {
                    try {
                        Cursor c = getContentResolver().query(ContactsContract.Contacts.CONTENT_URI, null, null, null, null);
                        c.close();
                        Thread.sleep(500);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        }.start();

此时点击`view1`，Bingo！标题中的异常在原先的进程中被抛出！

知其然也要知其所以然。接下来就回归源码，一探究竟吧。

当点击`view1`启动`SecondActivity`时，`TransactionTooLargeException`被抛出，其函数调用栈如下:

	android.os.TransactionTooLargeException
	at android.os.BinderProxy.transactNative(Native Method)
	at android.os.BinderProxy.transact(Binder.java:496)
	at android.app.ApplicationThreadProxy.scheduleLaunchActivity(ApplicationThreadNative.java:797)
	at com.android.server.am.ActivityStackSupervisor.realStartActivityLocked(ActivityStackSupervisor.java:1181)
	at com.android.server.am.ActivityStackSupervisor.startSpecificActivityLocked(ActivityStackSupervisor.java:1281)
	at com.android.server.am.ActivityStack.resumeTopActivityInnerLocked(ActivityStack.java:1891)
	at com.android.server.am.ActivityStack.resumeTopActivityLocked(ActivityStack.java:1455)
	at com.android.server.am.ActivityStackSupervisor.resumeTopActivitiesLocked(ActivityStackSupervisor.java:2473)
	at com.android.server.am.ActivityStack.completePauseLocked(ActivityStack.java:998)
	at com.android.server.am.ActivityStack.activityPausedLocked(ActivityStack.java:896)
	at com.android.server.am.ActivityManagerService.activityPaused(ActivityManagerService.java:6311)
	at android.app.ActivityManagerNative.onTransact(ActivityManagerNative.java:512)
	at com.android.server.am.ActivityManagerService.onTransact(ActivityManagerService.java:2208)
	at android.os.Binder.execTransact(Binder.java:446)

查看`ActivityStackSupervisor.startSpecificActivityLocked`:

	1045    void startSpecificActivityLocked(ActivityRecord r,
	1046            boolean andResume, boolean checkConfig) {
	1047        // Is this activity's application already running?
	1048        ProcessRecord app = mService.getProcessRecordLocked(r.processName,
	1049                r.info.applicationInfo.uid, true);
	1050
	1051        r.task.stack.setLaunchTime(r);
	1052
	1053        if (app != null && app.thread != null) {
	1054            try {
	1055                if ((r.info.flags&ActivityInfo.FLAG_MULTIPROCESS) == 0
	1056                        || !"android".equals(r.info.packageName)) {
	1057                    // Don't add this if it is a platform component that is marked
	1058                    // to run in multiple processes, because this is actually
	1059                    // part of the framework so doesn't make sense to track as a
	1060                    // separate apk in the process.
	1061                    app.addPackage(r.info.packageName, mService.mProcessStats);
	1062                }
	1063                realStartActivityLocked(r, app, andResume, checkConfig);
	1064                return;
	1065            } catch (RemoteException e) {
	1066                Slog.w(TAG, "Exception when starting activity "
	1067                        + r.intent.getComponent().flattenToShortString(), e);
	1068            }
	1069
	1070            // If a dead object exception was thrown -- fall through to
	1071            // restart the application.
	1072        }
	1073
	1074        mService.startProcessLocked(r.processName, r.info.applicationInfo, true, 0,
	1075                "activity", r.intent.getComponent(), false, false, true);
	1076    }

由于执行第1063行的`realStartActivityLocked`时发生`TransactionTooLargeException`，代码继续执行1074行调用`ActivityManagerService.startProcessLocked`.

	2576     final ProcessRecord startProcessLocked(String processName,
	2577             ApplicationInfo info, boolean knownToBeDead, int intentFlags,
	2578             String hostingType, ComponentName hostingName, boolean allowWhileBooting,
	2579             boolean isolated, boolean keepIfLarge) {
	2580         ProcessRecord app;
	2581         if (!isolated) {
	2582             app = getProcessRecordLocked(processName, info.uid, keepIfLarge);
	2583         } else {
	2584             // If this is an isolated process, it can't re-use an existing process.
	2585             app = null;
	2586         }
	......        
	2597         if (app != null && app.pid > 0) {
	2598             if (!knownToBeDead || app.thread == null) {
	2599                 // We already have the app running, or are waiting for it to
	2600                 // come up (we have a pid but not yet its thread), so keep it.
	2601                 if (DEBUG_PROCESSES) Slog.v(TAG, "App already running: " + app);
	2602                 // If this is a new package in the process, add the package to the list
	2603                 app.addPackage(info.packageName, mProcessStats);
	2604                 return app;
	2605             }
	2606 
	2607             // An application record is attached to a previous process,
	2608             // clean it up now.
	2609             if (DEBUG_PROCESSES || DEBUG_CLEANUP) Slog.v(TAG, "App died: " + app);
	2610             handleAppDiedLocked(app, true, true);
	2611         }
	2612 
	2613         String hostingNameStr = hostingName != null
	2614                 ? hostingName.flattenToShortString() : null;
	2615 
	......
	2645         if (app == null) {
	2646             app = newProcessRecordLocked(info, processName, isolated);
	2647             if (app == null) {
	2648                 Slog.w(TAG, "Failed making new process record for "
	2649                         + processName + "/" + info.uid + " isolated=" + isolated);
	2650                 return null;
	2651             }
	2652             mProcessNames.put(processName, app.uid, app);
	2653             if (isolated) {
	2654                 mIsolatedProcesses.put(app.uid, app);
	2655             }
	2656         } else {
	2657             // If this is a new package in the process, add the package to the list
	2658             app.addPackage(info.packageName, mProcessStats);
	2659         }
	2660 
	2661         // If the system is not ready yet, then hold off on starting this
	2662         // process until it is.
	2663         if (!mProcessesReady
	2664                 && !isAllowedWhileBooting(info)
	2665                 && !allowWhileBooting) {
	2666             if (!mProcessesOnHold.contains(app)) {
	2667                 mProcessesOnHold.add(app);
	2668             }
	2669             if (DEBUG_PROCESSES) Slog.v(TAG, "System not ready, putting on hold: " + app);
	2670             return app;
	2671         }
	2672 
	2673         startProcessLocked(app, hostingType, hostingNameStr);
	2674         return (app.pid != 0) ? app : null;
	2675     }

通过2582行得到的`app`非空，程序进入2597行的if语句。由于传入的`knownToBeDead`为`true`，下面的`handleAppDiedLocked`被调用。

	3604     private final void handleAppDiedLocked(ProcessRecord app,
	3605             boolean restarting, boolean allowRestart) {
	3606         cleanUpApplicationRecordLocked(app, restarting, allowRestart, -1);
	3607         if (!restarting) {
	3608             removeLruProcessLocked(app);
	3609         }
	3610 
	3611         if (mProfileProc == app) {
	3612             clearProfilerLocked();
	3613         }
	3614 
	3615         // Remove this application's activities from active lists.
	3616         boolean hasVisibleActivities = mStackSupervisor.handleAppDiedLocked(app);
	3617 
	3618         app.activities.clear();
	3619 
	3620         if (app.instrumentationClass != null) {
	3621             Slog.w(TAG, "Crash of app " + app.processName
	3622                   + " running instrumentation " + app.instrumentationClass);
	3623             Bundle info = new Bundle();
	3624             info.putString("shortMsg", "Process crashed.");
	3625             finishInstrumentationLocked(app, Activity.RESULT_CANCELED, info);
	3626         }
	3627 
	3628         if (!restarting) {
	3629             if (!mStackSupervisor.resumeTopActivitiesLocked()) {
	3630                 // If there was nothing to resume, and we are not already
	3631                 // restarting this process, but there is a visible activity that
	3632                 // is hosted by the process...  then make sure all visible
	3633                 // activities are running, taking care of restarting this
	3634                 // process.
	3635                 if (hasVisibleActivities) {
	3636                     mStackSupervisor.ensureActivitiesVisibleLocked(null, 0);
	3637                 }
	3638             }
	3639         }
	3640     }

由于传入的`restarting`为`true`, 此时`handleAppDiedLocked`主要做的是清除之前进程中存在的`Activity`。

继续`startProcessLocked`.函数最后会调用`startProcessLocked(app, hostingType, hostingNameStr);`.

	2681     private final void startProcessLocked(ProcessRecord app,
	2682             String hostingType, String hostingNameStr) {
	......
	2767             // Start the process.  It will either succeed and return a result containing
	2768             // the PID of the new process, or else throw a RuntimeException.
	2769             Process.ProcessStartResult startResult = Process.start("android.app.ActivityThread",
	2770                     app.processName, uid, uid, gids, debugFlags, mountExternal,
	2771                     app.info.targetSdkVersion, app.info.seinfo, null);
	......
	2813             app.setPid(startResult.pid);
	2814             app.usingWrapper = startResult.usingWrapper;
	2815             app.removed = false;
	2816             synchronized (mPidsSelfLocked) {
	2817                 this.mPidsSelfLocked.put(startResult.pid, app);
	2818                 Message msg = mHandler.obtainMessage(PROC_START_TIMEOUT_MSG);
	2819                 msg.obj = app;
	2820                 mHandler.sendMessageDelayed(msg, startResult.usingWrapper
	2821                         ? PROC_START_TIMEOUT_WITH_WRAPPER : PROC_START_TIMEOUT);
	2822             }
	2823         } catch (RuntimeException e) {
	2824             // XXX do better error recovery.
	2825             app.setPid(0);
	2826             Slog.e(TAG, "Failure starting process " + app.processName, e);
	2827         }
	2828     }

可以看到，`startProcessLocked(app, hostingType, hostingNameStr);`会重新创建一个进程并且将`app`对应的`pid`换成新的进程id。新的进程创建后会执行`ActivityThread`的`main`函数，其中会调用`ActivityManagerService.attachApplicationLocked`来通知`ActivityManagerService`进程创建成功：

	4844     private final boolean attachApplicationLocked(IApplicationThread thread,
	4845             int pid) {
	4846 
	4847         // Find the application record that is being attached...  either via
	4848         // the pid if we are running in multiple processes, or just pull the
	4849         // next app record if we are emulating process with anonymous threads.
	4850         ProcessRecord app;
	4851         if (pid != MY_PID && pid >= 0) {
	4852             synchronized (mPidsSelfLocked) {
	4853                 app = mPidsSelfLocked.get(pid);
	4854             }
	4855         } else {
	4856             app = null;
	4857         }
	......
	4900         app.makeActive(thread, mProcessStats);
	4901         app.curAdj = app.setAdj = -100;
	4902         app.curSchedGroup = app.setSchedGroup = Process.THREAD_GROUP_DEFAULT;
	4903         app.forcingToForeground = null;
	4904         app.foregroundServices = false;
	4905         app.hasShownUi = false;
	4906         app.debugging = false;
	4907         app.cached = false;
	4908 
	4909         mHandler.removeMessages(PROC_START_TIMEOUT_MSG, app);
	4910 
	4911         boolean normalMode = mProcessesReady || isAllowedWhileBooting(app.info);
	......
	4997         boolean badApp = false;
	4998         boolean didSomething = false;
	4999 
	5000         // See if the top visible activity is waiting to run in this process...
	5001         if (normalMode) {
	5002             try {
	5003                 if (mStackSupervisor.attachApplicationLocked(app, mHeadless)) {
	5004                     didSomething = true;
	5005                 }
	5006             } catch (Exception e) {
	5007                 badApp = true;
	5008             }
	5009         }
	5010 
	......
	5044         if (badApp) {
	5045             // todo: Also need to kill application to deal with all
	5046             // kinds of exceptions.
	5047             handleAppDiedLocked(app, false, true);
	5048             return false;
	5049         }
	5050 
	5051         if (!didSomething) {
	5052             updateOomAdjLocked();
	5053         }
	5054 
	5055         return true;
	5056     }

`attachApplicationLocked`对`ProcessRecord`中的一些成员变量赋值后调用`ActivityStackSupervisor.attachApplicationLocked`:

	369     boolean attachApplicationLocked(ProcessRecord app, boolean headless) throws Exception {
	370         boolean didSomething = false;
	371         final String processName = app.processName;
	372         for (int stackNdx = mStacks.size() - 1; stackNdx >= 0; --stackNdx) {
	373             final ActivityStack stack = mStacks.get(stackNdx);
	374             if (!isFrontStack(stack)) {
	375                 continue;
	376             }
	377             ActivityRecord hr = stack.topRunningActivityLocked(null);
	378             if (hr != null) {
	379                 if (hr.app == null && app.uid == hr.info.applicationInfo.uid
	380                         && processName.equals(hr.processName)) {
	381                     try {
	382                         if (headless) {
	383                             Slog.e(TAG, "Starting activities not supported on headless device: "
	384                                     + hr);
	385                         } else if (realStartActivityLocked(hr, app, true, true)) {
	386                             didSomething = true;
	387                         }
	388                     } catch (Exception e) {
	389                         Slog.w(TAG, "Exception in new application when starting activity "
	390                               + hr.intent.getComponent().flattenToShortString(), e);
	391                         throw e;
	392                     }
	393                 }
	394             }
	395         }
	396         if (!didSomething) {
	397             ensureActivitiesVisibleLocked(null, 0);
	398         }
	399         return didSomething;
	400     }

此时在385行再次调用`realStartActivityLocked`，又会抛出`TransactionTooLargeException`. 因此回到`ActivityManagerService.attachApplicationLocked`的5007行，`badApp`被置为`true`. 接下来`handleAppDiedLocked`再次被调用，不同于上一次，这次第二个参数为`false`. 因此在`handleAppDiedLocked`的3608行执行`removeLruProcessLocked`将进程对应的`ProcessRecord`从**LRU列表中移除**！至此标题中的crash被抛出的原因也就找到了：`ActivityManagerService`将程序进程对应的`ProcessRecord`从LRU列表中移除了，但心慈手软没有杀死进程。

最后需要说明这个trick在5.0以上的系统是不起作用的。回到`ActivityManagerService.attachApplicationLocked`的5045行，那里有一段注释

	// todo: Also need to kill application to deal with all
	// kinds of exceptions.

于是Google说到做到，5.0以上的系统便多了这么一句：

	app.kill("error during init", true);

终于结束了！有人说何必这么麻烦，一句话的事：

	throw new SecurityException("Unable to find app for caller...");

嗯，有道理！
