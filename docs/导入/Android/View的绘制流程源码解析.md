
> 本文源码基于Android sdk 30，代码会只保留和本文相关内容。

不知道大家有没有好奇过，我们经常说Activity的根视图是DecorView，那DecorView是什么时候创建的、谁创建的呢？它和Activity的关系是什么？我们遇到的BadTokenException跟他有关系吗？
今天我们一起来看看Activity的window的前世今生。
## 起源
我们都知道Activity是AMS创建的，会通过ActivityThread中的mH回调过来，最终创建方法在ActivityThread的 `handleLaunchActivity`方法中`performLaunchActivity`方法创建的。这个方法触发了创建PhoneWindow、DecorView和WindowManagerImpl，接下来就一起看看整个流程是什么样的。

我们都知道，Activity的创建打开会流转到`handleResumeActivity`方法中，一起看看这个方法做了什么。
```java
@Override  
public void handleResumeActivity(ActivityClientRecord r, boolean finalStateRequest,  
        boolean isForward, String reason) {  
    ...
    if (r.window == null && !a.mFinished && willBeVisible) { 
	    //1. phoneWindow 
        r.window = r.activity.getWindow();  
        //2. docorView
        View decor = r.window.getDecorView();  
		...
        ViewManager wm = a.getWindowManager();  
        //获取相关属性
        WindowManager.LayoutParams l = r.window.getAttributes();  
        ...
        if (a.mVisibleFromClient) {  
            if (!a.mWindowAdded) {  
                a.mWindowAdded = true;  
                //3. 将view添加
                wm.addView(decor, l);  
            } else {               
                a.onWindowAttributesChanged(l);  
            }  
        }  
    }   
    ...
}
```
这个方法我们主要关注四个注释中标注的点，首先看下window的来源。
```java
//activity
    public Window getWindow() {
        return mWindow;
    }
    
final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config, String referrer, IVoiceInteractor voiceInteractor,
            Window window, ActivityConfigCallback activityConfigCallback, IBinder assistToken,
            IBinder shareableActivityToken) {
        attachBaseContext(context);
        mFragments.attachHost(null /*parent*/);
        //这里
        mWindow = new PhoneWindow(this, window, activityConfigCallback);
        mWindow.setWindowControllerCallback(mWindowControllerCallback);
        mWindow.setCallback(this);
        mWindow.setOnWindowDismissedCallback(this);
        mWindow.getLayoutInflater().setPrivateFactory(this);
        if (info.softInputMode != WindowManager.LayoutParams.SOFT_INPUT_STATE_UNSPECIFIED) {
            mWindow.setSoftInputMode(info.softInputMode);
        }
        ...
        }
    }
```
所以最终发现这里的这个window就是我们平时说的PhoneWindow了。从第二个注释也可以发现，DecorView就是PhoneWindow类内部的一个引用对象，一起看看他的调用创建逻辑。
```java
//PhoneWindow
    @Override
    public final @NonNull View getDecorView() {
        if (mDecor == null || mForceDecorInstall) {
            installDecor();
        }
        return mDecor;
    }
    
    private void installDecor() {
        mForceDecorInstall = false;
        if (mDecor == null) {
            mDecor = generateDecor(-1);
        } else {
            mDecor.setWindow(this);
        }
        if (mContentParent == null) {
            mContentParent = generateLayout(mDecor);
        }
        ...
    ...
    }
    
    
    protected DecorView generateDecor(int featureId) {
        Context context;
        if (mUseDecorContext) {
            Context applicationContext = getContext().getApplicationContext();
            if (applicationContext == null) {
                context = getContext();
            } else {
                context = new DecorContext(applicationContext, this);
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
从上面的代码可以看出，在第一次获取DecorView时，会通过`installDecor`的`generateDecor`初始化一个decorView对象，并通过`generateLayout`方法根据他的配置参数初始化不同的layout xml。再通过`r.window.getAttributes()`获取对应的参数，来到了最终的方法，`wm.addView(decor, l)`，activity中的 wm来自于 `attach`方法。
```java
final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config, String referrer, IVoiceInteractor voiceInteractor,
            Window window, ActivityConfigCallback activityConfigCallback, IBinder assistToken,
            IBinder shareableActivityToken) {
        ...
        mWindow.setWindowManager(
                (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
                mToken, mComponent.flattenToString(),
                (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
        mWindowManager = mWindow.getWindowManager();
        mCurrentConfig = config;

        mWindow.setColorMode(info.colorMode);
        mWindow.setPreferMinimalPostProcessing(
                (info.flags & ActivityInfo.FLAG_PREFER_MINIMAL_POST_PROCESSING) != 0);

        setAutofillOptions(application.getAutofillOptions());
        setContentCaptureOptions(application.getContentCaptureOptions());
    }
    
    //Window
    public void setWindowManager(WindowManager wm, IBinder appToken, String appName,
            boolean hardwareAccelerated) {
        mAppToken = appToken;
        mAppName = appName;
        mHardwareAccelerated = hardwareAccelerated;
        if (wm == null) {
            wm = (WindowManager)mContext.getSystemService(Context.WINDOW_SERVICE);
        }
        mWindowManager = ((WindowManagerImpl)wm).createLocalWindowManager(this);
    }
    //WindowManagerImpl
    public WindowManagerImpl createLocalWindowManager(Window parentWindow) {
        return new WindowManagerImpl(mContext, parentWindow);
    }
    
        @Override
    public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
        applyDefaultToken(params);
        mGlobal.addView(view, params, mContext.getDisplayNoVerify(), mParentWindow,
                mContext.getUserId());
    }

```
所以最终`wm.addView`调用的是WindowManagerImpl中`mGlobal.addView`方法，那mGlobal其实就是WindowManagerGlobal，在WindowManagerImpl初始化时获取这个单例。
```java
//WIndowManagerGlobal

public void addView(View view, ViewGroup.LayoutParams params,
            Display display, Window parentWindow, int userId) {
            ...
            //创建ViewRootImpl
            root = new ViewRootImpl(view.getContext(), display);
            view.setLayoutParams(wparams);
            mViews.add(view);
            mRoots.add(root);
            mParams.add(wparams);
            //核心方法，将参数关联
            root.setView(view, wparams, panelParentView, userId);
            ...
    }

//ViewRootImpl
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView,
            int userId) {
        synchronized (this) {
                ...
                // Schedule the first layout -before- adding to the window
                // manager, to make sure we do the relayout before receiving
                // any other events from the system.
                requestLayout();
                ...
                    res = mWindowSession.addToDisplayAsUser(mWindow, mSeq, mWindowAttributes,
                            getHostVisibility(), mDisplay.getDisplayId(), userId, mTmpFrame,
                            mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                            mAttachInfo.mDisplayCutout, inputChannel,
                            mTempInsets, mTempControls);

                ...
                switch (res) {
                        //BadTokenException判断
                }        
    }

```
可以看到通过这个WindowManagerGlobal创建了一个我们View绘图流程中的核心对象ViewRootImpl，接下来调用到了对象中`requestLayout`，这个方法会触发我们今天的主角方法`performTraversals`，所有的测量、布局、绘制方法都是由他来触发的，继续来跟代码。
```java
//ViewRootImpl

    @Override
    public void requestLayout() {
        if (!mHandlingLayoutInLayoutRequest) {
            checkThread();
            mLayoutRequested = true;
            scheduleTraversals();
        }
    }
    //所谓只有主线程能做ui操作的crash判断就来自于这个方法
    void checkThread() {
        if (mThread != Thread.currentThread()) {
            throw new CalledFromWrongThreadException(
                    "Only the original thread that created a view hierarchy can touch its views.");
        }
    }

    @UnsupportedAppUsage
    void scheduleTraversals() {
        if (!mTraversalScheduled) {
            mTraversalScheduled = true;
            //为handler加barrier来保证绘制ui的优先级
            mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
            mChoreographer.postCallback(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
            notifyRendererOfFramePending();
            pokeDrawLockIfNeeded();
        }
    }
    
    final TraversalRunnable mTraversalRunnable = new TraversalRunnable();
    
    final class TraversalRunnable implements Runnable {
        @Override
        public void run() {
            doTraversal();
        }
    }
    
        void doTraversal() {
        if (mTraversalScheduled) {
            mTraversalScheduled = false;
            //移除barrier
            mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);
            //主角登场
            performTraversals();
        }
    }
    
     private void performTraversals() {
     ...
            int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);
            int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);
            performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
            performLayout(lp, mWidth, mHeight);
            performDraw();
     }
