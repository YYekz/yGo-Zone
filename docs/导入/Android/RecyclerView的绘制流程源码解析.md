# RecyclerView源码解析

> 本文基于 `androidx.recyclerview:recyclerview:1.2.1` 版本。

如果说每个Android开发都一定用过的组件，那RecyclerView一定位列其中。那么RecyclerView好在哪里呢？

本文先从绘制入手，到缓存最后以QA的形式收尾。

## 绘制

### onMeasure
说到底RecyclerView还是一个View，更严格的说，他是一个ViewGroup。还分别实现了`ScrollingView`、`NestedScrollingChild2`和`NestedScrollingChild3`接口，这三个的具体作用，我们后面细说。
```java
public class RecyclerView extends ViewGroup implements ScrollingView,
        NestedScrollingChild2, NestedScrollingChild3 {
        }
```
那既然是View就逃不开`onMeasure`、`onLayout`、`onDraw`，先来看下`onMeasure`。
```java
@Override
    protected void onMeasure(int widthSpec, int heightSpec) {
        if (mLayout == null) {
            //情况一：mLayout为空
            defaultOnMeasure(widthSpec, heightSpec);
            return;
        }
        if (mLayout.isAutoMeasureEnabled()) {
              //情况二：自动测量打开
        }else{
                //情况三：自动测量关闭
        }
```
可以看到`onMeasure`分为三个逻辑分支：
#### 情况一：mLayout为空
`mLayout`是什么呢？其实就是在我们平时使用时，调用`setLayoutManager`所设置的LayoutManager。

```java
    void defaultOnMeasure(int widthSpec, int heightSpec) {
        final int width = LayoutManager.chooseSize(widthSpec,
                getPaddingLeft() + getPaddingRight(),
                ViewCompat.getMinimumWidth(this));
        final int height = LayoutManager.chooseSize(heightSpec,
                getPaddingTop() + getPaddingBottom(),
                ViewCompat.getMinimumHeight(this));

        setMeasuredDimension(width, height);
    }
    
    public static int chooseSize(int spec, int desired, int min) {
            final int mode = View.MeasureSpec.getMode(spec);
            final int size = View.MeasureSpec.getSize(spec);
            switch (mode) {
                case View.MeasureSpec.EXACTLY:
                    return size;
                case View.MeasureSpec.AT_MOST:
                    return Math.min(size, Math.max(desired, min));
                case View.MeasureSpec.UNSPECIFIED:
                default:
                    return Math.max(desired, min);
            }
        }
```
其实就是简单的根据不同的模式设置RecyclerView对应的宽高，没有调用child的测量绘制相关代码，这也就是我们平时未设置LayoutManager时界面为空的原因。

#### 情况二：自动测量打开（mLayout.isAutoMeasureEnabled()）
```java
if (mLayout.isAutoMeasureEnabled()) {
            final int widthMode = MeasureSpec.getMode(widthSpec);
            final int heightMode = MeasureSpec.getMode(heightSpec);
            
                //该方法的内部实现其实还是调用recyclerView.defaultOnMeasure()
            mLayout.onMeasure(mRecycler, mState, widthSpec, heightSpec);

            mLastAutoMeasureSkippedDueToExact =
                    widthMode == MeasureSpec.EXACTLY && heightMode == MeasureSpec.EXACTLY;
            if (mLastAutoMeasureSkippedDueToExact || mAdapter == null) {
	            //如果宽高都是精确模式或者adapter为空，则跳过下面的步骤
                return;
            }
            
            if (mState.mLayoutStep == State.STEP_START) {
                dispatchLayoutStep1();  //关键步骤1
            }

            mLayout.setMeasureSpecs(widthSpec, heightSpec);
            mState.mIsMeasuring = true;
            dispatchLayoutStep2(); //关键步骤2

            mLayout.setMeasuredDimensionFromChildren(widthSpec, heightSpec);

            if (mLayout.shouldMeasureTwice()) {
                mLayout.setMeasureSpecs(
                        MeasureSpec.makeMeasureSpec(getMeasuredWidth(), MeasureSpec.EXACTLY),
                        MeasureSpec.makeMeasureSpec(getMeasuredHeight(), MeasureSpec.EXACTLY));
                mState.mIsMeasuring = true;
                dispatchLayoutStep2();
                mLayout.setMeasuredDimensionFromChildren(widthSpec, heightSpec);
            }

            mLastAutoMeasureNonExactMeasuredWidth = getMeasuredWidth();
            mLastAutoMeasureNonExactMeasuredHeight = getMeasuredHeight();
        }

		//LayoutManager
        public void onMeasure(@NonNull Recycler recycler, @NonNull State state, int widthSpec,
                int heightSpec) {
            mRecyclerView.defaultOnMeasure(widthSpec, heightSpec);
        }
```
`mLayout.isAutoMeasureEnabled()`其实是判断`mAutoMeasure`的值，这个字段默认为false，且外部仅可以通过一个已废弃方法`mAutoMeasure(boolean enabled)`设置。 我们常使用的LinearLayoutManager默认返回了true。`mLayout.onMeasure()`内部的实现就是调用了之前说过的`defaultOnMeasure`。

