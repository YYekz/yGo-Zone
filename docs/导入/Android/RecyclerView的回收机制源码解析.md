[[RecyclerView的绘制流程源码解析]]
[[RecyclerView的缓存机制源码解析]]

上两篇文章讲解了RecyclerView的绘制流程以及RecyclerView的四级缓存中“取”的逻辑，接下来一起来看下这些缓存的ViewHolder是什么时候添加到缓存中“存”的逻辑。
在说之前先清楚一些ViewHolder的基本概念。
#### 关于移除
1. detach：轻量级的移除。临时remove，但最终还是会被attach回去的，比如调用`fill`之前的临时回收；
2. remove：重量级、彻底的移除。和旧的ViewGroup彻底切断联系。
#### 关于flag
FLAG_TAG | 含义 | 出现场景
-|-|-
FLAG_INVALIDE | ViewHolder的数据失效了 | 1. 调用了setAdatper；2.调用了notifyDataSetChanged()；3.调用了invalidateItemDecorations()
FLAG_UPDATE | ViewHolder的数据需要更新（需要重新绑定数据） | 1. 上述的三种方法都会；2. notifyItemChanged()
FLAG_REMOVED| ViewHolder已经被移除|调用了notifyItemRemoved()
FLAG_BOUND|ViewHolder已经绑定了某个位置的Item数据，数据有效|onBindViewHolder

还是回到那个我们最熟悉的方法中去`onLayoutChildren`，在方法中当找到锚点时，会调用`detachAndScrapAttachedViews`。它其实就是和ViewHolder相关的一个方法，在列表项填充之前需要先全部回收，然后重新`fill`，只是回收到的集合可能不一致。
```java
public void detachAndScrapAttachedViews(@NonNull Recycler recycler) {  
    final int childCount = getChildCount();  
    for (int i = childCount - 1; i >= 0; i--) {  
        final View v = getChildAt(i);  
        scrapOrRecycleView(recycler, i, v);  
    }  
}

private void scrapOrRecycleView(Recycler recycler, int index, View view) {  
    final ViewHolder viewHolder = getChildViewHolderInt(view);  
    if (viewHolder.shouldIgnore()) {  
        if (DEBUG) {  
            Log.d(TAG, "ignoring view " + viewHolder);  
        }  
        return;  
    }  
    if (viewHolder.isInvalid() && !viewHolder.isRemoved()  
            && !mRecyclerView.mAdapter.hasStableIds()) {  
        removeViewAt(index);  
        //重点方法
        recycler.recycleViewHolderInternal(viewHolder);  
    } else {  
        detachViewAt(index);  
        //重点方法
        recycler.scrapView(view);  
        mRecyclerView.mViewInfoStore.onViewDetached(viewHolder);  
    }  
}

```
在上一章我们讲到回收的统一入口是这两个重点方法，现在我们来详细的说明下这两个方法的异同：
1. `recycleViewHolderInternal`：当ViewHolder失效但是没有被完全移除，则直接删除该项进入mCachedViews或者是回收池；
2. `scrapView`：detach该项并进入scrap集合中，并没有真正的删除该项。
### recycleViewHolderInternal
看下代码进行具体分析：
```java
void recycleViewHolderInternal(ViewHolder holder) {  
	...
    if (forceRecycle || holder.isRecyclable()) {  
        if (mViewCacheMax > 0  
                && !holder.hasAnyOfTheFlags(ViewHolder.FLAG_INVALID  
                | ViewHolder.FLAG_REMOVED  
                | ViewHolder.FLAG_UPDATE  
                | ViewHolder.FLAG_ADAPTER_POSITION_UNKNOWN)) {  
            // Retire oldest cached view  
            int cachedViewSize = mCachedViews.size();  
            if (cachedViewSize >= mViewCacheMax && cachedViewSize > 0) {  
                recycleCachedViewAt(0);  
                cachedViewSize--;  
            }  
            ...  
            mCachedViews.add(targetCacheIndex, holder);  
            cached = true;  
        }  
        if (!cached) {  
            addViewHolderToRecycledViewPool(holder, true);  
            recycled = true;  
        }  
    } 
    ... 
}

void recycleCachedViewAt(int cachedViewIndex) {  
    ViewHolder viewHolder = mCachedViews.get(cachedViewIndex);  
    addViewHolderToRecycledViewPool(viewHolder, true);  
    mCachedViews.remove(cachedViewIndex);  
}
```
可以看到当view没有失效且没有移除且没有更新且位置确定时会被添加到mCachedViews中。
> 前文讲到过mCachedViews的默认缓存大小为2，当已有两个元素时则使用FIFO的原则，将表头的元素存入缓存池RecycledPool中，然后再把新的元素存入mCachedViews中；

