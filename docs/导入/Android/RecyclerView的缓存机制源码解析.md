[[RecyclerView的绘制流程源码解析]]
书接上文，我们分析过RecyclerView的绘制源码后，一起来看看众所周知的RecyclerView强大的缓存机制以及实现原理。

让我们带着问题去看复用。
1. 回收复用了什么？
2. RecyclerView什么时候回收复用的？
3. 回收到哪里去？回收规则是什么？
4. RecyclerView是如何利用回收复用的机制提升性能的？

RecyclerView中有个内部类Recycler，他是缓存和复用机制的主要实现者，一起来看下Recycler的实现逻辑。
先来看下他的内部结构：
```java
public final class Recycler {  

	//一级缓存 用来存储的是正在屏幕中显示的ViewHolder
    final ArrayList<ViewHolder> mAttachedScrap = new ArrayList<>();  
    //mChangedScrap一般为空
    ArrayList<ViewHolder> mChangedScrap = null;  

	//二级缓存 用来存储不在屏幕中显示的ViewHolder
    final ArrayList<ViewHolder> mCachedViews = new ArrayList<ViewHolder>();  
	  
    private int mRequestedCacheMax = DEFAULT_CACHE_SIZE;  
    int mViewCacheMax = DEFAULT_CACHE_SIZE;  
	
	//四级缓存 当屏幕外缓存当前ItemType>2时，即放入该pool中
    RecycledViewPool mRecyclerPool;  

	//三级缓存 开发者自定义
    private ViewCacheExtension mViewCacheExtension;  
  
    static final int DEFAULT_CACHE_SIZE = 2;

	...
}
```
根据上述代码我们可以先回答上问中的第一个问题，**回收复用了ViewHolder**。大概梳理下缓存类别：
1. 一级缓存：mAttachedScrap和mChangedScrap。使用来缓存正在屏幕中加载显示的ViewHolder。其中，mChangedScrap一般为空，在使用动画相关时才会有存取操作，在动画执行时存入，在动画执行完毕后即清空；
2. 二级缓存：mCachedViews。缓存因为滚动而刚消失在屏幕可见区域的ViewHolder只缓存两个，FIFO。
3. 三级缓存：mViewCacheExtension。用户自定义缓存。
4. 四级缓存：mRecyclerPool。缓存池，缓存mCachedViews存储不下时，按照ItemType存放至缓存池中，每个ItemType默认缓存最多5个。

