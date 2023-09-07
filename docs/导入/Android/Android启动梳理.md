# BadTokenException引发的思考
最近在crash中发现了一个每个Android开发都或多或少遇到过的“老朋友”——`BadTokenException`，决定仔细看看到底是为啥，才出现了这篇文章。

> 本文基于android-30源码编写。

## 思考
1. Token何时创建？何时销毁？是什么？
2. 为什么会出现BadTokenException？
3. Activity为什么会延迟10s才会触发onDestroy？

## 拆解源码
让我们带着问题一起来看一下源码。

首先看下`Token`
```java
static class Token extends IApplicationToken.Stub {
        private WeakReference<ActivityRecord> weakActivity;
        private final String name;
        private final String tokenString;
        
        //1. 创建
        Token(Intent intent) {
            name = intent.getComponent().flattenToShortString();
            tokenString = "Token{" + Integer.toHexString(System.identityHashCode(this)) + "}";
        }
        
        //2. 绑定
        private void attach(ActivityRecord activity) {
            if (weakActivity != null) {
                throw new IllegalStateException("Already attached..." + this);
            }
            weakActivity = new WeakReference<>(activity);
        }
        
        //3. 获取
        private static @Nullable ActivityRecord tokenToActivityRecordLocked(Token token) {
            if (token == null) {
                return null;
            }
            ActivityRecord r = token.weakActivity.get();
            if (r == null || r.getRootTask() == null) {
                return null;
            }
            return r;
        }
        
        @Override
        public String getName() {
            return name;
        }
}
```
    
首先`Token`是`ActivityRecord`的静态内部类，可以看到Token是一个Binder，继承了`IApplicationToken.Stub`，一起来看下。
```java
/** {@hide} */
interface IApplicationToken
{
  String getName();
}
```
这不是我们的重点，我们继续看`Token`的创建时机，是在`ActivityRecord`的构造方法中创建。
```java
ActivityRecord(ActivityTaskManagerService _service, WindowProcessController _caller,
            int _launchedFromPid, int _launchedFromUid, String _launchedFromPackage,
            @Nullable String _launchedFromFeature, Intent _intent, String _resolvedType,
            ActivityInfo aInfo, Configuration _configuration, ActivityRecord _resultTo,
            String _resultWho, int _reqCode, boolean _componentSpecified,
            boolean _rootVoiceInteraction, ActivityStackSupervisor supervisor,
            ActivityOptions options, ActivityRecord sourceRecord) {
        super(_service.mWindowManager, new Token(_intent).asBinder(), TYPE_APPLICATION, true,
                null /* displayContent */, false /* ownerCanManageAppTokens */);
        ...
        appToken.attach(this);           
    }
```
那问题中**Token何时创建？** 已经有了答案，**在ActivityRecord的构造方法中创建**。
> ActivityRecord：ActivityRecord是每个Activity在AMS中的描述，保存了所有的activity信息，包括启动模式，包名，进程名等等。以后细说。

在构造方法创建过`Token`之后，同样在构造方法中会执行`attach`方法，将当前`ActivityRecord`与`Token`绑定，在当前`Token`中保存对应`ActivityRecord`的弱引用。
`Token`中的`tokenToActivityRecordLocked`方法会被频繁调用，是通过`Token`获取对应`ActivityRecord`的功能。`ActivityRecord`封装`forTokenLocked`方法对外部调用。
```java
    static @Nullable ActivityRecord forTokenLocked(IBinder token) {
        try {
            return Token.tokenToActivityRecordLocked((Token)token);
        } catch (ClassCastException e) {
            Slog.w(TAG, "Bad activity token: " + token, e);
            return null;
        }
    }
```
经过这一系列的分析我们明白了，`Token`是在`ActivityRecord`创建时创建，而`ActivityRecord`是在`Activity`创建时保存在AMS中的描述，所以**Token是什么？**
**Token是Activity在AMS上的一个唯一标识，通过binder跨进程标识在AMS上和在本地对象的同一性。**

> Binder可以跨进程通信可以简单的理解为framework是我们的Server服务端，而上层代码则为Client本地。

那最后一个问题来了，token是什么时候销毁的呢？
#### 调用时序图
![Activity的结束](Activity的结束.png)

看下`ActivityThread`中的关键方法：
```java
 @Override
    public void handleDestroyActivity(IBinder token, boolean finishing, int configChanges,
            boolean getNonConfigInstance, String reason) {
        ActivityClientRecord r = performDestroyActivity(token, finishing,
                configChanges, getNonConfigInstance, reason);
        if (r != null) {
            cleanUpPendingRemoveWindows(r, finishing);
            WindowManager wm = r.activity.getWindowManager();
            View v = r.activity.mDecor;
            if (v != null) {
                if (r.activity.mVisibleFromServer) {
                    mNumVisibleActivities--;
                }
                IBinder wtoken = v.getWindowToken();
                if (r.activity.mWindowAdded) {
                    if (r.mPreserveWindow) {
                        r.mPendingRemoveWindow = r.window;
                        r.mPendingRemoveWindowManager = wm;
                        r.window.clearContentView();
                    } else {
                        wm.removeViewImmediate(v);
                    }
                }
                if (wtoken != null && r.mPendingRemoveWindow == null) {
                    WindowManagerGlobal.getInstance().closeAll(wtoken,
                            r.activity.getClass().getName(), "Activity");
                } else if (r.mPendingRemoveWindow != null) {
                    WindowManagerGlobal.getInstance().closeAllExceptView(token, v,
                            r.activity.getClass().getName(), "Activity");
                }
                r.activity.mDecor = null;
            }
            if (r.mPendingRemoveWindow == null) {
                WindowManagerGlobal.getInstance().closeAll(token,
                        r.activity.getClass().getName(), "Activity");
            }
            Context c = r.activity.getBaseContext();
            if (c instanceof ContextImpl) {
                ((ContextImpl) c).scheduleFinalCleanup(
                        r.activity.getClass().getName(), "Activity");
            }
        }
        if (finishing) {
            try {
                ActivityTaskManager.getService().activityDestroyed(token);
            } catch (RemoteException ex) {
                throw ex.rethrowFromSystemServer();
            }
        }
        mSomeActivitiesChanged = true;
    }
```
可以看到在最后Activity退出时

### 相关文章
[Activity中的token](https://www.jianshu.com/p/1422c02da9da)