```
好了，看到这里大家应该知道了，其实就是对应我们的view的绘制三板斧：measure、layout、draw。
`performMeasure()`：从根节点（DecorView）向下遍历所有的view树，完成所有View和ViewGroup的测量，计算出View和ViewGroup需要显示的宽高以及属性；
`performLayout()`：从根节点（DecorView）向下遍历view树，完成所有View和ViewGroup的布局工作，根据测量出来的宽高以及属性，算出所有View和ViewGroup在屏幕中显示的区域位置；
`performDraw()`：从根节点（DecorView）向下遍历view树，完成所有View和ViewGroup的绘制工作。
大体介绍完毕，那我们就一个个方法跟进去看看大致流程。

### performMeasure所触发的measure流程
在讲measure流程之前，需要先说一个类MeasureSpec。
```java
public static class MeasureSpec {
        private static final int MODE_SHIFT = 30;
        private static final int MODE_MASK  = 0x3 << MODE_SHIFT;

        /** @hide */
        @IntDef({UNSPECIFIED, EXACTLY, AT_MOST})
        @Retention(RetentionPolicy.SOURCE)
        public @interface MeasureSpecMode {}

        /**
         * Measure specification mode: The parent has not imposed any constraint
         * on the child. It can be whatever size it wants.
         */
        public static final int UNSPECIFIED = 0 << MODE_SHIFT;