mCachedViews和RecycledViewPool的存储规则是互斥的，通过cached进行判断，一个元素不可能同时存在于mCachedViews和RecycledViewPool中。

看一下为RecycledViewPool添加元素的方法：
```java
public void putRecycledView(ViewHolder scrap) {  
    final int viewType = scrap.getItemViewType();  
    final ArrayList<ViewHolder> scrapHeap = getScrapDataForType(viewType).mScrapHeap;  
    if (mScrap.get(viewType).mMaxScrap <= scrapHeap.size()) {  
        return;  
    }   
    scrap.resetInternal();  
    scrapHeap.add(scrap);  
}
```
其实前文已经讲过，Pool的缓冲池默认大小是按照ItemType来区分的，每个Type最多存储5个ViewHolder，超过的则会直接丢弃而不进行存储；
其中`scrap.resetInternal()`会将ViewHolder的信息全部移除（payloads、flags等等），因此下次使用的时候需要重新绑定数据。

### scrapView
```java
void scrapView(View view) {  
    final ViewHolder holder = getChildViewHolderInt(view);  
    if (holder.hasAnyOfTheFlags(ViewHolder.FLAG_REMOVED | ViewHolder.FLAG_INVALID)  
            || !holder.isUpdated() || canReuseUpdatedViewHolder(holder)) {  
        holder.setScrapContainer(this, false);  
        mAttachedScrap.add(holder);  
    } else {  
        if (mChangedScrap == null) {  
            mChangedScrap = new ArrayList<ViewHolder>();  
        }  
        holder.setScrapContainer(this, true);  
        mChangedScrap.add(holder);  
    }  
}
```
逻辑很简单，就是判断如果ViewHolder被移除、无效、没动画、不需要更新可重用（默认为true）的时候，就会利用mAttachedScrap进行保存，否则就用mChangedScrap进行保存，一般默认情况下我们用到的都是mAttachedScrap。