这里会看到了一个`state.mLayoutStep`。State对象创建在RecyclerView创建时，Step共分为三步。
|Key|意义|赋值时机|
|-|-|-|
|STEP_START| 表示初始化状态，调用`dispatchLayoutStep1`。 |`mLayoutStep`的默认值和`STEP_ANIMATIONS`也就是`dispatchLayoutStep3`执行完毕之后|
|STEP_LAYOUT|当`mLayoutStep`为该值时，会调用`dispatchLayoutStep2`方法。 | `dispatchLayoutStep1`执行完毕后，会将`mLayoutStep`置为`STEP_LAYOUT`|
|STEP_ANIMATIONS| 执行`dispatchLayoutStep3`方法，执行动画阶段。 | `dispatchLayoutStep2`执行完毕后，会将`mLayoutStep`置为`STEP_ANIMATIONS` |

好了接着到了`dispatchLayoutStep1`，简化代码如下。
```java
/**
     * The first step of a layout where we;
     * - process adapter updates
     * - decide which animation should run
     * - save information about current views
     * - If necessary, run predictive layout and save its information
     */
    private void dispatchLayoutStep1() {
        mState.assertLayoutStep(State.STEP_START);
        fillRemainingScrollValues(mState);
        mState.mIsMeasuring = false;
        startInterceptRequestLayout();
        mViewInfoStore.clear();
        onEnterLayoutOrScroll();
        processAdapterUpdatesAndSetAnimationFlags();
        saveFocusInfo();
        //初始化部分State的状态
        mState.mTrackOldChangeHolders = mState.mRunSimpleAnimations && mItemsChanged;
        mItemsAddedOrRemoved = mItemsChanged = false;
        mState.mInPreLayout = mState.mRunPredictiveAnimations;
        mState.mItemCount = mAdapter.getItemCount();
        findMinMaxChildLayoutPositions(mMinMaxLayoutPositions);
        if (mState.mRunSimpleAnimations) {
            // 第一步：找到所有未移除的ViewHolder
        }
        if (mState.mRunPredictiveAnimations) {
            //第二步：进行预布局
            mLayout.onLayoutChildren(mRecycler, mState);
        } else {
            clearOldPositions();
        }
        onExitLayoutOrScroll();
        stopInterceptRequestLayout(false);
        //设置为下一步状态
        mState.mLayoutStep = State.STEP_LAYOUT;
    }
```
文中关键的两步是根据`mState.mRunSimpleAnimations`和`mState.mRunPredictiveAnimations`来的，那他们俩是什么意思呢？一起来看下赋值逻辑，其实就是在前面几行的`processAdapterUpdatesAndSetAnimationFlags()`中。
```java
    private void processAdapterUpdatesAndSetAnimationFlags() {
        ...
        mState.mRunSimpleAnimations = mFirstLayoutComplete
                && mItemAnimator != null
                && (mDataSetHasChangedAfterLayout
                || animationTypeSupported
                || mLayout.mRequestedSimpleAnimations)
                && (!mDataSetHasChangedAfterLayout
                || mAdapter.hasStableIds());
        mState.mRunPredictiveAnimations = mState.mRunSimpleAnimations
                && animationTypeSupported
                && !mDataSetHasChangedAfterLayout
                && predictiveItemAnimationsEnabled();
    }
```
这个方法只被两个方法调用，分别是：`onMeasure`和`dispatchLayout`。我们重点关注下这两个值的赋值逻辑，里面有一个`mFirstLayoutComplete`字段，该字段赋值逻辑在`onAttachedToWindow`和`onLayout`方法，**也就是说当我们RecyclerView第一次加载数据时，并不会执行动画。**
```java
protected void onAttachedToWindow() {
        ...
        mFirstLayoutComplete = mFirstLayoutComplete && !isLayoutRequested();
        ...
    }
    
    @Override
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        TraceCompat.beginSection(TRACE_ON_LAYOUT_TAG);
        dispatchLayout();
        TraceCompat.endSection();
        mFirstLayoutComplete = true;
    }
```
回到我们的`dispatchLayoutStep1`，其实该方法的主要意义就是预布局，做一些初始化操作。走完上述逻辑后，将`mState.mLayoutStep`设置为`State.STEP_LAYOUT`，进入下一个关键方法`dispatchLayoutStep2`。
```java
 private void dispatchLayoutStep2() {
        startInterceptRequestLayout();
        onEnterLayoutOrScroll();
        mState.assertLayoutStep(State.STEP_LAYOUT | State.STEP_ANIMATIONS);
        mAdapterHelper.consumeUpdatesInOnePass();
        //1. itemCount方法被调用，获取当前列表中的item数量
        mState.mItemCount = mAdapter.getItemCount();
        mState.mDeletedInvisibleItemCountSincePreviousLayout = 0;
        if (mPendingSavedState != null && mAdapter.canRestoreState()) {
            if (mPendingSavedState.mLayoutState != null) {
                mLayout.onRestoreInstanceState(mPendingSavedState.mLayoutState);
            }
            mPendingSavedState = null;
        }
        mState.mInPreLayout = false;
        //2. 调用LayoutManager方法，为子view布局
        mLayout.onLayoutChildren(mRecycler, mState);

        mState.mStructureChanged = false;

        // onLayoutChildren may have caused client code to disable item animations; re-check
        mState.mRunSimpleAnimations = mState.mRunSimpleAnimations && mItemAnimator != null;
        //3. 进入下一状态
        mState.mLayoutStep = State.STEP_ANIMATIONS;
        onExitLayoutOrScroll();
        stopInterceptRequestLayout(false);
    }
```
这个方法才是真正对子view进行测量和布局的方法，且这一方法可能会被多次调用。