        /**
         * Measure specification mode: The parent has determined an exact size
         * for the child. The child is going to be given those bounds regardless
         * of how big it wants to be.
         */
        public static final int EXACTLY     = 1 << MODE_SHIFT;

        /**
         * Measure specification mode: The child can be as large as it wants up
         * to the specified size.
         */
        public static final int AT_MOST     = 2 << MODE_SHIFT;
}
```
MeasureSpec是概括了父布局传递给子布局的view布局要求，是一个Int型，高2位用来表示mode，后30位用于表示size，两个参数合并为一个int型对象，可以有效减少对象分配，该类就用于将size和mode装包和拆包，提供获取方法。
总共有三种模式：
1. UNSPECIFIED：未指定尺寸模式。父布局没有对子view强加任何限制。它可以是任意想要的尺寸。
2. EXACTLY：精确模式。父布局决定了子view的精确尺寸。精确尺寸包括确定值比如50dp，100dp，需要注意的是，MATCH_PARENT也是精确的，表示和父布局尺寸一样大。
3. AT_MOST。最大值。子view可以一直大到指定值。也就是WRAP_CONTENT，他的最大值也不会超过父布局的给定约束。


```java
    private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
            mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    }
    
    public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
        boolean optical = isLayoutModeOptical(this);
        if (optical != isLayoutModeOptical(mParent)) {
            Insets insets = getOpticalInsets();
            int oWidth  = insets.left + insets.right;
            int oHeight = insets.top  + insets.bottom;
            widthMeasureSpec  = MeasureSpec.adjust(widthMeasureSpec,  optical ? -oWidth  : oWidth);
            heightMeasureSpec = MeasureSpec.adjust(heightMeasureSpec, optical ? -oHeight : oHeight);
        }
        onMeasure(widthMeasureSpec, heightMeasureSpec);
                
    }