当我们在滑动RecyclerView时，会调用到`fill`方法，前文已经讲过，但在fill方法中有一段逻辑我们当时没说，现在一起来看下。
```java
int fill(RecyclerView.Recycler recycler, LayoutState layoutState,  
        RecyclerView.State state, boolean stopOnFocusable) {
        ...
        if (layoutState.mScrollingOffset != LayoutState.SCROLLING_OFFSET_NaN) {  
	        ...
	        recycleByLayoutState(recycler, layoutState);  
        }
	...
    }
    
private void recycleByLayoutState(RecyclerView.Recycler recycler, LayoutState layoutState) {  
    if (!layoutState.mRecycle || layoutState.mInfinite) {  
        return;  
    }  
    int scrollingOffset = layoutState.mScrollingOffset;  
    int noRecycleSpace = layoutState.mNoRecycleSpace;  
    if (layoutState.mLayoutDirection == LayoutState.LAYOUT_START) {  
        recycleViewsFromEnd(recycler, scrollingOffset, noRecycleSpace);  
    } else {  
        recycleViewsFromStart(recycler, scrollingOffset, noRecycleSpace);  
    }  
}

private void recycleViewsFromStart(RecyclerView.Recycler recycler, int scrollingOffset,  
        int noRecycleSpace) {    
    final int limit = scrollingOffset - noRecycleSpace;  
    final int childCount = getChildCount();  
    if (mShouldReverseLayout) {  
        for (int i = childCount - 1; i >= 0; i--) {  
            View child = getChildAt(i);  
            if (mOrientationHelper.getDecoratedEnd(child) > limit  
                    || mOrientationHelper.getTransformedEndWithDecoration(child) > limit) {  
                // stop here  
                recycleChildren(recycler, childCount - 1, i);  
                return;            
                }  
        }  
    } else {  
        for (int i = 0; i < childCount; i++) {  
            View child = getChildAt(i);  
            if (mOrientationHelper.getDecoratedEnd(child) > limit  
                    || mOrientationHelper.getTransformedEndWithDecoration(child) > limit) {  
                // stop here  
                recycleChildren(recycler, 0, i);  
                return;           
                 }  
        }  
    }  
}

private void recycleChildren(RecyclerView.Recycler recycler, int startIndex, int endIndex) {  
	...
    if (endIndex > startIndex) {  
        for (int i = endIndex - 1; i >= startIndex; i--) {  
            removeAndRecycleViewAt(i, recycler);  
        }  
    } else {  
        for (int i = startIndex; i > endIndex; i--) {  
            removeAndRecycleViewAt(i, recycler);  
        }  
    }  
}

public void removeAndRecycleViewAt(int index, @NonNull Recycler recycler) {  
    final View view = getChildAt(index);  
    removeViewAt(index);  
    recycler.recycleView(view);  
}

public void recycleView(@NonNull View view) {  
    ViewHolder holder = getChildViewHolderInt(view);  
    if (holder.isTmpDetached()) {  
        removeDetachedView(view, false);  
    }  
    if (holder.isScrap()) {  
        holder.unScrap();  
    } else if (holder.wasReturnedFromScrap()) {  
        holder.clearReturnedFromScrapFlag();  
    }  
    recycleViewHolderInternal(holder);  
    }  
}
```
可以看到当我们向上滑动的时候，会调用到`recycleViewsFromStart`，会从顶部开始回收View。而往下滑时则是从底部开始回收View。我们说下往上滑，可以看到最终还是调用了`recycleViewHolderInternal`方法，而由于滑动屏幕过程中并不会给ViewHolder设置状态，所以此时ViewHolder满足存入mCachedViews的条件。所以当下一次需要使用时，可以直接从mCachedViews中取出而无需绑定数据。
其实从这里可以看出，scrap缓存中缓存的数据是和屏幕滑动完全没关系的一套缓存，他只是在`onLayout`的时候再调用`fill`之前把所有的ViewHolder都临时缓存起来，之后再重新attach上去，仅仅是和布局有关的临时缓存。实际上在布局方法结束之后，scrap就会被清空，在布局结束之后mAttachedScrap和mChangedScrap中不该有任何的数据。在`dispatchLayoutStep3`中能看到。
```java
private void dispatchLayoutStep3() {
	...
	mLayout.removeAndRecycleScrapInt(mRecycler);
	...
}

void removeAndRecycleScrapInt(Recycler recycler) {  
    final int scrapCount = recycler.getScrapCount();  
    // Loop backward, recycler might be changed by removeDetachedView()  
    for (int i = scrapCount - 1; i >= 0; i--) {  
        final View scrap = recycler.getScrapViewAt(i);  
        final ViewHolder vh = getChildViewHolderInt(scrap);  
        if (vh.shouldIgnore()) {  
            continue;  
        }      
        vh.setIsRecyclable(false);  
        if (vh.isTmpDetached()) {  
            mRecyclerView.removeDetachedView(scrap, false);  
        }  
        if (mRecyclerView.mItemAnimator != null) {  
            mRecyclerView.mItemAnimator.endAnimation(vh);  
        }  
        vh.setIsRecyclable(true);  
        recycler.quickRecycleScrapView(scrap);  
    }  
    recycler.clearScrap();  
    if (scrapCount > 0) {  
        mRecyclerView.invalidate();  
    }  
}

void clearScrap() {  
    mAttachedScrap.clear();  
    if (mChangedScrap != null) {  
        mChangedScrap.clear();  
    }  
}
```


## RecyclerView的刷新逻辑