先看第一步，也就是我们设置adapter中重写的`getItemCount`，获取当前需要布局的子item数量保存在State中。然后调用LayoutManager的`onLayoutChildren`通知LayoutManager去为子view布局，该方法父类中无任何实现逻辑，是每个LayoutManager必须实现的方法。
```java 
        public void onLayoutChildren(Recycler recycler, State state) {
            Log.e(TAG, "You must override onLayoutChildren(Recycler recycler, State state) ");
        }
```
举个我们最尝使用的LinearLayoutManager为例，看下他的onLayoutChildren具体做了什么。
```java
@Override  
public void onLayoutChildren(RecyclerView.Recycler recycler, RecyclerView.State state) {  
    if (mPendingSavedState != null || mPendingScrollPosition != RecyclerView.NO_POSITION) {  
        if (state.getItemCount() == 0) {  
            removeAndRecycleAllViews(recycler);  
            return;        
            }  
    }  
    if (mPendingSavedState != null && mPendingSavedState.hasValidAnchor()) {  
        mPendingScrollPosition = mPendingSavedState.mAnchorPosition;  
    }  
  
    ensureLayoutState();  
    mLayoutState.mRecycle = false;  
	//判断布局方向：从右至左或从左至右
    resolveShouldLayoutReverse();  

	//获取当前已获得焦点的child
    final View focused = getFocusedChild();  
    if (!mAnchorInfo.mValid || mPendingScrollPosition != RecyclerView.NO_POSITION  
            || mPendingSavedState != null) {  
        mAnchorInfo.reset();  
        mAnchorInfo.mLayoutFromEnd = mShouldReverseLayout ^ mStackFromEnd;  
        //获取当前的锚点view，更新锚点信息
        updateAnchorInfoForLayout(recycler, state, mAnchorInfo);  
        mAnchorInfo.mValid = true;  
    } else if (focused != null && (mOrientationHelper.getDecoratedStart(focused)  
            >= mOrientationHelper.getEndAfterPadding()  
            || mOrientationHelper.getDecoratedEnd(focused)  
            <= mOrientationHelper.getStartAfterPadding())) {     
        mAnchorInfo.assignFromViewAndKeepVisibleRect(focused, getPosition(focused));  
    }  
    if (DEBUG) {  
        Log.d(TAG, "Anchor info:" + mAnchorInfo);  
    }  

    ...//省略一些方向、空间的判断信息代码
  
    onAnchorReady(recycler, state, mAnchorInfo, firstLayoutDirection);  

	//核心方法，view回收
    detachAndScrapAttachedViews(recycler);  
    mLayoutState.mInfinite = resolveIsInfinite();  
    mLayoutState.mIsPreLayout = state.isPreLayout();  
    // noRecycleSpace not needed: recycling doesn't happen in below's fill  
    // invocations because mScrollingOffset is set to SCROLLING_OFFSET_NaN    mLayoutState.mNoRecycleSpace = 0;  
    if (mAnchorInfo.mLayoutFromEnd) {  
        // 从下向上  
        updateLayoutStateToFillStart(mAnchorInfo);  
        mLayoutState.mExtraFillSpace = extraForStart;  
        fill(recycler, mLayoutState, state, false);  
        startOffset = mLayoutState.mOffset;  
        final int firstElement = mLayoutState.mCurrentPosition;  
        if (mLayoutState.mAvailable > 0) {  
            extraForEnd += mLayoutState.mAvailable;  
        }  
        // fill towards end  
        updateLayoutStateToFillEnd(mAnchorInfo);  
        mLayoutState.mExtraFillSpace = extraForEnd;  
        mLayoutState.mCurrentPosition += mLayoutState.mItemDirection;  
        fill(recycler, mLayoutState, state, false);  
        endOffset = mLayoutState.mOffset;  
  
        if (mLayoutState.mAvailable > 0) {  
            // end could not consume all. add more items towards start  
            extraForStart = mLayoutState.mAvailable;  
            updateLayoutStateToFillStart(firstElement, startOffset);  
            mLayoutState.mExtraFillSpace = extraForStart;  
            fill(recycler, mLayoutState, state, false);  
            startOffset = mLayoutState.mOffset;  
        }  
    } else {  
        // 从上到下
        updateLayoutStateToFillEnd(mAnchorInfo);  
        //先向下填充
        mLayoutState.mExtraFillSpace = extraForEnd;  
        //填充方法
        fill(recycler, mLayoutState, state, false);  
        endOffset = mLayoutState.mOffset;  
        final int lastElement = mLayoutState.mCurrentPosition;  
        if (mLayoutState.mAvailable > 0) {  
            extraForStart += mLayoutState.mAvailable;  
        }  
        //再向上填充
        updateLayoutStateToFillStart(mAnchorInfo);  
        mLayoutState.mExtraFillSpace = extraForStart;  
        mLayoutState.mCurrentPosition += mLayoutState.mItemDirection;  
        fill(recycler, mLayoutState, state, false);  
        startOffset = mLayoutState.mOffset;  

		//如果还有可用空间则再向下填充一次
        if (mLayoutState.mAvailable > 0) {  
            extraForEnd = mLayoutState.mAvailable;  
            // start could not consume all it should. add more items towards end  
            updateLayoutStateToFillEnd(lastElement, endOffset);  
            mLayoutState.mExtraFillSpace = extraForEnd;  
            fill(recycler, mLayoutState, state, false);  
            endOffset = mLayoutState.mOffset;  
        }  
    }  
	...
}
```
很清晰，可以将LLM的`onLayoutChildren`拆分为为几步：
##### 1. 首先寻找当前的锚点，并计算出锚点的位置；
```java
private void updateAnchorInfoForLayout(RecyclerView.Recycler recycler, RecyclerView.State state,  
        AnchorInfo anchorInfo) {  
        //recyclerView重建，恢复上次的布局
    if (updateAnchorFromPendingData(state, anchorInfo)) {  
        if (DEBUG) {  
            Log.d(TAG, "updated anchor info from pending information");  
        }  
        return;  
    }  
		//从children中获取锚点信息，如果有view获取了焦点，则该child就为锚点；否则根据填充方向选择第一个或者最后一个可见的view。
    if (updateAnchorFromChildren(recycler, state, anchorInfo)) {  
        if (DEBUG) {  
            Log.d(TAG, "updated anchor info from existing children");  
        }  
        return;  
    }  
    if (DEBUG) {  
        Log.d(TAG, "deciding anchor info for fresh state");  
    }  
    anchorInfo.assignCoordinateFromPadding();  
    //都没有，则根据填充方向来判断，是第一个可见的view或者是最后一个可见的view。一般第一次加载recyclerview会使用该方法，因为此时view上没有任何子view。
    anchorInfo.mPosition = mStackFromEnd ? state.getItemCount() - 1 : 0;  
}
```
在找到对应的锚点后，通过`detachAndScrapAttachedViews`方法进行view的回收；
##### 2. 根据当前设置的方向分别向上和向下进行填充；
##### 3.如果填充不满就再填充一次；
刚才讲到了回收view的方法，那获取view的方法呢？其实就在当前方法的`fill`方法中，一起来看下该方法。
```java
int fill(RecyclerView.Recycler recycler, LayoutState layoutState,  
        RecyclerView.State state, boolean stopOnFocusable) {  
	...
    while ((layoutState.mInfinite || remainingSpace > 0) && layoutState.hasMore(state)) {  
	    ... 
        layoutChunk(recycler, state, layoutState, layoutChunkResult);  
		...
    }  
    return start - layoutState.mAvailable;  
}
```
从关键代码可以看出，如果判断出当前还有空间可以展示，就不断地填充进去。其中，真正添加view的方法，就是`layoutChunk`。
```java
void layoutChunk(RecyclerView.Recycler recycler, RecyclerView.State state,  
        LayoutState layoutState, LayoutChunkResult result) {  
        //获取view
    View view = layoutState.next(recycler);  
		//添加view
    RecyclerView.LayoutParams params = (RecyclerView.LayoutParams) view.getLayoutParams();  
    if (layoutState.mScrapList == null) {  
        if (mShouldReverseLayout == (layoutState.mLayoutDirection  
                == LayoutState.LAYOUT_START)) {  
            addView(view);  
        } else {  
            addView(view, 0);  
        }  
    } else {  
        if (mShouldReverseLayout == (layoutState.mLayoutDirection  
                == LayoutState.LAYOUT_START)) {  
            addDisappearingView(view);  
        } else {  
            addDisappearingView(view, 0);  
        }  
    }  
	//测量view
    measureChildWithMargins(view, 0, 0);  
    ...
	//布局itemview   
    layoutDecoratedWithMargins(view, left, top, right, bottom);  
	...
}

public void measureChildWithMargins(@NonNull View child, int widthUsed, int heightUsed) {  
    final LayoutParams lp = (LayoutParams) child.getLayoutParams();  
  
    final Rect insets = mRecyclerView.getItemDecorInsetsForChild(child);  
    widthUsed += insets.left + insets.right;  
    heightUsed += insets.top + insets.bottom;  
  
    final int widthSpec = getChildMeasureSpec(getWidth(), getWidthMode(),  
            getPaddingLeft() + getPaddingRight()  
                    + lp.leftMargin + lp.rightMargin + widthUsed, lp.width,  
            canScrollHorizontally());  
    final int heightSpec = getChildMeasureSpec(getHeight(), getHeightMode(),  
            getPaddingTop() + getPaddingBottom()  
                    + lp.topMargin + lp.bottomMargin + heightUsed, lp.height,  
            canScrollVertically());  
    if (shouldMeasureChild(child, widthSpec, heightSpec, lp)) {  
        child.measure(widthSpec, heightSpec);  
    }  
}

public void layoutDecoratedWithMargins(@NonNull View child, int left, int top, int right,  
        int bottom) {  
    final LayoutParams lp = (LayoutParams) child.getLayoutParams();  
    final Rect insets = lp.mDecorInsets;  
    child.layout(left + insets.left + lp.leftMargin, top + insets.top + lp.topMargin,  
            right - insets.right - lp.rightMargin,  
            bottom - insets.bottom - lp.bottomMargin);  
}
```
最终终于看到了`measureChild`方法，通过这个方法进行子view的测量，然后通过`layoutDecoratedWithMargins`对子view进行布局。
> 顺便提一下，这个layoutState.next(recycler) 方法是RecyclerView缓存机制中非常重要的一个方法。

