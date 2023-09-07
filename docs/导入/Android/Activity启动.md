# App启动
> 本文基于Android 11(android-30)源码版本。

Activity是我们Android开发最常用的组件，是页面的基础，是四大组件的之一，让我们一起来看看Activity相关的问题。

## 相关问题


1. 打开一个App代码执行流程是什么样的？
2. Application是什么时候初始化的？
3. 四大组件是在什么阶段初始化的？
4. 如何打开一个新Activity？第一个Activity是怎么打开的？`activity.startActivity()`到底做了什么？
5. 未注册的Activity在何时校验是否注册在清单文件Manifest中？
6. Activity关闭的延时10s问题遇到过吗？为什么会出现这样的情况？

带着这些问题，一起来看一下Activity相关的源码吧。

## 开啃

我们都知道在Android也是运行JVM上，那一定会有一个主类（拥有main方法），去作为第一个加载的类。在Android中这个第一个加载的主类就是 ActivityThread。所以一切的开始从它的`main`开始。
```java
public static void main(String[] args) {
        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ActivityThreadMain");

        AndroidOs.install();
        CloseGuard.setEnabled(false);
        Environment.initForCurrentUser();

        final File configDir = Environment.getUserConfigDirectory(UserHandle.myUserId());
        TrustedCertificateStore.setDefaultUserDirectory(configDir);

        initializeMainlineModules();

        Process.setArgV0("<pre-initialized>");
        
        //1
        Looper.prepareMainLooper();

        long startSeq = 0;
        if (args != null) {
            for (int i = args.length - 1; i >= 0; --i) {
                if (args[i] != null && args[i].startsWith(PROC_START_SEQ_IDENT)) {
                    startSeq = Long.parseLong(
                            args[i].substring(PROC_START_SEQ_IDENT.length()));
                }
            }
        }
        //2
        ActivityThread thread = new ActivityThread();
        thread.attach(false, startSeq);

        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }

        if (false) {
            Looper.myLooper().setMessageLogging(new
                    LogPrinter(Log.DEBUG, "ActivityThread"));
        }

        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
        //3
        Looper.loop();

        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
```
这是app运行的第一个应用层方法（一行没省略），可以看到这个方法中除了一些Trace信息、系统方法以外，主要为两个作用：

### 主线程的Handler初始化
注释1和注释3的`prepareMainLooper()`和`loop`方法调用，开启了主线程Handler的消息处理，这里也解释了Android一个经典的问题：**为什么只有主线程的Handler发送消息时不需要Looper的prepare和loop方法调用呢？** 其实这里的代码也就回答了这个问题，不是不需要，而是系统替我们做掉了这个操作。主线程的Handler不仅消化我们应用内部分发的主线程消息任务，也会处理系统内部分发的绘制等众多任务。Handler相关问题我们这里不细讲，默认大家都懂。

### ActivityThread的创建及绑定
从上面截图中可以看到，先创建了 ActivityThread，在创建时，会同时创建 ApplicationThread。
> ApplicationThread其实是一个ActivityThread的实现binder接口的内部类，主要功能其实就是在系统Service和内部ActivityThraed之间做垮进程逻辑中转，通过调用ActivityThread中的Handler(mH)将逻辑分发到ActivityThread中。

`attach`传了false，我们重点关注方法中false的逻辑。
```java
    //ActivityThread.java
    ActivityThread() {
        mResourcesManager = ResourcesManager.getInstance();
    }
   
   private void attach(boolean system, long startSeq) {
        sCurrentActivityThread = this;
        mSystemThread = system;
        sCurrentActivityThread = this;
        mSystemThread = system;
        if (!system) {
            android.ddm.DdmHandleAppName.setAppName("<pre-initialized>",
                                                    UserHandle.myUserId());
            RuntimeInit.setApplicationObject(mAppThread.asBinder());
                //1
            final IActivityManager mgr = ActivityManager.getService();
            try {
                mgr.attachApplication(mAppThread, startSeq);
            } catch (RemoteException ex) {
                throw ex.rethrowFromSystemServer();
            }
                //2
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
                            ActivityTaskManager.getService().releaseSomeActivities(mAppThread);
                        } catch (RemoteException e) {
                            throw e.rethrowFromSystemServer();
                        }
                    }
                }
            });
        }else{
            ...
        }    
        ...
```
先看注释1处，引入了一个Android中非常关键的类 ActivityManagerService。调用了`attachApplication`方法，最终调到了这里。
> ActivityManagerService（AMS）：核心Service。activity的创建、activity之间的关系、资源管理等功能类。