我们平时使用时，最常见的刷新方法就是`notifyDataSetChanged`、`notifyItemChanged`、`notfyItemRemove`，那就一起来看一下这些方法的源码到底做了些什么吧。
### notifyDataSetChanged
```java
//adapter
public final void notifyDataSetChanged() {  
    mObservable.notifyChanged();  
}

//AdapterDataObservable
public void notifyChanged() {  
    for (int i = mObservers.size() - 1; i >= 0; i--) {  
        mObservers.get(i).onChanged();  
    }  
}

//RecyclerViewDataObserver
public void onChanged() {  
    assertNotInLayoutOrScroll(null);  
    mState.mStructureChanged = true;  
  
    processDataSetCompletelyChanged(true);  
    if (!mAdapterHelper.hasPendingUpdates()) {  
        requestLayout();  
    }  
}

```
从源码跟踪下来可以看到主要分为两个操作：
1. 数据改变之前的预处理操作:`processDataSetCompletelyChanged`；
2. 如果没有其余即将更新的操作，则重新布局：`requestLayout`；

那先来看下预处理操作方法：
```java
void processDataSetCompletelyChanged(boolean dispatchItemsChanged) {  
    mDispatchItemsChangedEvent |= dispatchItemsChanged;  
    mDataSetHasChangedAfterLayout = true;  
    markKnownViewsInvalid();  
}

void markKnownViewsInvalid() {  
    final int childCount = mChildHelper.getUnfilteredChildCount();  
    for (int i = 0; i < childCount; i++) {  
        final ViewHolder holder = getChildViewHolderInt(mChildHelper.getUnfilteredChildAt(i));  
        if (holder != null && !holder.shouldIgnore()) {  
	        //给每一个ViewHolder添加两个flag：有更新和已失效
            holder.addFlags(ViewHolder.FLAG_UPDATE | ViewHolder.FLAG_INVALID);  
        }  
    }  
    markItemDecorInsetsDirty();  
    mRecycler.markKnownViewsInvalid();  
}

void markKnownViewsInvalid() {  
    final int cachedCount = mCachedViews.size();  
    for (int i = 0; i < cachedCount; i++) {  
        final ViewHolder holder = mCachedViews.get(i);  
        if (holder != null) {  
	        //给每一个mCacheViews的ViewHolder也添加两个flag：有更新和已失效
            holder.addFlags(ViewHolder.FLAG_UPDATE | ViewHolder.FLAG_INVALID);  
            holder.addChangePayload(null);  
        }  
    }  
  
    if (mAdapter == null || !mAdapter.hasStableIds()) {  
        recycleAndClearCachedViews();  
    }  
}

void recycleAndClearCachedViews() {  
    final int count = mCachedViews.size();  
    for (int i = count - 1; i >= 0; i--) {  
        recycleCachedViewAt(i);  
    }  
    //清空mCachedViews
    mCachedViews.clear();  
    if (ALLOW_THREAD_GAP_WORK) {  
        mPrefetchRegistry.clearPrefetchPositions();  
    }  
}

void recycleCachedViewAt(int cachedViewIndex) {  
    ViewHolder viewHolder = mCachedViews.get(cachedViewIndex);  
    //将mCacheViews里的所有ViewHolder添加到RecycledViewPool中
    addViewHolderToRecycledViewPool(viewHolder, true);  
    mCachedViews.remove(cachedViewIndex);  
}
```
可以看到这个与处理的方法其实主要做了这几件事儿：
1. 找到当前所有的ViewHolder添加上这两个标签：FLAG_UPDATE和FLAG_INVALID；
2. 将mCachedViews中所有的ViewHolder添加上这两个标签：FLAG_UPDATE和FLAG_INVALID；
3. 将mCachedViews中所有ViewHolder添加至RecycledViewPool中；
4. 将mCachedViews清空；
5. 最后调用`requestLayout`方法重新布局；
因为所有的ViewHolder都已添加标记FLAG_INVALID，因此之后使用时都需要重新绑定数据。（这也是我们常说的所谓使用`notifyDataSetChanged`刷新性能消耗大的原因）

