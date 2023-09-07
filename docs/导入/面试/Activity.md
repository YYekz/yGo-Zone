ActivtyThread -> main

ActivityThread thread = new ActivityThread();
thread.attach();

ActivtyThread -> attach()
1. 绑定ApplicationThread
IActivityManager mgr = ActivityManager.getService();
mgr.attachApplication(mAppThread, startSeq);(mAppThread在ActivityThread成员变量中初始化创建)
2. 垃圾回收线程注册
BinderInternal.addGcWatcher({
	ActivityTaskManager.getService().releaseSomeActivities(mAppThread);
});
3. 注册了全局绘制变化监听
ViewRootImpl.addConfigCallback(configChangedCallback);

AMS -> attachApplication() -> attachApplicationLocked() -> thread.bindApplication()

ActivityThread.ApplicationThread -> bindApplication() -> ActivityThread.sendMessage(H.BIND_APPLICATION, data); -> H.handleBindApplication()
-> LoadApk.makeApplication()
1. Application到这里创建完毕；
	1. mInstrumentation.newApplication();
	2. instrumentation.callApplicationOnCreate(app);
2. ContentProvider也初始化完毕；
	1. installContentProviders
3. 预加载字体；
	1. data.info.getResources().preloadFonts(preloadedFontsResource);


ClientTransaction

`requestLayout` 调用之后，调用了 `mWindowSession.addToDisplay` 方法，来完成最终的 `Window` 的添加过程。

在上面代码中，`mWindowSession` 的类型是 `IWindowSession`，他是一个 Binder 对象，真正的实现是 `Session`，也就是 Window 的添加过程是一次 IPC 调用。
在 `Session` 内部会通过 `WindoweManagerServer` 来实现 Window 的添加，如此一来，Window 的添加过程就交给了 WindowManagerServer 去处理。`WMS` 会为其分配 `Surface`，确定窗口显示的次序，最终通过 `SurfaceFlinger` 将这些 `Surface` 绘制到屏幕上。



ViewRootImpl.requestLayout
```java
@Override  
public void requestLayout() {  
    if (!mHandlingLayoutInLayoutRequest) {  
        checkThread();  
        mLayoutRequested = true;  
        scheduleTraversals();  
    }  
}

void scheduleTraversals() {  
    if (!mTraversalScheduled) {  
        mTraversalScheduled = true;  
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();  
        mChoreographer.postCallback(  
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);  
        notifyRendererOfFramePending();  
        pokeDrawLockIfNeeded();  
    }  
}

Choreographer.doCallbacks -> TraversalRunnable.doTraversal

void doTraversal() {  
    if (mTraversalScheduled) {  
        mTraversalScheduled = false;  
        mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier); 
		...
        performTraversals();  
		...
    }  
}

private void performTraversals(){
	...
	performMeasure();
	performLayout();
	performDraw();

}

private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {  
    if (mView == null) {  
        return;  
    }  
    Trace.traceBegin(Trace.TRACE_TAG_VIEW, "measure");  
    try {  
        mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);  
    } finally {  
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);  
    }  
}

mView.measure() -> mView.onMeasure() layout、draw同理

```


### invalidate

调用 View.invalidate() 方法后会逐级往上调用父 View 的相关方法，最终在 Choreographer 的控制下调用 ViewRootImpl.performTraversals() 方法。由于 mLayoutRequested == false，因此只有满足 `mFirst || windowShouldResize || insetsChanged || viewVisibilityChanged || params != null ...` 等条件才会执行 measure 和 layout 流程，否则只执行 draw 流程，draw 流程的执行过程与是否开启硬件加速有关：

-   关闭硬件加速则从 DecorView 开始往下的所有子 View 都会被重新绘制。
-   开启硬件加速则只有调用 invalidate 方法的 View 才会重新绘制。

View 在绘制后会设置 PFLAG_DRAWN 标志位。

### requestLayout

调用 View.requestLayout 方法后会依次调用 performMeasure, performLayout 和 performDraw 方法，调用者 View 及其父 View 会从上往下重新进行 measure, layout 流程，一般情况下不会执行 draw 流程(子 View 会通过判断其尺寸/顶点是否发生改变而决定是否重新 measure/layout/draw 流程)。