收回来，在上篇文章讲到`layoutChunk`时，提到了一个很关键的缓存方法叫`layoutState.next`方法。
```java
View next(RecyclerView.Recycler recycler) {  
    if (mScrapList != null) {  
        return nextViewFromScrapList();  
    }  
    final View view = recycler.getViewForPosition(mCurrentPosition);  
    mCurrentPosition += mItemDirection;  
    return view;  
}

//Recycler
public View getViewForPosition(int position) {  
    return getViewForPosition(position, false);  
}

View getViewForPosition(int position, boolean dryRun) {  
    return tryGetViewHolderForPositionByDeadline(position, dryRun, FOREVER_NS).itemView;  
}
```
最终调用到了`tryGetViewHolderForPositionByDeadline`方法，这个方法在缓存中非常重要。让我们一起来看下。 
```java

ViewHolder tryGetViewHolderForPositionByDeadline(int position,  
        boolean dryRun, long deadlineNs) {  
	
    boolean fromScrapOrHiddenOrCache = false;  
    ViewHolder holder = null;  
    // 0. 从changedScrap里取
	holder = getChangedScrapViewForPosition(position);  
    // 1) Find by position from scrap/hidden list/cache  
    if (holder == null) {  
        holder = getScrapOrHiddenOrCachedHolderForPosition(position, dryRun);  
    }  
    if (holder == null) {  
        final int type = mAdapter.getItemViewType(offsetPosition);  
        // 2) Find from scrap/cache via stable ids, if exists  
        if (mAdapter.hasStableIds()) {  
            holder = getScrapOrCachedViewForId(mAdapter.getItemId(offsetPosition),  
                    type, dryRun);  
        }  
        if (holder == null && mViewCacheExtension != null) {  
            // We are NOT sending the offsetPosition because LayoutManager does not  
            // know it.            
            final View view = mViewCacheExtension  
                    .getViewForPositionAndType(this, position, type);  
            if (view != null) {  
                holder = getChildViewHolder(view);  
            }  
        }  
        if (holder == null) { // fallback to pool  
            holder = getRecycledViewPool().getRecycledView(type);  
        }  
        if (holder == null) {  
            holder = mAdapter.createViewHolder(RecyclerView.this, type);  
        }  
    }  
  
    boolean bound = false;  
    if (mState.isPreLayout() && holder.isBound()) {  
        // do not update unless we absolutely have to.  
        holder.mPreLayoutPosition = position;  
    } else if (!holder.isBound() || holder.needsUpdate() || holder.isInvalid()) {  
        bound = tryBindViewHolderByDeadline(holder, offsetPosition, position, deadlineNs);  
    }  
  
    final ViewGroup.LayoutParams lp = holder.itemView.getLayoutParams();  
    final LayoutParams rvLayoutParams;  
    if (lp == null) {  
        rvLayoutParams = (LayoutParams) generateDefaultLayoutParams();  
        holder.itemView.setLayoutParams(rvLayoutParams);  
    } else if (!checkLayoutParams(lp)) {  
        rvLayoutParams = (LayoutParams) generateLayoutParams(lp);  
        holder.itemView.setLayoutParams(rvLayoutParams);  
    } else {  
        rvLayoutParams = (LayoutParams) lp;  
    }  
    rvLayoutParams.mViewHolder = holder;  
    rvLayoutParams.mPendingInvalidate = fromScrapOrHiddenOrCache && bound;  
    return holder;  
}
```
其实流程已经很清晰了，先从各级缓存中去找，找不到了创建一个新的配置好返回出去。
### 第零个缓存方法：`getChangedScrapViewForPosition`
是从changedScrap里取也就是我们说的动画期间才会有内容的缓存，不重要，我们跳过；
### 第一个缓存方法：`getScrapOrHiddenOrCachedHolderForPosition`
```java
ViewHolder getScrapOrHiddenOrCachedHolderForPosition(int position, boolean dryRun) {  
    final int scrapCount = mAttachedScrap.size();  
	//从mAttachedScrap中取，如果都如何要求，则直接返回
    for (int i = 0; i < scrapCount; i++) {  
        final ViewHolder holder = mAttachedScrap.get(i);  
        if (!holder.wasReturnedFromScrap() && holder.getLayoutPosition() == position  
                && !holder.isInvalid() && (mState.mInPreLayout || !holder.isRemoved())) {  
            holder.addFlags(ViewHolder.FLAG_RETURNED_FROM_SCRAP);  
            return holder;  
        }  
    }  

	//dryRun参数默认为false
    if (!dryRun) {  
	    //从mHiddenViews里面取
        View view = mChildHelper.findHiddenNonRemovedView(position);  
        if (view != null) {  
            final ViewHolder vh = getChildViewHolderInt(view);  
            mChildHelper.unhide(view);  
            int layoutIndex = mChildHelper.indexOfChild(view);  
            if (layoutIndex == RecyclerView.NO_POSITION) {  
                throw new IllegalStateException("layout index should not be -1 after "  
                        + "unhiding a view:" + vh + exceptionLabel());  
            }  
            mChildHelper.detachViewFromParent(layoutIndex);  
            //关键方法
            scrapView(view);  
            vh.addFlags(ViewHolder.FLAG_RETURNED_FROM_SCRAP  
                    | ViewHolder.FLAG_BOUNCED_FROM_HIDDEN_LIST);  
            return vh;  
        }  
    }  
  
	//从mCachedViews中取
    final int cacheSize = mCachedViews.size();  
    for (int i = 0; i < cacheSize; i++) {  
        final ViewHolder holder = mCachedViews.get(i);  
         if (!holder.isInvalid() && holder.getLayoutPosition() == position  
                && !holder.isAttachedToTransitionOverlay()) {  
            if (!dryRun) {  
                mCachedViews.remove(i);  
            }   
            return holder;  
        }  
    }  
    return null;  
}
```
可以看到流程其实很简单，总共分三步：
1. 从mAttachedScrap中查找符合条件的，有就返回；
2. 从mHiddenViews中查找符合条件的，有就从mHiddenViews中移出并添加至mAttachedScrap中；
3. 从mCachedViews中查找符合条件的，有就正常返回并从mCachedViews中移出。
> mHiddenViews是哪里来的？
> 其实也是来自和动画相关的缓存操作，当给某个ViewHolder添加动画调用`addAnimatingView`时会将这个view缓存至mHiddenView中，不是核心主流程，不用特别关心。