### notifyItemChanged
那既然`notifyDataSetChanged`方法刷新性能消耗大，那怎么样优化呢？可以找到对应需要更新的item位置，使用`notifyItemChanged`更新单个item不就好了！
那我们来一起看看源码中`notifyItemChanged`的实现。
```java

//adapter
public final void notifyItemChanged(int position) {  
    mObservable.notifyItemRangeChanged(position, 1);  
}

public void notifyItemRangeChanged(int positionStart, int itemCount) {  
    notifyItemRangeChanged(positionStart, itemCount, null);  
}

//AdapterDataObservable
public void notifyItemRangeChanged(int positionStart, int itemCount,  
        @Nullable Object payload) {  
    for (int i = mObservers.size() - 1; i >= 0; i--) {  
        mObservers.get(i).onItemRangeChanged(positionStart, itemCount, payload);  
    }  
}

@Override  
public void onItemRangeChanged(int positionStart, int itemCount, Object payload) {  
    assertNotInLayoutOrScroll(null);  
    if (mAdapterHelper.onItemRangeChanged(positionStart, itemCount, payload)) {  
        triggerUpdateProcessor();  
    }  
}

boolean onItemRangeChanged(int positionStart, int itemCount, Object payload) {  
    if (itemCount < 1) {  
        return false;  
    }  
    mPendingUpdates.add(obtainUpdateOp(UpdateOp.UPDATE, positionStart, itemCount, payload));  
    mExistingUpdateTypes |= UpdateOp.UPDATE;  
    return mPendingUpdates.size() == 1;  
}

void triggerUpdateProcessor() {  
    if (POST_UPDATES_ON_ANIMATION && mHasFixedSize && mIsAttached) {  
        ViewCompat.postOnAnimation(RecyclerView.this, mUpdateChildViewsRunnable);  
    } else {  
        mAdapterUpdateDuringMeasure = true;  
        requestLayout();  
    }  
}
```
从上面的代码逻辑来看，就是给mPendingUpdates中添加了这次更新操作，记录了当前更新的位置。
在[[RecyclerView的绘制流程源码解析]]中我们提到了`dispatchLayoutStep1`中有一条调用链路是会用来处理变更的，我们一起来再看下。
```java
private void dispatchLayoutStep1() {
	...
	processAdapterUpdatesAndSetAnimationFlags();
	...
}

private void processAdapterUpdatesAndSetAnimationFlags() {  
    if (mDataSetHasChangedAfterLayout) {        
        mAdapterHelper.reset();  
        if (mDispatchItemsChangedEvent) {  
            mLayout.onItemsChanged(this);  
        }  
    }  
    if (predictiveItemAnimationsEnabled()) {  
        mAdapterHelper.preProcess();  
    } else {  
        mAdapterHelper.consumeUpdatesInOnePass();  
    }  
}

void consumeUpdatesInOnePass() {   
    consumePostponedUpdates();  
    final int count = mPendingUpdates.size();  
    for (int i = 0; i < count; i++) {  
        UpdateOp op = mPendingUpdates.get(i);  
        switch (op.cmd) {  
            case UpdateOp.ADD:  
                mCallback.onDispatchSecondPass(op);  
                mCallback.offsetPositionsForAdd(op.positionStart, op.itemCount);  
                break;            
            case UpdateOp.REMOVE:  
                mCallback.onDispatchSecondPass(op);  
                mCallback.offsetPositionsForRemovingInvisible(op.positionStart, op.itemCount);  
                break;            
            case UpdateOp.UPDATE:  
                mCallback.onDispatchSecondPass(op);  
                mCallback.markViewHoldersUpdated(op.positionStart, op.itemCount, op.payload);  
                break;            
            case UpdateOp.MOVE:  
                mCallback.onDispatchSecondPass(op);  
                mCallback.offsetPositionsForMove(op.positionStart, op.itemCount);  
                break;        }  
        if (mOnItemProcessedCallback != null) {  
            mOnItemProcessedCallback.run();  
        }  
    }  
    recycleUpdateOpsAndClearList(mPendingUpdates);  
    mExistingUpdateTypes = 0;  
}

@Override  
public void markViewHoldersUpdated(int positionStart, int itemCount, Object payload) {  
    viewRangeUpdate(positionStart, itemCount, payload);  
    mItemsChanged = true;  
}

void viewRangeUpdate(int positionStart, int itemCount, Object payload) {  
    final int childCount = mChildHelper.getUnfilteredChildCount();  
    final int positionEnd = positionStart + itemCount;  
  
    for (int i = 0; i < childCount; i++) {  
        final View child = mChildHelper.getUnfilteredChildAt(i);  
        final ViewHolder holder = getChildViewHolderInt(child);  
        if (holder == null || holder.shouldIgnore()) {  
            continue;  
        }  
        if (holder.mPosition >= positionStart && holder.mPosition < positionEnd) {         
            holder.addFlags(ViewHolder.FLAG_UPDATE);  
            holder.addChangePayload(payload);  
            ((LayoutParams) child.getLayoutParams()).mInsetsDirty = true;  
        }  
    }  
    mRecycler.viewRangeUpdate(positionStart, itemCount);  
}

void viewRangeUpdate(int positionStart, int itemCount) {  
    final int positionEnd = positionStart + itemCount;  
    final int cachedCount = mCachedViews.size();  
    for (int i = cachedCount - 1; i >= 0; i--) {  
        final ViewHolder holder = mCachedViews.get(i);  
        if (holder == null) {  
            continue;  
        }  
  
        final int pos = holder.mPosition;  
        if (pos >= positionStart && pos < positionEnd) {  
            holder.addFlags(ViewHolder.FLAG_UPDATE);  
            recycleCachedViewAt(i);  
		}
    }  
}
```
主要的步骤就是：
1. RecyclerView出于数据改变范围内的ViewHolder被标记FLAG_UPDATE；
2. 有数据更新且已缓存在mCachedViews中的ViewHolder被标记FLAG_UPDATE，并且被RecycledViewPool回收;
3. 最终，有FLAG_UPDATE的ViewHolder需要重新绑定数据。