```
从这俩方法可以看出以下信息：
1. 首先可以看到这个方法是一个final方法，说明view的所有子类都不能重写该方法，但官方提供了对应的`onMeasure`方法供自定义使用；
2. mView是什么？其实就是在addView时传入进来的DecorView了，这里其实就是在执行DecorView的`measure()`方法；
3. 方法传入了父布局的宽高MeasureSpec，用于算出当前view的宽高信息（如果自定义的是一个ViewGroup，那则必须重写onMeasure方法，如果自定义的是一个View，则并不是必须重写该方法）。

`onMeasure`是我们的老朋友了，自定义view一定见过。
```java
//View
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
                getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
    }
    
        protected int getSuggestedMinimumWidth() {
        return (mBackground == null) ? mMinWidth : max(mMinWidth, mBackground.getMinimumWidth());
    }
    
        public static int getDefaultSize(int size, int measureSpec) {
        int result = size;
        int specMode = MeasureSpec.getMode(measureSpec);
        int specSize = MeasureSpec.getSize(measureSpec);

        switch (specMode) {
        case MeasureSpec.UNSPECIFIED:
            result = size;
            break;
        case MeasureSpec.AT_MOST:
        case MeasureSpec.EXACTLY:
            result = specSize;
            break;
        }
        return result;
    }
```
函数内部的默认实现方法很简单，就一句代码，但是注释写了很多，有兴趣的可以去看看，这里大概提炼一下注释的意思：
1. 如果重写该方法，**必须**调用`setMeasuredDimension()`来存储该view的宽高，如果不这样做则会触发IllegalStateException，由`measure`方法抛出；
2. 如果重写该方法，子类需要负责确保测量的宽高应至少是该view的最小宽和高。（`getSuggestedMininumHeight`和`getSuggestedMininumWidth`）

一起来看看`setMeasuredDimension`方法。
```java
    protected final void setMeasuredDimension(int measuredWidth, int measuredHeight) {
        boolean optical = isLayoutModeOptical(this);
        if (optical != isLayoutModeOptical(mParent)) {
            Insets insets = getOpticalInsets();
            int opticalWidth  = insets.left + insets.right;
            int opticalHeight = insets.top  + insets.bottom;

            measuredWidth  += optical ? opticalWidth  : -opticalWidth;
            measuredHeight += optical ? opticalHeight : -opticalHeight;
        }
        setMeasuredDimensionRaw(measuredWidth, measuredHeight);
    }

    private void setMeasuredDimensionRaw(int measuredWidth, int measuredHeight) {
        mMeasuredWidth = measuredWidth;
        mMeasuredHeight = measuredHeight;

        mPrivateFlags |= PFLAG_MEASURED_DIMENSION_SET;
    }