最后一步将state设置为下一状态，进行`dispatchLayoutStep3`，这个方法主要是做和动画相关的操作并将State中的step重置回START状态。那现在我们一起来看看`dispatchLayoutStep3`到底做了什么。
```java
private void dispatchLayoutStep3() {
        mState.assertLayoutStep(State.STEP_ANIMATIONS);
        startInterceptRequestLayout();
        onEnterLayoutOrScroll();
        mState.mLayoutStep = State.STEP_START;
        if (mState.mRunSimpleAnimations) {
            //大段的动画执行
            ...
        }    
        //回收当前废弃的viewHolder
        mLayout.removeAndRecycleScrapInt(mRecycler);
        //state状态的reset
        mState.mPreviousLayoutItemCount = mState.mItemCount;
        mDataSetHasChangedAfterLayout = false;
        mDispatchItemsChangedEvent = false;
        mState.mRunSimpleAnimations = false;

        mState.mRunPredictiveAnimations = false;
        mLayout.mRequestedSimpleAnimations = false;
        if (mRecycler.mChangedScrap != null) {
            mRecycler.mChangedScrap.clear();
        }
        if (mLayout.mPrefetchMaxObservedInInitialPrefetch) {
            // Initial prefetch has expanded cache, so reset until next prefetch.
            // This prevents initial prefetches from expanding the cache permanently.
            mLayout.mPrefetchMaxCountObserved = 0;
            mLayout.mPrefetchMaxObservedInInitialPrefetch = false;
            mRecycler.updateViewCacheSize();
        }
        
        //告知layoutManager layout完毕
        mLayout.onLayoutCompleted(mState);
        onExitLayoutOrScroll();
        stopInterceptRequestLayout(false);
        mViewInfoStore.clear();
        if (didChildRangeChange(mMinMaxLayoutPositions[0], mMinMaxLayoutPositions[1])) {
            dispatchOnScrolled(0, 0);
        }
        recoverFocusFromState();
        resetFocusInfo();
    }
```
其实看下来发现`dispatchLayoutStep3`主要就做了三件事情：
1. reset掉当前State的相关状态，包括`mLayStep`重置到START状态以及其他动画等值reset；
2. 回收废弃的ViewHolder（在缓存章节展开讲）；
3. 告知`mLayout`，RecyclerView的`onLayout`方法执行完毕；