需要我们特殊关心的是`scrapView`方法，是我们将ViewHolder添加到scrap的唯一入口，根据不同的条件选择放入mAttachedScrap中或者是mChangedScrap中。
如果拿到ViewHolder之后，则会校验ViewHolder是否有效。
```java
if (holder != null) {  
    if (!validateViewHolderForOffsetPosition(holder)) {  
        // recycle holder (and unscrap if relevant) since it can't be used  
        if (!dryRun) {  
            // we would like to recycle this but need to make sure it is not used by  
            // animation logic etc.            holder.addFlags(ViewHolder.FLAG_INVALID);  
            if (holder.isScrap()) {  
                removeDetachedView(holder.itemView, false);  
                holder.unScrap();  
            } else if (holder.wasReturnedFromScrap()) {  
                holder.clearReturnedFromScrapFlag();  
            }  
            recycleViewHolderInternal(holder);  
        }  
        holder = null;  
    } else {  
        fromScrapOrHiddenOrCache = true;  
    }  
}

boolean validateViewHolderForOffsetPosition(ViewHolder holder) {  
    if (holder.isRemoved()) {  
        if (DEBUG && !mState.isPreLayout()) {  
            throw new IllegalStateException("should not receive a removed view unless it"  
                    + " is pre layout" + exceptionLabel());  
        }  
        return mState.isPreLayout();  
    }  
    if (holder.mPosition < 0 || holder.mPosition >= mAdapter.getItemCount()) {  
        throw new IndexOutOfBoundsException("Inconsistency detected. Invalid view holder "  
                + "adapter position" + holder + exceptionLabel());  
    }  
    if (!mState.isPreLayout()) {  
        final int type = mAdapter.getItemViewType(holder.mPosition);  
        if (type != holder.getItemViewType()) {  
            return false;  
        }  
    }  
    if (mAdapter.hasStableIds()) {  
        return holder.getItemId() == mAdapter.getItemId(holder.mPosition);  
    }  
    return true;  
}
```
什么算是有效呢呢？会根据ViewHolder当前的id、itemType是否一致，ViewHolder是否被标记为Removed或者Deteched等等flag来判断。如果无效则需要将ViewHolder从scrap缓存中删除，并且将该ViewHolder根据不同条件添加到mCachedViews或者mRecyclerPool中，也就是`recycleViewHolderInternal`方法，这个方法也是将ViewHolder回收到mCachedViews或者mRecyclerPool的统一入口。

那`scrapView`和`recycleViewHolderInternal`有什么区别呢？我们将在下一章关于RecyclerView的回收机制中详细讲解。