```java
thread.bindApplication(processName, appInfo, providerList, null, profilerInfo,
                        null, null, null, testMode,
                        mBinderTransactionTrackingEnabled, enableTrackAllocation,
                        isRestrictedBackupMode || !normalMode, app.isPersistent(),
                        new Configuration(app.getWindowProcessController().getConfiguration()),
                        app.compat, getCommonServicesLocked(app.isolated),
                        mCoreSettingsObserver.getCoreSettingsLocked(),
                        buildSerial, autofillOptions, contentCaptureOptions,
                        app.mDisabledCompatChanges);
```
通过`ApplicationThread.bindApplication`回传回ActivityThread的内部类ApplicationThread中，通过`sendMessage(H.BIND_APPLICATION, data);`发送handler给ActivityThread中处理该事件，最终调到了`handleBindApplication`中。
```java
private void handleBindApplication(AppBindData data) {
    ...
    if (ii != null) {
            ApplicationInfo instrApp;
            try {
                instrApp = getPackageManager().getApplicationInfo(ii.packageName, 0,
                        UserHandle.myUserId());
            } catch (RemoteException e) {
                instrApp = null;
            }
            if (instrApp == null) {
                instrApp = new ApplicationInfo();
            }
            ii.copyTo(instrApp);
            instrApp.initForUser(UserHandle.myUserId());
            final LoadedApk pi = getPackageInfo(instrApp, data.compatInfo,
                    appContext.getClassLoader(), false, true, false);

            final ContextImpl instrContext = ContextImpl.createAppContext(this, pi,
                    appContext.getOpPackageName());

            try {
                final ClassLoader cl = instrContext.getClassLoader();
                mInstrumentation = (Instrumentation)
                    cl.loadClass(data.instrumentationName.getClassName()).newInstance();
            } catch (Exception e) {
                throw new RuntimeException(
                    "Unable to instantiate instrumentation "
                    + data.instrumentationName + ": " + e.toString(), e);
            }

            final ComponentName component = new ComponentName(ii.packageName, ii.name);
            mInstrumentation.init(this, instrContext, appContext, component,
                    data.instrumentationWatcher, data.instrumentationUiAutomationConnection);

            if (mProfiler.profileFile != null && !ii.handleProfiling
                    && mProfiler.profileFd == null) {
                mProfiler.handlingProfiling = true;
                final File file = new File(mProfiler.profileFile);
                file.getParentFile().mkdirs();
                Debug.startMethodTracing(file.toString(), 8 * 1024 * 1024);
            }
        } else {
            mInstrumentation = new Instrumentation();
            mInstrumentation.basicInit(this);
        }
        ...
         Application app;
        final StrictMode.ThreadPolicy savedPolicy = StrictMode.allowThreadDiskWrites();
        final StrictMode.ThreadPolicy writesAllowedPolicy = StrictMode.getThreadPolicy();
        try {

            app = data.info.makeApplication(data.restrictedBackupMode, null);

            app.setAutofillOptions(data.autofillOptions);
            app.setContentCaptureOptions(data.contentCaptureOptions);

            mInitialApplication = app;

            if (!data.restrictedBackupMode) {
                if (!ArrayUtils.isEmpty(data.providers)) {
                    installContentProviders(app, data.providers);
                }
            }
            try {
                mInstrumentation.onCreate(data.instrumentationArgs);
            }
            catch (Exception e) {
                throw new RuntimeException(
                    "Exception thrown in onCreate() of "
                    + data.instrumentationName + ": " + e.toString(), e);
            }
            try {
                mInstrumentation.callApplicationOnCreate(app);
            } catch (Exception e) {
                if (!mInstrumentation.onException(app, e)) {
                    throw new RuntimeException(
                      "Unable to create application " + app.getClass().getName()
                      + ": " + e.toString(), e);
                }
            }
            
        FontsContract.setApplicationContextForResources(appContext);
        if (!Process.isIsolated()) {
            try {
                final ApplicationInfo info =
                        getPackageManager().getApplicationInfo(
                                data.appInfo.packageName,
                                PackageManager.GET_META_DATA /*flags*/,
                                UserHandle.myUserId());
                if (info.metaData != null) {
                    final int preloadedFontsResource = info.metaData.getInt(
                            ApplicationInfo.METADATA_PRELOADED_FONTS, 0);
                    if (preloadedFontsResource != 0) {
                        data.info.getResources().preloadFonts(preloadedFontsResource);
                    }
                }
            } catch (RemoteException e) {
                throw e.rethrowFromSystemServer();
            }
        }
```
这个方法比较长，做的主要是app初始化信息相关操作，主要有几步：
1. ContextImpl的创建，这里的ContextImpl引用其实就是平时我们使用的ApplicationContext；
2. Instrumentation的创建；
3. Application的创建；
4. 初始化ContentProvider，可以看到ContentProvider的初始化是在Application初始化过程中就已经初始化了的；
5. 通过刚刚创建的Instrumentation的`callApplicationOnCreate`调用Application的`onCreate`方法；
6. 提前加载字体资源文件；