现在我们还是回到`onMeasure`中看一下情况三，完成`onMeasure`的梳理。
#### 情况三：自动测量关闭
```java
        {
            if (mHasFixedSize) {
                mLayout.onMeasure(mRecycler, mState, widthSpec, heightSpec);
                return;
            }
            // custom onMeasure
            if (mAdapterUpdateDuringMeasure) {
                startInterceptRequestLayout();
                onEnterLayoutOrScroll();
                processAdapterUpdatesAndSetAnimationFlags();
                onExitLayoutOrScroll();

                if (mState.mRunPredictiveAnimations) {
                    mState.mInPreLayout = true;
                } else {
                    // consume remaining updates to provide a consistent state with the layout pass.
                    mAdapterHelper.consumeUpdatesInOnePass();
                    mState.mInPreLayout = false;
                }
                mAdapterUpdateDuringMeasure = false;
                stopInterceptRequestLayout(false);
            } else if (mState.mRunPredictiveAnimations) {
                setMeasuredDimension(getMeasuredWidth(), getMeasuredHeight());
                return;
            }

            if (mAdapter != null) {
                mState.mItemCount = mAdapter.getItemCount();
            } else {
                mState.mItemCount = 0;
            }
            startInterceptRequestLayout();
            mLayout.onMeasure(mRecycler, mState, widthSpec, heightSpec);
            stopInterceptRequestLayout(false);
            mState.mInPreLayout = false; // clear
        }
```
其实核心目的就是调用`mLayout.onMeasure`方法也就是上文说过的`defaultOnMeasure`，这里就不再细说了。