### 第二个缓存方法：`getScrapOrCachedViewForId`
```java
if (mAdapter.hasStableIds()) {  
    holder = getScrapOrCachedViewForId(mAdapter.getItemId(offsetPosition),  
            type, dryRun);  
    if (holder != null) {  
        // update position  
        holder.mPosition = offsetPosition;  
        fromScrapOrHiddenOrCache = true;  
    }  
}

public final boolean hasStableIds() {  
    return mHasStableIds;  
}


ViewHolder getScrapOrCachedViewForId(long id, int type, boolean dryRun) {  
    final int count = mAttachedScrap.size();  
    for (int i = count - 1; i >= 0; i--) {  
        final ViewHolder holder = mAttachedScrap.get(i);  
        if (holder.getItemId() == id && !holder.wasReturnedFromScrap()) {  
            if (type == holder.getItemViewType()) {  
                holder.addFlags(ViewHolder.FLAG_RETURNED_FROM_SCRAP);  
                if (holder.isRemoved()) {                 
                    if (!mState.isPreLayout()) {  
                        holder.setFlags(ViewHolder.FLAG_UPDATE, ViewHolder.FLAG_UPDATE  
                                | ViewHolder.FLAG_INVALID | ViewHolder.FLAG_REMOVED);  
                    }  
                }  
                return holder;  
            } else if (!dryRun) {                 
                mAttachedScrap.remove(i);  
                removeDetachedView(holder.itemView, false);
	            //   
                quickRecycleScrapView(holder.itemView);  
            }  
        }  
    }  
    final int cacheSize = mCachedViews.size();  
    for (int i = cacheSize - 1; i >= 0; i--) {  
        final ViewHolder holder = mCachedViews.get(i);  
        if (holder.getItemId() == id && !holder.isAttachedToTransitionOverlay()) {  
            if (type == holder.getItemViewType()) {  
                if (!dryRun) {  
                    mCachedViews.remove(i);  
                }  
                return holder;  
            } else if (!dryRun) {  
                recycleCachedViewAt(i);  
                return null;            
                }  
        }  
    }  
    return null;  
}

```
如果`hasStableIds()`返回true的话，就会尝试从cachedView中去取。
> 那这个hasStableIds()到底是什么意思呢？
> 默认情况下这个方法返回为false。官方文档中解释为，设置为true则表示每个Item都是唯一的。
> 1.  `setHasStableIds`这个方法主要跟`notifyDataSetChanged`有关；
2. 当HasStableIds被set为true时，通过`scrapOrRecycleView`方法回收的Holder则永远不会放到`mRecyclerPool`中；
3. 重新设置Adapter和调用`notifyDataSetChanged`方法时，RecyclerView的有效缓存依然会保留；
4. 由`notifyDataSetChanged`发起的重新布局，如果HasStableIds为true，则会最大程度保证不Create Holder。虽然当前所有Attached Item都会重新绑定一次数据，但`mCachedViews`中的缓存还在，所以后面如果要显示的Item刚好在`mCachedViews`中，就不用重新绑定数据了，理论上会提高流畅度（当然了，这么小的差距肉眼根本无法感知的）。

可以看到这个判断和第二种缓存逻辑差不多，只是多了id和ItemType的判断。大体分为几步：
![[未命名文件 (1).png]]
大体逻辑其实就是根据position找到对应的ViewHolder然后匹配id和itemType是否相同，不相同就会从当前缓存中将其删除并添加到下一级缓存中。

```java
void quickRecycleScrapView(View view) {  
    final ViewHolder holder = getChildViewHolderInt(view);  
    holder.mScrapContainer = null;  
    holder.mInChangeScrap = false;  
    holder.clearReturnedFromScrapFlag();  
    recycleViewHolderInternal(holder);  
}

void recycleCachedViewAt(int cachedViewIndex) {  
    if (DEBUG) {  
        Log.d(TAG, "Recycling cached view at index " + cachedViewIndex);  
    }  
    ViewHolder viewHolder = mCachedViews.get(cachedViewIndex);  
    if (DEBUG) {  
        Log.d(TAG, "CachedViewHolder to be recycled: " + viewHolder);  
    }  
    addViewHolderToRecycledViewPool(viewHolder, true);  
    mCachedViews.remove(cachedViewIndex);  
}
```
其中`quickRecycleScrapView`方法最终也是调用了`recycleViewHolderInternal`，而recycleCachedViewAt是直接将ViewHolder从mCachedViews中移除，添加至RecycledViewPoo中。