[比较一下requestLayout和invalidate方法 - 掘金](https://juejin.cn/post/6904518722564653070#heading-8)


[View的工作原理](https://mp.weixin.qq.com/s?__biz=MzU2NjgwNjc0NQ==&mid=2247483738&idx=1&sn=c47a06198c8eaefd5b72e5bba72750d7&chksm=fca790eccbd019fa0415da12d7fd89c440f9e82697a110db977b2656454a93c543294facda92&scene=21#wechat_redirect)

## 屏幕显示原理

三个阶段
0. 单buff缓冲，会造成画面撕裂。
> 双buffer缓冲区：Frame buffer展示数据，Back buffer供准备下一帧数据。
1. Android4.1 之前，双缓存但没有Vsync（垂直同步），缓存会在Vsync脉冲时交换，但是CPU/GPU绘制是随机的。
![[Pasted image 20230508144943.png|500]]
1. Android4.1开始，实现了Project Buffer，系统收到Vsync脉冲后，立即开始CPU/GPU计算然后将数据写入buffer，通过`Choreographer`实现(意为 舞蹈编导、编舞者。在这里就是指 对CPU/GPU绘制的指导—— 收到VSync信号 才开始绘制，保证绘制拥有完整的16.6ms，避免绘制的随机性)。
![[Pasted image 20230508145320.png|500]]
3. 三缓存。如果界面过于复杂，导致CPU/GPU处理时长超过了16.6ms。
![[Pasted image 20230508145427.png|500]]
这是新增一个Graphic Buffer缓冲区，可以在back buffer以及frame buffer都被占用的上图情况下，使用第三个buffer完成C帧计算，虽然还是会多现实一次A帧，但是后续显示就比较顺畅。（问题是会多一个Graphic Buffer所占用的内存）
![[Pasted image 20230508145645.png|500]]
## Choreographer
-   业界一般通过Choreographer来监控应用的帧率。
```java
1    //输入事件，首先执行  
2    public static final int CALLBACK_INPUT = 0;  
3    //动画，第二执行  
4    public static final int CALLBACK_ANIMATION = 1;  
5    //插入更新的动画，第三执行  
6    public static final int CALLBACK_INSETS_ANIMATION = 2;  
7    //绘制，第四执行  
8    public static final int CALLBACK_TRAVERSAL = 3;  
9    //提交，最后执行，  
10    public static final int CALLBACK_COMMIT = 4;
```

FrameDisplayEventReceiver：FrameDisplayEventReceiver是 DisplayEventReceiver 的子类，在 DisplayEventReceiver 的构造方法会通过 JNI 创建一个 IDisplayEventConnection 的 VSYNC 的监听者。

[WindowManager、ViewRootImpl、DocerView几个问题的理解 - 简书](https://www.jianshu.com/p/f6b26ea7d868)

[Window、WindowManager](https://mp.weixin.qq.com/s?__biz=MzU2NjgwNjc0NQ==&mid=2247483743&idx=1&sn=3f7590e6ba9cdc20ec1bcda12896f764&chksm=fca790e9cbd019ff72718d53d7d512a72582c80294b75d843ba5d72851a2ef73a811fbb2755f&scene=21#wechat_redirect)

[Android | 理解 Window 和 WindowManager - 掘金](https://juejin.cn/post/7076274407416528909#heading-25)
## Handler

-   postSyncBarrier()返回一个int类型的数值，通过这个数值可以撤销屏障即removeSyncBarrier()。
-   postSyncBarrier()是私有的，如果我们想调用它就得使用反射。插入普通消息会唤醒消息队列，但是插入屏障不会。

## 挂起


## BadToken
1. Activity finish WMS token销毁超时10s
2. Dialog没有使用Activity的context 而是使用Application的Context导致badtoken（将dialog WindowType设置为系统即可）

## 优化


## 相关文章
[Choreographer + Android Vsync](https://mp.weixin.qq.com/s?__biz=MzU2NjgwNjc0NQ==&mid=2247484087&idx=1&sn=d735851482041e09a694a6918b5a8d81&chksm=fca79301cbd01a1718dab4d65425464da34e3ed4446a9f34cd07e332f36549a7fc6ad4ad9561&scene=21#wechat_redirect)