```
现在其实就很明确了，其实就是上面注释说的我们需要关注的点，调用`setMeasuredDimension`将测量的值和最小值之间返回一个最大的值来做最终的尺寸。

在DecorView的`onMeasure`方法，就是根据根布局的MeasureSpec获取当前自身的长宽信息，调用到FrameLayout的`onMeasure`方法。
```java

 @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
	    //获取子view的数量
        int count = getChildCount();

        final boolean measureMatchParentChildren =
                MeasureSpec.getMode(widthMeasureSpec) != MeasureSpec.EXACTLY ||
                MeasureSpec.getMode(heightMeasureSpec) != MeasureSpec.EXACTLY;
        mMatchParentChildren.clear();

        int maxHeight = 0;
        int maxWidth = 0;
        int childState = 0;

        for (int i = 0; i < count; i++) {
            final View child = getChildAt(i);
            if (mMeasureAllChildren || child.getVisibility() != GONE) {
	            //遍历自己所有的非GONE的子view，测量他们的尺寸，然后拿到最大尺寸
                measureChildWithMargins(child, widthMeasureSpec, 0, heightMeasureSpec, 0);
                final LayoutParams lp = (LayoutParams) child.getLayoutParams();
                maxWidth = Math.max(maxWidth,
                        child.getMeasuredWidth() + lp.leftMargin + lp.rightMargin);
                maxHeight = Math.max(maxHeight,
                        child.getMeasuredHeight() + lp.topMargin + lp.bottomMargin);
                childState = combineMeasuredStates(childState, child.getMeasuredState());
                if (measureMatchParentChildren) {
                    if (lp.width == LayoutParams.MATCH_PARENT ||
                            lp.height == LayoutParams.MATCH_PARENT) {
                        mMatchParentChildren.add(child);
                    }
                }
            }
        }
		//添加padding
        maxWidth += getPaddingLeftWithForeground() + getPaddingRightWithForeground();
        maxHeight += getPaddingTopWithForeground() + getPaddingBottomWithForeground();

        //再对比一次
        maxHeight = Math.max(maxHeight, getSuggestedMinimumHeight());
        maxWidth = Math.max(maxWidth, getSuggestedMinimumWidth());

        final Drawable drawable = getForeground();
        if (drawable != null) {
            maxHeight = Math.max(maxHeight, drawable.getMinimumHeight());
            maxWidth = Math.max(maxWidth, drawable.getMinimumWidth());
        }

		//设置自己的尺寸
        setMeasuredDimension(resolveSizeAndState(maxWidth, widthMeasureSpec, childState),
                resolveSizeAndState(maxHeight, heightMeasureSpec,
                        childState << MEASURED_HEIGHT_STATE_SHIFT));
        
    }
    
protected void measureChildWithMargins(View child,  
        int parentWidthMeasureSpec, int widthUsed,  
        int parentHeightMeasureSpec, int heightUsed) {  
    final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();  
  
    final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,  
            mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin  
                    + widthUsed, lp.width);  
    final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,  
            mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin  
                    + heightUsed, lp.height);  
  
    child.measure(childWidthMeasureSpec, childHeightMeasureSpec);  
}

```
先看一下`measureChildWithMargins`方法，其实就是根据父布局的限制，以及自身布局的尺寸、margin、padding算出自身可用尺寸。在FrameLayout中，遍历调用了所有子view的这个方法，算出了所有子view的尺寸信息，根据这些信息，设置FrameLayout也就是文中DecorView的尺寸信息。

### performLayout触发的layout流程
现在我们继续回到ViewRootImpl中来看下`performMeasure`方法。
```java
private void performLayout(WindowManager.LayoutParams lp, int desiredWindowWidth,  
        int desiredWindowHeight) {  
        ...  
        final View host = mView;  
		if (host == null) {  
		    return;  
		}
		...
        host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());  
		...
}
```
上文已经说过mView就是DecorView，所以host就是DecorView，但`layout`方法是final的，DecorView是FrameLayout的子类是ViewGroup的子类，所以一起看一下ViewGroup中的layout方法。
```java
//ViewGroup
@Override  
public final void layout(int l, int t, int r, int b) {  
    if (!mSuppressLayout && (mTransition == null || !mTransition.isChangingLayout())) {  
        if (mTransition != null) {  
            mTransition.layoutChange(this);  
        }  
        super.layout(l, t, r, b);  
    } else {  
        // record the fact that we noop'd it; request layout when transition finishes  
        mLayoutCalledWhileSuppressed = true;  
    }  
}