### 第三个缓存方法：getViewForPositionAndType
```java
if (holder == null && mViewCacheExtension != null) {  
    // We are NOT sending the offsetPosition because LayoutManager does not  
    // know it.    final View view = mViewCacheExtension  
            .getViewForPositionAndType(this, position, type);  
    if (view != null) {  
        holder = getChildViewHolder(view);  
        if (holder == null) {  
            throw new IllegalArgumentException("getViewForPositionAndType returned"  
                    + " a view which does not have a ViewHolder"  
                    + exceptionLabel());  
        } else if (holder.shouldIgnore()) {  
            throw new IllegalArgumentException("getViewForPositionAndType returned"  
                    + " a view that is ignored. You must call stopIgnoring before"  
                    + " returning this view." + exceptionLabel());  
        }  
    }  
}
```
第三个缓存方法也称自定义缓存，是让开发者可以自行定义的缓存逻辑。

### 第四个缓存方法：getRecycledViewPool().getRecycledView(type)
```java
if (holder == null) { // fallback to pool  
    if (DEBUG) {  
        Log.d(TAG, "tryGetViewHolderForPositionByDeadline("  
                + position + ") fetching from shared pool");  
    }  
    holder = getRecycledViewPool().getRecycledView(type);  
    if (holder != null) {  
        holder.resetInternal();  
        if (FORCE_INVALIDATE_DISPLAY_LIST) {  
            invalidateDisplayListInt(holder);  
        }  
    }  
}
```
先一起来看一下RecycledViewPool的数据结构：
```java
public static class RecycledViewPool {  

    private static final int DEFAULT_MAX_SCRAP = 5;  
 
    static class ScrapData {  
        final ArrayList<ViewHolder> mScrapHeap = new ArrayList<>();  
        int mMaxScrap = DEFAULT_MAX_SCRAP;  
        long mCreateRunningAverageNs = 0;  
        long mBindRunningAverageNs = 0;  
    }  
  
    SparseArray<ScrapData> mScrap = new SparseArray<>();  
  
    private int mAttachCount = 0;
    ...
    }
```
可以看到其实就是用SparseArray存储ViewHolder的信息，而内部的key则是我们使用的ItemType。而value为ArrayList，需要注意的是ViewHolder是通过remove方法来获取的，再返回的同时进行删除操作。

### 第五个方法：创建 createViewHolder
在经历了四级缓存之后，还是没有获取到符合条件的ViewHolder那怎么办呢？我们需要创建它，终于到了我们平时中最常见的方法了。
```java
if (holder == null) {  
    holder = mAdapter.createViewHolder(RecyclerView.this, type);  
}

boolean bound = false;  
if (mState.isPreLayout() && holder.isBound()) {  
    // do not update unless we absolutely have to.  
    holder.mPreLayoutPosition = position;  
} else if (!holder.isBound() || holder.needsUpdate() || holder.isInvalid()) {  
    final int offsetPosition = mAdapterHelper.findPositionOffset(position);  
    bound = tryBindViewHolderByDeadline(holder, offsetPosition, position, deadlineNs);  

	final ViewGroup.LayoutParams lp = holder.itemView.getLayoutParams();  
	final LayoutParams rvLayoutParams;  
	if (lp == null) {  
	    rvLayoutParams = (LayoutParams) generateDefaultLayoutParams();  
	    holder.itemView.setLayoutParams(rvLayoutParams);  
	} else if (!checkLayoutParams(lp)) {  
	    rvLayoutParams = (LayoutParams) generateLayoutParams(lp);  
	    holder.itemView.setLayoutParams(rvLayoutParams);  
	} else {  
	    rvLayoutParams = (LayoutParams) lp;  
	}  
	rvLayoutParams.mViewHolder = holder;  
	rvLayoutParams.mPendingInvalidate = fromScrapOrHiddenOrCache && bound;  
	return holder;
}
```
这样可以保证接下来的方法一定有ViewHolder引用了，这时我们进行数据绑定，可以看到，当holder的状态是未绑定、需要更新或者无效的时候则调用`tryBindViewHolderByDeadline`进行数据绑定，`tryBindViewHolderByDeadline`其实就是调用了我们熟知的`adapter.bindViewHolder`方法进行数据绑定，然后进行params的初始化就不看了。
```java
private boolean tryBindViewHolderByDeadline(@NonNull ViewHolder holder, int offsetPosition,  
        int position, long deadlineNs) {  
    holder.mBindingAdapter = null;  
    holder.mOwnerRecyclerView = RecyclerView.this;  
    final int viewType = holder.getItemViewType();  
    long startBindNs = getNanoTime();  
	
    mAdapter.bindViewHolder(holder, offsetPosition);  
    long endBindNs = getNanoTime();  
    mRecyclerPool.factorInBindTime(holder.getItemViewType(), endBindNs - startBindNs);  
    attachAccessibilityDelegateOnBind(holder);  
    if (mState.isPreLayout()) {  
        holder.mPreLayoutPosition = position;  
    }  
    return true;  
}
```