#### 总结
看下来，`onMeasure`方法只是根据对应的基础判断走不同的状态初始化，未设置LayoutManager未设置或者自动测量未开启时，只会走基础测量`defaultOnMeasure`方法。开启自动测量后，会进行view的测量与布局。其实整体RecyclerView的测量逻辑都是委托给LayoutManager去实现和判断的。

### onLayout
来到`onLayout`，我们先看下代码。
```java
    @Override
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        TraceCompat.beginSection(TRACE_ON_LAYOUT_TAG);
        //核心方法
        dispatchLayout();
        TraceCompat.endSection();
        //就是上文中提到的是否是第一次layout，根据这个bool值去判断是否进行动画
        mFirstLayoutComplete = true;
    }
    
    void dispatchLayout() {
	    //未设置adapter
        if (mAdapter == null) {
            Log.e(TAG, "No adapter attached; skipping layout");
            return;
        }
        //未设置layoutmanager
        if (mLayout == null) {
            Log.e(TAG, "No layout manager attached; skipping layout");
            return;
        }
        mState.mIsMeasuring = false;
        if (mState.mLayoutStep == State.STEP_START) {
        //onMeasure方法中自动测量关闭时
            dispatchLayoutStep1();
            mLayout.setExactMeasureSpecsFrom(this);
            dispatchLayoutStep2();
        } else if (mAdapterHelper.hasUpdates() || mLayout.getWidth() != getWidth()
                || mLayout.getHeight() != getHeight()) {
            //如果view布局大小有变化
            mLayout.setExactMeasureSpecsFrom(this);
            dispatchLayoutStep2();
        } else {
            // always make sure we sync them (to ensure mode is exact)
            mLayout.setExactMeasureSpecsFrom(this);
        }
        dispatchLayoutStep3();
    }
```
能看到，`onLayout`中，核心逻辑是在`dispatchLayout`中的，该方法也不复杂。首先判断`mAdapter`和`mLayout`是否为空，所以当我们忘记设置这俩对象时，页面中是无法看到任何效果的，因为不会做任何的layout操作。接下来根据`mStep.mLayoutStep`的状态去判断是否调用`dispatchLayoutStep1`，如果再onMeasure方法中已经执行过该步骤（也就是说打开了自动测量且宽高不是设置绝对值的时候），则会判断view的布局大小是否有变化，如果有变化，则需要重新执行，`dispatchLayoutStep2`，去重新测量和布局子view（所以说该方法可能会多次执行）。最终会执行到`dispatchLayoutStep3`。其实说到底，**`onLayout`也就是`dispatchLayout`方法，是为了保证`dispatchLayoutStep1`、`dispatchLayoutStep2`和`dispatchLayoutStep3`务必都执行。**