//View
public void layout(int l, int t, int r, int b) {  
	...
    int oldL = mLeft;  
    int oldT = mTop;  
    int oldB = mBottom;  
    int oldR = mRight;  
	 //1
    boolean changed = isLayoutModeOptical(mParent) ?  
            setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);  
  
    if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {  
	    //2
        onLayout(changed, l, t, r, b);  
        ...
}
```
可以看到，ViewGroup中的`layout`方法并没有做什么操作，只是调用父类也就是View的layout方法。这个方法是布局机制中的第二个阶段（第一个阶段是测量），会按照第一阶段的结果对子view进行布局放置。派生类不应该重写这个方法，如果有基于view的派生子类应该重写`onLayout`方法。
回到代码上，这段代码调用了`setOpticalFrame`和`setFrame`方法，其实`setOpticalFrame`方法最终调用还是`setFrame`。
```java
protected boolean setFrame(int left, int top, int right, int bottom) {  
    boolean changed = false;  
    if (mLeft != left || mRight != right || mTop != top || mBottom != bottom) {  
        changed = true;  
  
		int drawn = mPrivateFlags & PFLAG_DRAWN;  
		
        int oldWidth = mRight - mLeft;  
        int oldHeight = mBottom - mTop;  
        int newWidth = right - left;  
        int newHeight = bottom - top;  
        boolean sizeChanged = (newWidth != oldWidth) || (newHeight != oldHeight);  
        
        invalidate(sizeChanged);  
  
        mLeft = left;  
        mTop = top;  
        mRight = right;  
        mBottom = bottom;  
        mRenderNode.setLeftTopRightBottom(mLeft, mTop, mRight, mBottom);  
        mPrivateFlags |= PFLAG_HAS_BOUNDS;
        if (sizeChanged) {  
            sizeChange(newWidth, newHeight, oldWidth, oldHeight);  
        }  
        if ((mViewFlags & VISIBILITY_MASK) == VISIBLE || mGhostView != null) {            
            mPrivateFlags |= PFLAG_DRAWN;  
            invalidate(sizeChanged);   
            invalidateParentCaches();  
        }  
    }  
    return changed;  
}
```
这个方法主要是为了：
1. 给该view分配尺寸和位置，其实实际的布局工作就是在这里完成的；
2. 返回值`changed`，如果新的尺寸和位置和之前有任何不同，都会返回true。
再看会view的`layout`方法，如果`changed`返回true，则会触发`onLayout`，也就是我们ViewGroup的`onLayout`。
```java
@Override  
protected abstract void onLayout(boolean changed,  
        int l, int t, int r, int b);
```
在ViewGroup中发现，`onLayout`方法是一个抽象方法，也就是说只要基于ViewGroup类的容器子类，都必须要重写该方法。事实上也确实如此，layout流程是**必要步骤**。那就直接进入FrameLayout中的`onLayout`看下。
```java
@Override  
protected void onLayout(boolean changed, int left, int top, int right, int bottom) {  
    layoutChildren(left, top, right, bottom, false /* no force left gravity */);  
}  
  