> Instrumentation可以把测试包和目标测试应用加载到同一个进程中运行，既然各个控件和测试代码都运行在同一个进程中了，测试代码当然就可以调用这些控件的方法了，同时修改和验证这些控件的一些数据。其实就是一个代理类，方便测试。

到这一步Application的创建就完成了，其实也没有很复杂，就是一些数据的初始化。来一张时序图大致描述下方法调用栈。

### App运行入口以及Application调用时序图
![Application的启动](Application的启动.png)

### Activity的创建
那Application初始化完毕了，第一个Activity是什么时候创建的呢？他和之后的平时我们使用`startActivity`创建Activity的流程有什么区别吗？

其实区别很小，对于Activity的创建流程没有任何变化，不管是第几个Activity创建初始化的流程都相同，只是**触发创建Activity的入口流程不同。** 
#### 首个Activity的创建
第一个Activity的启动入口在哪里呢？其实是在AMS的`systemReady`方法中。当系统初始化准备完毕，会调用该方法。
```java
public void systemReady(final Runnable goingCallback, @NonNull TimingsTraceAndSlog t) {
    if (bootingSystemUser) {
                t.traceBegin("sendUserStartBroadcast");
                final int callingUid = Binder.getCallingUid();
                final int callingPid = Binder.getCallingPid();
                long ident = Binder.clearCallingIdentity();
                try {
                    Intent intent = new Intent(Intent.ACTION_USER_STARTED);
                    intent.addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY
                            | Intent.FLAG_RECEIVER_FOREGROUND);
                    intent.putExtra(Intent.EXTRA_USER_HANDLE, currentUserId);
                    //1
                    broadcastIntentLocked(null, null, null, intent,
                            null, null, 0, null, null, null, OP_NONE,
                            null, false, false, MY_PID, SYSTEM_UID, callingUid, callingPid,
                            currentUserId);
                    intent = new Intent(Intent.ACTION_USER_STARTING);
                    intent.addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY);
                    intent.putExtra(Intent.EXTRA_USER_HANDLE, currentUserId);
                    broadcastIntentLocked(null, null, null, intent, null,
                            new IIntentReceiver.Stub() {
                                @Override
                                public void performReceive(Intent intent, int resultCode,
                                        String data, Bundle extras, boolean ordered, boolean sticky,
                                        int sendingUser) {}
                            }, 0, null, null, new String[] {INTERACT_ACROSS_USERS}, OP_NONE, null,
                            true, false, MY_PID, SYSTEM_UID, callingUid, callingPid,
                            UserHandle.USER_ALL);
                } catch (Throwable e) {
                    Slog.wtf(TAG, "Failed sending first user broadcasts", e);
                } finally {
                    Binder.restoreCallingIdentity(ident);
                }
                t.traceEnd();
            } else {
                Slog.i(TAG, "Not sending multi-user broadcasts for non-system user "
                        + currentUserId);
            }

            t.traceBegin("resumeTopActivities");
                //2
            mAtmInternal.resumeTopActivities(false /* scheduleIdle */);
            t.traceEnd();

            if (bootingSystemUser) {
                t.traceBegin("sendUserSwitchBroadcasts");
                mUserController.sendUserSwitchBroadcasts(-1, currentUserId);
                t.traceEnd();
            }
        }
```
代码较长，主要是做数据初始化，以及两个主要的操作：
1. 初始化对应的Broadcast；
2. 调用ActivityTaskManagerService的`resumeTopActivities`开始走Activity的创建流程（接下来与其他Activity的创建流程相同）。
> ActivityTaskManagerService （ATMS）是一个Android11新增的类，将原来AMS中所有对Activity管理、状态的方法都剥离成为一个独立的类，将逻辑解耦。
#### 时序图
![第一个Activity启动](第一个Activity启动.png)