#### 总结

`onLayout`方法其实就是：保证三兄弟一定执行到，告知LayoutManager当前执行状态方便LayoutManager处理相关逻辑。
那大概总结一下三兄弟相关任务。
| 方法名 | 任务 | 执行次数 |
|-|-|-|
|dispatchLayoutStep1| 根据相关配置判断是否执行动画，是否进行预加载，保存旧item信息 | 一次流程一次 |
|dispatchLayoutStep2| 告知LayoutManager进行子view的测量和布局 | 一次流程至少一次|
|dispatchLayoutStep3| 执行相关动画，告知LayoutManager当前状态以及废弃ViewHolder回收 |一次流程一次|

### onDraw
进入绘制的最后一步`onDraw`。
```java
    @Override
    public void onDraw(Canvas c) {
        super.onDraw(c);

        final int count = mItemDecorations.size();
        for (int i = 0; i < count; i++) {
            mItemDecorations.get(i).onDraw(c, this, mState);
        }
    }
```
可以看到，`onDraw`方法实现非常简单，就是调用ViewGroup的`dispatchDraw`通知子View绘制以及根据当前的ItemDecoration去绘制每个子view。

## 核心组件
### ViewHolder

### Adapter

### ItemDecoration

### Recycler
Recyler负责管理ViewHolder，它可以回收起已经不被展示的ViewHolder，并在恰当的时候复用这些ViewHolder。它最重要的一个能力就是根据position提供一个ViewHolder/View，使用者无需关心这个ViewHolder是新创建的还是复用已有的，Recycler帮助RecyclerView处理ViewHolder的缓存。