void layoutChildren(int left, int top, int right, int bottom, boolean forceLeftGravity) {  
    final int count = getChildCount();  
  
    final int parentLeft = getPaddingLeftWithForeground();  
    final int parentRight = right - left - getPaddingRightWithForeground();  
  
    final int parentTop = getPaddingTopWithForeground();  
    final int parentBottom = bottom - top - getPaddingBottomWithForeground();  
  
    for (int i = 0; i < count; i++) {  
        final View child = getChildAt(i);  
        if (child.getVisibility() != GONE) {  
            final LayoutParams lp = (LayoutParams) child.getLayoutParams();  
  
            final int width = child.getMeasuredWidth();  
            final int height = child.getMeasuredHeight();  
			  ...
            child.layout(childLeft, childTop, childLeft + width, childTop + height);  
        }  
    }  
}
```
其实FrameLayout方法中`onLayout`直接调用了`layoutChildren`方法，这个方法内部也很简单，遍历所有的非GONE子view（所以INVISIBLE其实还是会占位），获取他的宽高，根据当前的位置算出他的四个点的位置，触发子view的`layout`方法，分发下去，如果是ViewGroup则继续分发，如果是叶子结点的view，则走完布局流程。
> 上面说到layout是必要步骤，但在讲measure时，没有提到必要步骤，为什么呢？
> 从layoutChildren中可以发现，在计算顶点位置时，是根据宽高来算的，宽高获取是通过view的getMeasuredWidth和getMeasuredHeight，这俩方法内部其实就是直接获取view的mMeasuredWidth、mMeasuredHeight，如果能通过其他渠道复制width、height，则measure阶段也可以不需要，当然不建议这么做哈，按照流程实现是最合理的方式。

那我们还是在看下DecorView的`onLayout`方法，其实就是调用FrameLayout的`onLayout`方法。
```java
@Override  
protected void onLayout(boolean changed, int left, int top, int right, int bottom) {  
    super.onLayout(changed, left, top, right, bottom);  
	...
}
```

### performDraw触发的draw流程
老规矩，直接上代码。
```java
private void performDraw() {  
	...
    boolean canUseAsync = draw(fullRedrawNeeded);  
	...
}
private boolean draw(boolean fullRedrawNeeded) {
	...
	if (!drawSoftware(surface, mAttachInfo, xOffset, yOffset,  
	        scalingRequired, dirty, surfaceInsets)) {  
	    return false;  
	}
	...
}

private boolean drawSoftware(Surface surface, AttachInfo attachInfo, int xoff, int yoff,  
        boolean scalingRequired, Rect dirty, Rect surfaceInsets) {  
    ...
    mView.draw(canvas);  
    ...
    return true;  
}