#### 其他Activity的创建
看完了系统启动第一个Activity的触发创建流程，看一下其他Activity的创建触发流程。也就是Android中经典问题：**startActivity和startActivityForResult有区别吗？**
```java
public void startActivity(Intent intent, @Nullable Bundle options) {
        ...
        if (options != null) {
            startActivityForResult(intent, -1, options);
        } else {
            // Note we want to go through this call for compatibility with
            // applications that may have overridden the method.
            startActivityForResult(intent, -1);
        }
    }
```
其实只要稍微点一下Activity的源码就可以回答这个问题，`startActivity`和`startActivityForResult`没有区别，最终都是调用`startActivityForResult`。
```java
public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
            @Nullable Bundle options) {
        if (mParent == null) {
            options = transferSpringboardActivityOptions(options);
            //1
            Instrumentation.ActivityResult ar =
                mInstrumentation.execStartActivity(
                    this, mMainThread.getApplicationThread(), mToken, this,
                    intent, requestCode, options);
            if (ar != null) {
                //2
                mMainThread.sendActivityResult(
                    mToken, mEmbeddedID, requestCode, ar.getResultCode(),
                    ar.getResultData());
            }
            if (requestCode >= 0) {
                mStartedActivity = true;
            }

            cancelInputsAndStartExitTransition(options);
        } else {
            if (options != null) {
                mParent.startActivityFromChild(this, intent, requestCode, options);
            } else {
                mParent.startActivityFromChild(this, intent, requestCode);
            }
        }
    }
```
可以看到最终触发的是Instrumentation的`execStartActivity`，拿到结果发送给ActivityThread做逻辑处理，主要是调用Activity的`onResume`方法以及触发`dispatchActivityResult`逻辑，也就是我们平时使用的`onActivityResult`，这个逻辑比较简单感兴趣可以查看源码这里就不细说了。言归正传。
```java
public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {
        IApplicationThread whoThread = (IApplicationThread) contextThread;
        Uri referrer = target != null ? target.onProvideReferrer() : null;
        if (referrer != null) {
            intent.putExtra(Intent.EXTRA_REFERRER, referrer);
        }
        try {
            intent.migrateExtraStreamToClipData(who);
            intent.prepareToLeaveProcess(who);
            //1
            int result = ActivityTaskManager.getService().startActivity(whoThread,
                    who.getBasePackageName(), who.getAttributionTag(), intent,
                    intent.resolveTypeIfNeeded(who.getContentResolver()), token,
                    target != null ? target.mEmbeddedID : null, requestCode, 0, null, options);
                //2
            checkStartActivityResult(result, intent);
        } catch (RemoteException e) {
            throw new RuntimeException("Failure from system", e);
        }
        return null;
    }
```
可以看到使用了ATMS的`startActivity`去创建Activity，并拿到回调结果展示对应结果。在ATMS中最终回调的方法是`startActivityAsUser`。
```java
private int startActivityAsUser(IApplicationThread caller, String callingPackage,
            @Nullable String callingFeatureId, Intent intent, String resolvedType,
            IBinder resultTo, String resultWho, int requestCode, int startFlags,
            ProfilerInfo profilerInfo, Bundle bOptions, int userId, boolean validateIncomingUser) {
        return getActivityStartController().obtainStarter(intent, "startActivityAsUser")
                .setCaller(caller)
                .setCallingPackage(callingPackage)
                .setCallingFeatureId(callingFeatureId)
                .setResolvedType(resolvedType)
                .setResultTo(resultTo)
                .setResultWho(resultWho)
                .setRequestCode(requestCode)
                .setStartFlags(startFlags)
                .setProfilerInfo(profilerInfo)
                .setActivityOptions(bOptions)
                .setUserId(userId)
                .execute();
    }
    
        private int executeRequest(Request request) {
        //省略非常多行的合法性校验
        ...
        //1
        final ActivityRecord r = new ActivityRecord(mService, callerApp, callingPid, callingUid,
                callingPackage, callingFeatureId, intent, resolvedType, aInfo,
                mService.getGlobalConfiguration(), resultRecord, resultWho, requestCode,
                request.componentSpecified, voiceSession != null, mSupervisor, checkedOptions,
                sourceRecord);
        mLastStartActivityRecord = r;

        if (r.appTimeTracker == null && sourceRecord != null) {
                    r.appTimeTracker = sourceRecord.appTimeTracker;
        }

        final ActivityStack stack = mRootWindowContainer.getTopDisplayFocusedStack();
        if (voiceSession == null && stack != null && (stack.getResumedActivity() == null
                || stack.getResumedActivity().info.applicationInfo.uid != realCallingUid)) {
            if (!mService.checkAppSwitchAllowedLocked(callingPid, callingUid,
                    realCallingPid, realCallingUid, "Activity start")) {
                if (!(restrictedBgActivity && handleBackgroundActivityAbort(r))) {
                    mController.addPendingActivityLaunch(new PendingActivityLaunch(r,
                            sourceRecord, startFlags, stack, callerApp, intentGrants));
                }
                ActivityOptions.abort(checkedOptions);
                return ActivityManager.START_SWITCHES_CANCELED;
            }
        }

        mService.onStartActivitySetDidAppSwitch();
        mController.doPendingActivityLaunches(false);
        //2
        mLastStartActivityResult = startActivityUnchecked(r, sourceRecord, voiceSession,
                request.voiceInteractor, startFlags, true /* doResume */, checkedOptions, inTask,
                restrictedBgActivity, intentGrants);

        if (request.outActivity != null) {
            request.outActivity[0] = mLastStartActivityRecord;
        }

        return mLastStartActivityResult;
    }
```
该方法最终会调用ActivityStarter的`execute`方法，在方法中主要触发内部核心方法`executeRequest`，该方法非常长，代码就不贴了，想看可以查看对应类方法的源码，主要功能其实就是**检查合法性**，也就是我们上文说的拿到非法结果则抛出对应Exception。当然，若一切检查正常，则创建ActivityRecord，并继续下一步。
> ActivityRecord 见名知意，是Activity的相关信息的集合类，存储Activity生命周期内所有信息。需要注意的是这是在Service端的信息，Activity注明的BadToken和Activity延迟10s关闭也是和这个相关的，在其他文章中细说。