## onLayout
在最开始使用RecyclerView时，经常出现忘记设置layoutManger的情况，导致什么都不展示，是为什么呢？跟进去看一下。


### LayoutManager

### AdapterHelper



## 带着问题去看
### 没有设置LayoutManager时，RecyclerView为什么不会显示子view？
```java
在RecyclerView的onMeasure中，
protected void onMeasure(int widthSpec, int heightSpec) {
    if (mLayout == null) {
        defaultOnMeasure(widthSpec, heightSpec);
        return;
    }
    if (mLayout.isAutoMeasureEnabled()) {
        ...
        }else{
        ...
        }
}
```
mLayout就是我们设置的LayoutManager对象的引用，那defaultOnMeasure中做了什么？其实就是根据对应的Layout模式进行宽高获取。
```java
void defaultOnMeasure(int widthSpec, int heightSpec) {
        // calling LayoutManager here is not pretty but that API is already public and it is better
        // than creating another method since this is internal.
        final int width = LayoutManager.chooseSize(widthSpec,
                getPaddingLeft() + getPaddingRight(),
                ViewCompat.getMinimumWidth(this));
        final int height = LayoutManager.chooseSize(heightSpec,
                getPaddingTop() + getPaddingBottom(),
                ViewCompat.getMinimumHeight(this));

        setMeasuredDimension(width, height);
    }

public static int chooseSize(int spec, int desired, int min) {
            final int mode = View.MeasureSpec.getMode(spec);
            final int size = View.MeasureSpec.getSize(spec);
            switch (mode) {
                case View.MeasureSpec.EXACTLY:
                    return size;
                case View.MeasureSpec.AT_MOST:
                    return Math.min(size, Math.max(desired, min));
                case View.MeasureSpec.UNSPECIFIED:
                default:
                    return Math.max(desired, min);
            }
        }
```
最关键的地方其实是因为在RecyclerView中的onLayout方法中，当mLayout为空时，会return layout逻辑，所以就不会显示View了。
```java
void dispatchLayout() {
        if (mAdapter == null) {
            Log.w(TAG, "No adapter attached; skipping layout");
            // leave the state in START
            return;
        }
        if (mLayout == null) {
            Log.e(TAG, "No layout manager attached; skipping layout");
            // leave the state in START
            return;
        }
    ...
}
```