```
在ViewRootImpl最终通过`drawSoftware`调用到了DecorView的draw方法，看下代码。
```java
@Override  
public void draw(Canvas canvas) {  
    super.draw(canvas);  
  
    if (mMenuBackground != null) {  
        mMenuBackground.draw(canvas);  
    }  
}
```
可以看到这里我们关注的重点还是`super.draw`方法，但其实FrameLayout和ViewGroup都没有重写该方法，所以直接看下View的`draw`。
```java
public void draw(Canvas canvas) {  

    drawBackground(canvas);  
  
    if (!verticalEdges && !horizontalEdges) {  
	/*  
	 * Draw traversal performs several drawing steps which must be executed * in the appropriate order: 
	 * 
	 *      1. Draw the background 
	 *      2. If necessary, save the canvas' layers to prepare for fading 
	 *      3. Draw view's content 
	 *      4. Draw children 
	 *      5. If necessary, draw the fading edges and restore layers 
	 *      6. Draw decorations (scrollbars for instance) 
	 *      7. If necessary, draw the default focus highlight 
	 */
        // Step 3, draw the content  
        onDraw(canvas);  
  
        // Step 4, draw the children  
        dispatchDraw(canvas);  
  
        // Step 6, draw decorations (foreground, scrollbars)  
        onDrawForeground(canvas);  
  
        // Step 7, draw the default focus highlight  
        drawDefaultFocusHighlight(canvas);  
  
        return;  
    }  
  
```
这段代码的注释中描述了`draw`方法需要完成的7个主要步骤，翻译一下：
1. 渲染背景；
2. 如果有必要，保存画布的图层为褪色做准备；
3. 画view的内容，也就是`onDraw`方法；
4. 画子view；
5. 如果有必要，绘制褪色边缘和恢复层；
6. 画装饰，例如滚动条；
7. 如果有必要，绘制默认焦点高亮；
其实这里我们主要关注两个步骤，第一点绘制自己，也就是`onDraw`方法，第二点就是`dispatchDraw`通知子view进行绘制。我们先看下DecorView的`onDraw`方法。
```java
@Override  
public void onDraw(Canvas c) {  
    super.onDraw(c);  
  
    mBackgroundFallback.draw(this, mContentRoot, c, mWindow.mContentParent,  
            mStatusColorViewState.view, mNavigationColorViewState.view);  
}
```
其实就是调用父类的`onDraw`方法，犹豫FrameLayout和ViewGroup都没有重写这个方法，而View中的该方法是空实现，所以其实`super.onDraw`什么都没做。DecorView绘制完自己的东西，就执行了我们说的第二点`dispatchDraw`，DecorView和FrameLayout都没有实现dispatchDraw，我们直接看一下ViewGroup中该方法的实现。
```java
@Override  
protected void dispatchDraw(Canvas canvas) {  
	...
    for (int i = 0; i < childrenCount; i++) {  
        while (transientIndex >= 0 && mTransientIndices.get(transientIndex) == i) {  
            final View transientChild = mTransientViews.get(transientIndex);  
            if ((transientChild.mViewFlags & VISIBILITY_MASK) == VISIBLE ||  
                    transientChild.getAnimation() != null) {  
                more |= drawChild(canvas, transientChild, drawingTime);  
            }  
	         ...
        }  
        ...
}

protected boolean drawChild(Canvas canvas, View child, long drawingTime) {  
    return child.draw(canvas, this, drawingTime);  
}
```
其实发现`dispatchDraw`就是分发给每一个子view的`draw`方法。传递的参数分别为当下的画布，view以及当前画动作发生的时间点。这样又回到了draw方法上，上面已经分析过了，如果是ViewGroup则继续分发给子view去调用draw方法，如果是叶子结点则画自己即可。

到这里，DecorView的创建以及绘制流程以及相关的PhoneWindow和ViewRootImpl的创建节点就彻底介绍完了。

# View绘制

1. view 不停找 parent 可以一直找到 DecorView，按理说 DecorView 是顶点了，但是 DecorView 还有个虚拟父 view，ViewRootImpl。 ViewRootImpl 不是一个 View 或者ViewGroup，他有个成员 mView 是 DecorView，所有的操作从 ViewRootImpl 开始自上而下分发。
2. view 的 invalidate 不会导致 ViewRootImpl 的 invalidate 被调用，而是递归调用父 view的invalidateChildInParent，直到 ViewRootImpl 的 invalidateChildInParent，然后触发peformTraversals，会导致当前 view 被重绘,由于 mLayoutRequested 为 false，不会导致 onMeasure 和 onLayout 被调用，而 OnDraw 会被调用。
3. 一个 view 的 invalidate 会导致本身 PFLAG_INVALIDATED 置 1，导致本身以及父族 viewgroup 的 PFLAG_DRAWING_CACHE_VALID 置 0。
4. requestLayout 会直接递归调用父窗口的 requestLayout，直到 ViewRootImpl，然后触发 peformTraversals，由于 mLayoutRequested 为 true，会导致 onMeasure 和onLayout 被调用。不一定会触发 OnDraw。
5. requestLayout 触发 onDraw 可能是因为在在 layout 过程中发现 l, t, r, b 和以前不一样，那就会触发一次 invalidate，所以触发了onDraw，也可能是因为别的原因导致 mDirty 非空（比如在跑动画）。
6. requestLayout 会导致自己以及父族 view 的 PFLAG_FORCE_LAYOUT 和 PFLAG_INVALIDATED 标志被设置。
7. 一般来说，只要刷新的时候就调用 invalidate，需要重新 measure 就调用 requestLayout，后面再跟个 invalidate（为了保证重绘）。


### 相关文章

[Android View绘制流程](https://www.cnblogs.com/huansky/p/11911549.html)
[事件分发](https://www.cnblogs.com/huansky/p/9656394.html)

[【朝花夕拾】Android自定义View篇之（一）View绘制流程 - 宋者为王 - 博客园](https://www.cnblogs.com/andy-songwei/p/10955062.html)
[Android View的绘制流程 - 简书](https://www.jianshu.com/p/5a71014e7b1b?from=singlemessage)