最终通过`startActivityUnchecked` 调用`startActivityInner`。
```java
int startActivityInner(final ActivityRecord r, ActivityRecord sourceRecord,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            int startFlags, boolean doResume, ActivityOptions options, Task inTask,
            boolean restrictedBgActivity, NeededUriGrants intentGrants) {
        setInitialState(r, options, inTask, doResume, startFlags, sourceRecord, voiceSession,
                voiceInteractor, restrictedBgActivity);
            //删掉很多行
                ...
        final ActivityStack topStack = mRootWindowContainer.getTopDisplayFocusedStack();
        if (topStack != null) {
            //1
            startResult = deliverToCurrentTopIfNeeded(topStack, intentGrants);
            if (startResult != START_SUCCESS) {
                return startResult;
            }
        }

        if (mTargetStack == null) {
            mTargetStack = getLaunchStack(mStartActivity, mLaunchFlags, targetTask, mOptions);
        }
        if (newTask) {
            final Task taskToAffiliate = (mLaunchTaskBehind && mSourceRecord != null)
                    ? mSourceRecord.getTask() : null;
            setNewTask(taskToAffiliate);
            if (mService.getLockTaskController().isLockTaskModeViolation(
                    mStartActivity.getTask())) {
                Slog.e(TAG, "Attempted Lock Task Mode violation mStartActivity=" + mStartActivity);
                return START_RETURN_LOCK_TASK_MODE_VIOLATION;
            }
        } else if (mAddingToTask) {
            addOrReparentStartingActivity(targetTask, "adding to task");
        }

        if (mDoResume) {
            final ActivityRecord topTaskActivity =
                    mStartActivity.getTask().topRunningActivityLocked();
            if (!mTargetStack.isTopActivityFocusable()
                    || (topTaskActivity != null && topTaskActivity.isTaskOverlay()
                    && mStartActivity != topTaskActivity)) {
                mTargetStack.ensureActivitiesVisible(null /* starting */,
                        0 /* configChanges */, !PRESERVE_WINDOWS);
                mTargetStack.getDisplay().mDisplayContent.executeAppTransition();
            } else {
                if (mTargetStack.isTopActivityFocusable()
                        && !mRootWindowContainer.isTopDisplayFocusedStack(mTargetStack)) {
                    mTargetStack.moveToFront("startActivityInner");
                }
                //2
                mRootWindowContainer.resumeFocusedStacksTopActivities(
                        mTargetStack, mStartActivity, mOptions);
            }
        }
        mRootWindowContainer.updateUserStack(mStartActivity.mUserId, mTargetStack);

        mSupervisor.mRecentTasks.add(mStartActivity.getTask());
        mSupervisor.handleNonResizableTaskIfNeeded(mStartActivity.getTask(),
                mPreferredWindowingMode, mPreferredTaskDisplayArea, mTargetStack);

        return START_SUCCESS;
    }
```
这段代码也是非常长，同样非常重要，主要还是关注注释的两步，第一步`deliverToCurrentTopIfNeeded`方法其实就是根据启动Acitvity的启动模式去判断是否打开新Activity还是回调当前Activity的`onNewIntent`；第二步是打开Activity的操作，也就是说从这里开始，**上文说的第一个启动和其他启动的Activity流程开始统一**。
```java
boolean resumeFocusedStacksTopActivities(
            ActivityStack targetStack, ActivityRecord target, ActivityOptions targetOptions) {
        for (int displayNdx = getChildCount() - 1; displayNdx >= 0; --displayNdx) {
            boolean resumedOnDisplay = false;
            final DisplayContent display = getChildAt(displayNdx);
            for (int tdaNdx = display.getTaskDisplayAreaCount() - 1; tdaNdx >= 0; --tdaNdx) {
                final TaskDisplayArea taskDisplayArea = display.getTaskDisplayAreaAt(tdaNdx);
                for (int sNdx = taskDisplayArea.getStackCount() - 1; sNdx >= 0; --sNdx) {
                    final ActivityStack stack = taskDisplayArea.getStackAt(sNdx);
                    final ActivityRecord topRunningActivity = stack.topRunningActivity();
                    if (!stack.isFocusableAndVisible() || topRunningActivity == null) {
                        continue;
                    }
                    if (stack == targetStack) {
                        resumedOnDisplay |= result;
                        continue;
                    }
                    if (taskDisplayArea.isTopStack(stack) && topRunningActivity.isState(RESUMED)) {
                        stack.executeAppTransition(targetOptions);
                    } else {
                        resumedOnDisplay |= topRunningActivity.makeActiveIfNeeded(target);
                    }
                }
            }
            if (!resumedOnDisplay) {
                final ActivityStack focusedStack = display.getFocusedStack();
                if (focusedStack != null) {
                    //这里
                    result |= focusedStack.resumeTopActivityUncheckedLocked(target, targetOptions);
                } else if (targetStack == null) {
                    result |= resumeHomeActivity(null /* prev */, "no-focusable-task",
                            display.getDefaultTaskDisplayArea());
                }
            }
        }

        return result;
    }
```
可以看到调用`resumeTopActivityUncheckedLocked`后会调用`resumeTopActivityInnerLocked`后最终会调用` mStackSupervisor.startSpecificActivity(next, true, true);`。
```java
boolean realStartActivityLocked(ActivityRecord r, WindowProcessController proc,
            boolean andResume, boolean checkConfig) throws RemoteException {
                    
            //省略
            ...
                final ClientTransaction clientTransaction = ClientTransaction.obtain(
                        proc.getThread(), r.appToken);

                final DisplayContent dc = r.getDisplay().mDisplayContent;
                //1
                clientTransaction.addCallback(LaunchActivityItem.obtain(new Intent(r.intent),
                        System.identityHashCode(r), r.info,
                        mergedConfiguration.getGlobalConfiguration(),
                        mergedConfiguration.getOverrideConfiguration(), r.compat,
                        r.launchedFromPackage, task.voiceInteractor, proc.getReportedProcState(),
                        r.getSavedState(), r.getPersistentSavedState(), results, newIntents,
                        dc.isNextTransitionForward(), proc.createProfilerInfoIfNeeded(),
                        r.assistToken, r.createFixedRotationAdjustmentsIfNeeded()));

                // Set desired final state.
                final ActivityLifecycleItem lifecycleItem;
                if (andResume) {
                //2
                    lifecycleItem = ResumeActivityItem.obtain(dc.isNextTransitionForward());
                } else {
                    lifecycleItem = PauseActivityItem.obtain();
                }
                clientTransaction.setLifecycleStateRequest(lifecycleItem);

                mService.getLifecycleManager().scheduleTransaction(clientTransaction);

                if ((proc.mInfo.privateFlags & ApplicationInfo.PRIVATE_FLAG_CANT_SAVE_STATE) != 0
                        && mService.mHasHeavyWeightFeature) {
                    if (proc.mName.equals(proc.mInfo.packageName)) {
                        if (mService.mHeavyWeightProcess != null
                                && mService.mHeavyWeightProcess != proc) {
                            Slog.w(TAG, "Starting new heavy weight process " + proc
                                    + " when already running "
                                    + mService.mHeavyWeightProcess);
                        }
                        mService.setHeavyWeightProcess(r);
                    }
                }

            } catch (RemoteException e) {
                if (r.launchFailed) {
                    Slog.e(TAG, "Second failure launching "
                            + r.intent.getComponent().flattenToShortString() + ", giving up", e);
                    proc.appDied("2nd-crash");
                    r.finishIfPossible("2nd-crash", false /* oomAdj */);
                    return false;
                }

                r.launchFailed = true;
                proc.removeActivity(r);
                throw e;
            }
        } finally {
            endDeferResume();
        }
        return true;
    }
```
可以看到最终其实是调用LaunchActivityItem的`execute`方法，其实Activity的相关步骤都是通过类似这样的ClientTransactionItem子类去做逻辑隔离的。同时注意看注释2，当Start执行完后，会通过ResumeActivityItem的`execute`方法，回调给ActivityThread的对应方法触发`activity.onResume()`逻辑。
```java
    @Override
    public void execute(ClientTransactionHandler client, IBinder token,
            PendingTransactionActions pendingActions) {
        Trace.traceBegin(TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
        ActivityClientRecord r = new ActivityClientRecord(token, mIntent, mIdent, mInfo,
                mOverrideConfig, mCompatInfo, mReferrer, mVoiceInteractor, mState, mPersistentState,
                mPendingResults, mPendingNewIntents, mIsForward,
                mProfilerInfo, client, mAssistToken, mFixedRotationAdjustments);
        client.handleLaunchActivity(r, pendingActions, null /* customIntent */);
        Trace.traceEnd(TRACE_TAG_ACTIVITY_MANAGER);
    }
```
可以看到这里调用了`client.handleLaunchActivity`，那client是什么呢？其实就是ActivityThread。
```java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        ActivityInfo aInfo = r.activityInfo;
        if (r.packageInfo == null) {
            r.packageInfo = getPackageInfo(aInfo.applicationInfo, r.compatInfo,
                    Context.CONTEXT_INCLUDE_CODE);
        }

        ComponentName component = r.intent.getComponent();
        if (component == null) {
            component = r.intent.resolveActivity(
                mInitialApplication.getPackageManager());
            r.intent.setComponent(component);
        }

        if (r.activityInfo.targetActivity != null) {
            component = new ComponentName(r.activityInfo.packageName,
                    r.activityInfo.targetActivity);
        }
        //1
        ContextImpl appContext = createBaseContextForActivity(r);
        Activity activity = null;
        try {
            java.lang.ClassLoader cl = appContext.getClassLoader();
            //2
            activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
            StrictMode.incrementExpectedActivityCount(activity.getClass());
            r.intent.setExtrasClassLoader(cl);
            r.intent.prepareToEnterProcess();
            if (r.state != null) {
                r.state.setClassLoader(cl);
            }
        } catch (Exception e) {
            if (!mInstrumentation.onException(activity, e)) {
                throw new RuntimeException(
                    "Unable to instantiate activity " + component
                    + ": " + e.toString(), e);
            }
        }

        try {
            Application app = r.packageInfo.makeApplication(false, mInstrumentation);
            if (activity != null) {
                CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
                Configuration config = new Configuration(mCompatConfiguration);
                if (r.overrideConfig != null) {
                    config.updateFrom(r.overrideConfig);
                }
                if (DEBUG_CONFIGURATION) Slog.v(TAG, "Launching activity "
                        + r.activityInfo.name + " with config " + config);
                Window window = null;
                if (r.mPendingRemoveWindow != null && r.mPreserveWindow) {
                    window = r.mPendingRemoveWindow;
                    r.mPendingRemoveWindow = null;
                    r.mPendingRemoveWindowManager = null;
                }
                appContext.getResources().addLoaders(
                        app.getResources().getLoaders().toArray(new ResourcesLoader[0]));

                appContext.setOuterContext(activity);
                //3
                activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window, r.configCallback,
                        r.assistToken);

                if (customIntent != null) {
                    activity.mIntent = customIntent;
                }
                r.lastNonConfigurationInstances = null;
                checkAndBlockForNetworkAccess();
                activity.mStartedActivity = false;
                int theme = r.activityInfo.getThemeResource();
                if (theme != 0) {
                    activity.setTheme(theme);
                }

                activity.mCalled = false;
                //4
                if (r.isPersistable()) {
                    mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
                } else {
                    mInstrumentation.callActivityOnCreate(activity, r.state);
                }
                if (!activity.mCalled) {
                    throw new SuperNotCalledException(
                        "Activity " + r.intent.getComponent().toShortString() +
                        " did not call through to super.onCreate()");
                }
         }
            r.setState(ON_CREATE);

        return activity;
    }
```
这是最核心的创建Acitvity的方法，主要的作用是：
1. 创建Activity对应的Context上下文；
2. 通过Instrumentation创建Activity对象；
3. 为activity设置对应的必要信息；
4. 回调activity的`onCreate`方法；

至此，一个新的activity创建完成啦。到这一步就会和Window相关了，我们下一篇细说。洋洋洒洒这么多，还是上一个调用时序方便大家看。
#### 时序图
![Activity启动](Activity启动.png)



### 相关文章
https://blog.csdn.net/a19891024/article/details/54342799