最后一起看下滑动的时候RecyclerView的逻辑，view的滑动都从`onTouchEvent`的开始，
```java
case MotionEvent.ACTION_MOVE:

if (scrollByInternal(  
        canScrollHorizontally ? dx : 0,  
        canScrollVertically ? dy : 0,  
        e, TYPE_TOUCH)) {  
    getParent().requestDisallowInterceptTouchEvent(true);  
}

boolean scrollByInternal(int x, int y, MotionEvent ev, int type) {  
    ...
    if (mAdapter != null) {  
		...
        scrollStep(x, y, mReusableIntPair);  
		...
    }
}

void scrollStep(int dx, int dy, @Nullable int[] consumed) {  
    startInterceptRequestLayout();  
    onEnterLayoutOrScroll();  
  
    TraceCompat.beginSection(TRACE_SCROLL_TAG);  
    fillRemainingScrollValues(mState);  
  
    int consumedX = 0;  
    int consumedY = 0;  
    if (dx != 0) {  
        consumedX = mLayout.scrollHorizontallyBy(dx, mRecycler, mState);  
    }  
    if (dy != 0) {  
        consumedY = mLayout.scrollVerticallyBy(dy, mRecycler, mState);  
    }  
  
    TraceCompat.endSection();  
    repositionShadowingViews();  
  
    onExitLayoutOrScroll();  
    stopInterceptRequestLayout(false);  
  
    if (consumed != null) {  
        consumed[0] = consumedX;  
        consumed[1] = consumedY;  
    }  
}

//LinearLayoutManager
public int scrollVerticallyBy(int dy, RecyclerView.Recycler recycler,  
        RecyclerView.State state) {  
    if (mOrientation == HORIZONTAL) {  
        return 0;  
    }  
    return scrollBy(dy, recycler, state);  
}

int scrollBy(int delta, RecyclerView.Recycler recycler, RecyclerView.State state) {  
	...
    final int consumed = mLayoutState.mScrollingOffset  
            + fill(recycler, mLayoutState, state, false);  
	...
    return scrolled;  
}


```
又看到了我们熟悉的`fill`方法，还是会走到`layoutChunk`方法，调用`onLayoutChildren`。

总结一下，RecyclerView并不是一开始就调用`createViewHolder`去创建ViewHolder的，而是经过四级缓存机制获取，若都无满足条件的ViewHolder再进行创建，这样可以最大限度的保证ViewHolder的复用机制。
讲了这么多缓存相关的读取使用操作，但什么时候清除缓存呢？
## 相关文章
[每日一问 | 关于 RecyclerView$Adapter setHasStableIds(boolean)的一切-玩Android - wanandroid.com](https://wanandroid.com/wenda/show/15514)