### notifyItemRemove
其实大概想一下应该是和`notifyItemChanged`类似的操作和调用链路，唯一不同的地方是将给mPendingUpdates中添加的更新操作替换为移除操作，看下最后分发到remove流程的代码。
```java
@Override  
public void offsetPositionsForRemovingInvisible(int start, int count) {  
    offsetPositionRecordsForRemove(start, count, true);  
    mItemsAddedOrRemoved = true;  
    mState.mDeletedInvisibleItemCountSincePreviousLayout += count;  
}

void offsetPositionRecordsForRemove(int positionStart, int itemCount,  
        boolean applyToPreLayout) {  
    final int positionEnd = positionStart + itemCount;  
    final int childCount = mChildHelper.getUnfilteredChildCount();  
    for (int i = 0; i < childCount; i++) {  
        final ViewHolder holder = getChildViewHolderInt(mChildHelper.getUnfilteredChildAt(i));  
        if (holder != null && !holder.shouldIgnore()) {  
	        //当viewHolder的下标大于被删除的item时，调用offsetPosition
            if (holder.mPosition >= positionEnd) {  
                holder.offsetPosition(-itemCount, applyToPreLayout);  
                mState.mStructureChanged = true;  
            } else if (holder.mPosition >= positionStart) {  
	            //当viewholder的下表在被删除item的范围内时，调用该方法给item添加标记FLAG_REMOVE。
                holder.flagRemovedAndOffsetPosition(positionStart - 1, -itemCount,  
                        applyToPreLayout);  
                mState.mStructureChanged = true;  
            }  
        }  
    }  
    mRecycler.offsetPositionRecordsForRemove(positionStart, itemCount, applyToPreLayout);  
    requestLayout();  
}

void flagRemovedAndOffsetPosition(int mNewPosition, int offset, boolean applyToPreLayout) {  
    addFlags(ViewHolder.FLAG_REMOVED);  
    offsetPosition(offset, applyToPreLayout);  
    mPosition = mNewPosition;  
}  
  
void offsetPosition(int offset, boolean applyToPreLayout) {  
    if (mOldPosition == NO_POSITION) {  
        mOldPosition = mPosition;  
    }  
    if (mPreLayoutPosition == NO_POSITION) {  
        mPreLayoutPosition = mPosition;  
    }  
    if (applyToPreLayout) {  
        mPreLayoutPosition += offset;  
    }  
    mPosition += offset;  
    if (itemView.getLayoutParams() != null) {  
        ((LayoutParams) itemView.getLayoutParams()).mInsetsDirty = true;  
    }  
}
```
