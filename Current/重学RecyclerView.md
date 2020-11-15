## 重学RecyclerView

问题: 

1. RecyclerView的布局和动画
2. RecyclerView的复用机制
3. RecyclerView事件分发

### 1.RecyclerView的布局和动画

要了解RecyclerView的布局，依然和其他控件的布局一样，从onMeasure开始：

```java
@Override
    protected void onMeasure(int widthSpec, int heightSpec) {
        //... 没有LayoutManager的情况
        if (mLayout.isAutoMeasureEnabled()) {
            //AutoMeasure模式下的测量，LinearLayoutManager默认会走这个分支
            final int widthMode = MeasureSpec.getMode(widthSpec);
            final int heightMode = MeasureSpec.getMode(heightSpec);
            
            //这一行就是简单地基于RecyclerView的原定高宽来设定了一个初始大小，
            mLayout.onMeasure(mRecycler, mState, widthSpec, heightSpec);

            //如果是确定高度或者没有数据，直接返回，这里RecyclerView的高度和宽度如果是matchParent的话，就没有下面提前布局的事了
            final boolean measureSpecModeIsExactly =
                    widthMode == MeasureSpec.EXACTLY && heightMode == MeasureSpec.EXACTLY;
            if (measureSpecModeIsExactly || mAdapter == null) {
                return;
            }
		   //提前布局流程的开始
            if (mState.mLayoutStep == State.STEP_START) {
                dispatchLayoutStep1();
            }
            // set dimensions in 2nd step. Pre-layout should happen with old dimensions for
            // consistency
            mLayout.setMeasureSpecs(widthSpec, heightSpec);
            mState.mIsMeasuring = true;
            //这里不再是preLayout状态了，再执行一遍LayoutManager的onLayoutChild，在测量完之后会施行实际的布局动作，并将layoutStep改为State.STEP_ANIMATIONS
            dispatchLayoutStep2();

            //根据当前已布局的子项边界来确定维度
            mLayout.setMeasuredDimensionFromChildren(widthSpec, heightSpec);

            //省略重布局的情况
        } else {
            //RecyclerView定长的场景，不需要管其内容，直接进行测量
            if (mHasFixedSize) {
                mLayout.onMeasure(mRecycler, mState, widthSpec, heightSpec);
                return;
            }
            // custom onMeasure 大概是表示因为有列表项变化触发的重布局
            if (mAdapterUpdateDuringMeasure) {
                //mInterceptRequestLayoutDepth标志位+1 如果这个标志位大于1，则会禁止requestLayout调用
                //触发重布局,这个机制的直接影响是延迟回调dispatchLayout方法 下面的很多方法走着走着就会调到requestLayout
                startInterceptRequestLayout();
                //设置进入布局或者滚动状态的标志位，如果在这个标志位存在时进行数据变化操作会抛出异常
                onEnterLayoutOrScroll();
                //判断当前列表项变化的类型，并准备好相应的动画类型。这个方法内会顺便确定viewHolder它们在此次重绘中的position值变化
                processAdapterUpdatesAndSetAnimationFlags();
                onExitLayoutOrScroll();
			   //省略下面的一大段代码
        }
    }
```

然后是常规的onLayout，这里实际的逻辑在dispatchLayout里：

```java
void dispatchLayout() {
       //省略没有Adapter和LayoutManager的情况
        mState.mIsMeasuring = false;
    	//如果在onMeasure阶段进行过预布局，这里会跳过布局，直接走到第三步
        if (mState.mLayoutStep == State.STEP_START) {
            //处理子项变化，获取布局高度和标记各项重布局前的状态 如果此时高度不定，进行预布局
            dispatchLayoutStep1();
            //这里把LayoutManager的高度和宽度都设成EXACTLY，宽高取RecyclerView的高宽
            mLayout.setExactMeasureSpecsFrom(this);
            dispatchLayoutStep2();
        } else if (mAdapterHelper.hasUpdates() || mLayout.getWidth() != getWidth()
                || mLayout.getHeight() != getHeight()) {
            // First 2 steps are done in onMeasure but looks like we have to run again due to
            // changed size.
            mLayout.setExactMeasureSpecsFrom(this);
            dispatchLayoutStep2();
        } else {
            // always make sure we sync them (to ensure mode is exact)
            mLayout.setExactMeasureSpecsFrom(this);
        }
        dispatchLayoutStep3();
    }
```



dispatchLayoutStep1:

```java
/**
     * The first step of a layout where we;
     * - process adapter updates
     * - decide which animation should run
     * - save information about current views
     * - If necessary, run predictive layout and save its information
     */
    private void dispatchLayoutStep1() {
        //置一堆状态位
        mState.assertLayoutStep(State.STEP_START);
        fillRemainingScrollValues(mState);
        mState.mIsMeasuring = false;
        //仔细看这边的各种方法调用操作和自定义测量的过程如出一辙
        startInterceptRequestLayout();
        mViewInfoStore.clear();
        onEnterLayoutOrScroll();
        //处理本次重布局的变动子项，判断它们是增是减，并进行被影响项的移位
        processAdapterUpdatesAndSetAnimationFlags();
        saveFocusInfo();
        mState.mTrackOldChangeHolders = mState.mRunSimpleAnimations && mItemsChanged;
        mItemsAddedOrRemoved = mItemsChanged = false;
        //如果上面决定需要动画，则这里就会是true
        mState.mInPreLayout = mState.mRunPredictiveAnimations;
        mState.mItemCount = mAdapter.getItemCount();
        findMinMaxChildLayoutPositions(mMinMaxLayoutPositions);
		//为动画实现做准备
        if (mState.mRunSimpleAnimations) {
            // Step 0: Find out where all non-removed items are, pre-layout
            int count = mChildHelper.getChildCount();
            for (int i = 0; i < count; ++i) {
                final ViewHolder holder = getChildViewHolderInt(mChildHelper.getChildAt(i));
                if (holder.shouldIgnore() || (holder.isInvalid() && !mAdapter.hasStableIds())) {
                    continue;
                }
                //这里的默认实现是简单地保存当前viewHolder的位置
                final ItemHolderInfo animationInfo = mItemAnimator
                        .recordPreLayoutInformation(mState, holder,
                                ItemAnimator.buildAdapterChangeFlagsForAnimations(holder),
                                holder.getUnmodifiedPayloads());
                //这个方法会给ViewHolder加一个FLAG_PRE标记，表示这个子项是重布局之前就存在的，这种情况就只会对这个子项运行disappear或者presistent动画
                mViewInfoStore.addToPreLayout(holder, animationInfo);
                if (mState.mTrackOldChangeHolders && holder.isUpdated() && !holder.isRemoved()
                        && !holder.shouldIgnore() && !holder.isInvalid()) {
                    long key = getChangedHolderKey(holder);
                    //如果有一个ViewHolder满足需要展示动画且有所变化，则加进mOldChangeHolders列表
                    mViewInfoStore.addToOldChangeHolders(key, holder);
                }
            }
        }
        if (mState.mRunPredictiveAnimations) {
            //step1.预排列所有可见的子项
            // Save old positions so that LayoutManager can run its mapping logic.
            saveOldPositions();
            final boolean didStructureChange = mState.mStructureChanged;
            mState.mStructureChanged = false;
            // temporarily disable flag because we are asking for previous layout
            //调用LayoutManager对子项进行预排列
            mLayout.onLayoutChildren(mRecycler, mState);
            mState.mStructureChanged = didStructureChange;

            //省略过一段针对非preLayout对象的意义不明的循环
            // we don't process disappearing list because they may re-appear in post layout pass.
            clearOldPositions();
        } else {
            clearOldPositions();
        }
        onExitLayoutOrScroll();
        stopInterceptRequestLayout(false);
        mState.mLayoutStep = State.STEP_LAYOUT;
    }
```

dispatchLayoutStep2:

```java
private void dispatchLayoutStep2() {
        startInterceptRequestLayout();
        onEnterLayoutOrScroll();
    	//判断是不是布局状态或者动画执行状态
        mState.assertLayoutStep(State.STEP_LAYOUT | State.STEP_ANIMATIONS);
    	//通过一个循环消费所有mPendingUpdates内的子项变化（改变剩余子项的index之类的操作），前者的保存项来自于熟悉的notifyItemChanged方法，如果之前走过preProcess()方法，则这里只会处理掉mPostponedList里的数据(调用LayoutManager的子项变化监听)
        mAdapterHelper.consumeUpdatesInOnePass();
        mState.mItemCount = mAdapter.getItemCount();
        mState.mDeletedInvisibleItemCountSincePreviousLayout = 0;

        // Step 2: Run layout
        mState.mInPreLayout = false;
    	//再一次对子项进行布局，和dispatchLayoutStep1差不多 但是这里不是预布局了
        mLayout.onLayoutChildren(mRecycler, mState);

        mState.mStructureChanged = false;
        mPendingSavedState = null;

        //再次检查是否需要运行动画
        mState.mRunSimpleAnimations = mState.mRunSimpleAnimations && mItemAnimator != null;
        mState.mLayoutStep = State.STEP_ANIMATIONS;
        onExitLayoutOrScroll();
        stopInterceptRequestLayout(false);
}
```

dispatchLayoutStep3：

```java
private void dispatchLayoutStep3(){
    //布局已经结束 重置State回开始状态
    mState.assertLayoutStep(State.STEP_ANIMATIONS);
    startInterceptRequestLayout();
    onEnterLayoutOrScroll();
    mState.mLayoutStep = State.STEP_START;
    if (mState.mRunSimpleAnimations) {
        for (int i = mChildHelper.getChildCount() - 1; i >= 0; i--) {
            ViewHolder holder = getChildViewHolderInt(mChildHelper.getChildAt(i));
            if (holder.shouldIgnore()) {
                continue;
            }
            //这里的key在不指定唯一性id时，就是holder的position
            long key = getChangedHolderKey(holder);
            final ItemHolderInfo animationInfo = mItemAnimator
                .recordPostLayoutInformation(mState, holder);
            //这里的oldViewHolder就是dispatchLayoutStep1中放进列表的待展示动画项
            ViewHolder oldChangeViewHolder = mViewInfoStore.getFromOldChangeHolders(key);
            if (oldChangeViewHolder != null && !oldChangeViewHolder.shouldIgnore()) {
                final boolean oldDisappearing = mViewInfoStore.isDisappearing(
                            oldChangeViewHolder);
                final boolean newDisappearing = mViewInfoStore.isDisappearing(holder);
                if (oldDisappearing && oldChangeViewHolder == holder) {
                    // 如果新的ViewHolder和旧的ViewHolder是一个，且这个子项已经要移出屏幕，则启用新子项的隐藏动画而不是change动画（这里调用方法的目的不是为了加标志位，因为DISAPPEARED标记优先级更高，应该只是为了更新ViewHolder的信息）
                    mViewInfoStore.addToPostLayout(holder, animationInfo);
                } else {
                    final ItemHolderInfo preInfo = mViewInfoStore.popFromPreLayout(
                             oldChangeViewHolder);
                    //给当前子项加一个FLAG_POST标记，表示它在重布局后仍存在，可以展示相应动画
                    mViewInfoStore.addToPostLayout(holder, animationInfo);
                    ItemHolderInfo postInfo = mViewInfoStore.popFromPostLayout(holder);
                    if (preInfo == null) {
                        //提示开发者导致没有找到该子项的变化状态可能出现的错误原因
                        handleMissingPreInfoForChangeError(key, holder, oldChangeViewHolder);
                    } else {
                        //这个方法里，如果旧的子项即将消失，则将其移出复用View队列，对LayoutManager不可见，但依然作为RecyclerView的子项之一专用于动画展示，同时回调animateChange方法
                        animateChange(oldChangeViewHolder, holder, preInfo, postInfo,
                                oldDisappearing, newDisappearing);
                    }
                }
            } else {
                //如果没有旧子项，就直接标记当前子项
                mViewInfoStore.addToPostLayout(holder, animationInfo);
            }
        }

        // 上面的循环已经往mLayoutHolderMap里加入了各种标记为（DISAPPEARED、PRE、POST等，会根据这些标记位的组合判断需要执行哪种动画），这个方法会遍历它，并根据其要执行的动画回调mViewInfoProcessCallback的各个方法，而后者就调用了ItemAnimator的各个animateXXX回调
        mViewInfoStore.process(mViewInfoProcessCallback);
    }
    //省略一堆标记位的重置位
	mLayout.onLayoutCompleted(mState);
    onExitLayoutOrScroll();
    stopInterceptRequestLayout(false);
    mViewInfoStore.clear();
    //如果布局后子项的范围产生了变动，通过滚动相关监听通知使用者
    if (didChildRangeChange(mMinMaxLayoutPositions[0], mMinMaxLayoutPositions[1])) {
        dispatchOnScrolled(0, 0);
    }
    recoverFocusFromState();
    resetFocusInfo();
}
```



onLayoutChildren:

```java
public void onLayoutChildren(RecyclerView.Recycler recycler, RecyclerView.State state) {
        // layout algorithm:
        // 1) by checking children and other variables, find an anchor coordinate and an anchor
        //  item position.
        // 2) fill towards start, stacking from bottom
        // 3) fill towards end, stacking from top
        // 4) scroll to fulfill requirements like stack from bottom.
        // create layout state
       	//省略若干延迟触发的逻辑
        ensureLayoutState();
        mLayoutState.mRecycle = false;
        // resolve layout direction
        resolveShouldLayoutReverse();

        //省略若干由于布局变化重定向焦点的逻辑
        // LLM may decide to layout items for "extra" pixels to account for scrolling target,
        // caching or predictive animations.

        //省略若干前后额外间距的计算
        int startOffset;
        int endOffset;
        final int firstLayoutDirection;
        if (mAnchorInfo.mLayoutFromEnd) {
            firstLayoutDirection = mShouldReverseLayout ? LayoutState.ITEM_DIRECTION_TAIL
                    : LayoutState.ITEM_DIRECTION_HEAD;
        } else {
            firstLayoutDirection = mShouldReverseLayout ? LayoutState.ITEM_DIRECTION_HEAD
                    : LayoutState.ITEM_DIRECTION_TAIL;
        }
	    //LLM的这一句是空实现
        onAnchorReady(recycler, state, mAnchorInfo, firstLayoutDirection);
    	//暂时将所有当前的View都丢进mAttachScrap集合里
        detachAndScrapAttachedViews(recycler);
        mLayoutState.mInfinite = resolveIsInfinite();
        mLayoutState.mIsPreLayout = state.isPreLayout();
        mLayoutState.mNoRecycleSpace = 0;
        if (mAnchorInfo.mLayoutFromEnd) {
            //省略从尾部开始布局的逻辑 就是下面逻辑的镜像
        }else{
            // fill towards start
            //初始化State的状态
            updateLayoutStateToFillStart(mAnchorInfo);
            mLayoutState.mExtraFillSpace = extraForStart;
            //事实上对View进行生命周期循环和布局的方法
            fill(recycler, mLayoutState, state, false);
            startOffset = mLayoutState.mOffset;
            final int firstElement = mLayoutState.mCurrentPosition;
            if (mLayoutState.mAvailable > 0) {
                extraForEnd += mLayoutState.mAvailable;
            }
            // fill towards end
            //如果上面的预布局已经耗尽了屏幕空间，这里就不会有什么结果，很快地返回
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
        }

        //省略一段对由于布局变化造成的可能有的间隙的修补
    	//非PreLayout状态下生效，如果存在一个将要加入的子项（ScrapList里），就提前将其加入到布局中去，然后把mLayoutState.mScrapList设成了null
        layoutForPredictiveAnimations(recycler, state, startOffset, endOffset);
        if (!state.isPreLayout()) {
            mOrientationHelper.onLayoutComplete();
        } else {
            mAnchorInfo.reset();
        }
        mLastStackFromEnd = mStackFromEnd;
    }
```

fill：

```java
int fill(RecyclerView.Recycler recycler, LayoutState layoutState,
            RecyclerView.State state, boolean stopOnFocusable) {
        //可布局的最大高度，如果是从头开始的布局，则这里是屏幕高度
        final int start = layoutState.mAvailable;
        //省略一个判断
        int remainingSpace = layoutState.mAvailable + layoutState.mExtraFillSpace;
        LayoutChunkResult layoutChunkResult = mLayoutChunkResult;
        while ((layoutState.mInfinite || remainingSpace > 0) && layoutState.hasMore(state)) {
            layoutChunkResult.resetInternal();
            if (RecyclerView.VERBOSE_TRACING) {
                TraceCompat.beginSection("LLM LayoutChunk");
            }
            //这里开始实际依次测量子项的大小
            layoutChunk(recycler, state, layoutState, layoutChunkResult);
            //在LinearLayout中 只有出现错误找不到View时才会置这个标志位进行中断
            if (layoutChunkResult.mFinished) {
                break;
            }
            layoutState.mOffset += layoutChunkResult.mConsumed * layoutState.mLayoutDirection;
            //累计被消耗的高度
            if (!layoutChunkResult.mIgnoreConsumed || layoutState.mScrapList != null
                    || !state.isPreLayout()) {
                layoutState.mAvailable -= layoutChunkResult.mConsumed;
                // we keep a separate remaining space because mAvailable is important for recycling
                remainingSpace -= layoutChunkResult.mConsumed;
            }

           //省略一段ScrollOffset相关的判断
            if (stopOnFocusable && layoutChunkResult.mFocusable) {
                break;
            }
        }
    	//返回这次填充消耗的高度
        return start - layoutState.mAvailable;
    }
```



layoutChunk:

```java
void layoutChunk(RecyclerView.Recycler recycler, RecyclerView.State state,
            LayoutState layoutState, LayoutChunkResult result) {
        //从缓存、回收池中获取一个View实例，或者最后通过createViewHolder来构造一个
        View view = layoutState.next(recycler);
        if (view == null) {
            result.mFinished = true;
            return;
        }
        RecyclerView.LayoutParams params = (RecyclerView.LayoutParams) view.getLayoutParams();
        if (layoutState.mScrapList == null) {
            if (mShouldReverseLayout == (layoutState.mLayoutDirection
                    == LayoutState.LAYOUT_START)) {
                //一般会走到这个分支，这里除了把view从之前的其他缓存集合中移除之外，最主要的是最后调到了RecyclerView的attachViewToParent方法，正式成为RecyclerView的子项
                addView(view);
            } else {
                addView(view, 0);
            }
        } else {/**省略**/}
    	//在这里调了子项的onMesure方法测量宽高
        measureChildWithMargins(view, 0, 0);
    	//这个参数表示了子项所占用的所有高度，便于fill对所有子项的使用空间进行计算
        result.mConsumed = mOrientationHelper.getDecoratedMeasurement(view);
        int left, top, right, bottom;
        if (mOrientation == VERTICAL) {
            if (isLayoutRTL()) {
                right = getWidth() - getPaddingRight();
                left = right - mOrientationHelper.getDecoratedMeasurementInOther(view);
            } else {
                left = getPaddingLeft();
                right = left + mOrientationHelper.getDecoratedMeasurementInOther(view);
            }
            if (layoutState.mLayoutDirection == LayoutState.LAYOUT_START) {
                bottom = layoutState.mOffset;
                top = layoutState.mOffset - result.mConsumed;
            } else {
                //省去倒置布局计算
            }
        } else {
            //省略纵向布局计算
        }
        // 累计计算了子项的各种盒模型大小，最后走了子项的onLayout
        layoutDecoratedWithMargins(view, left, top, right, bottom);
        // Consume the available space if the view is not removed OR changed
        if (params.isItemRemoved() || params.isItemChanged()) {
            result.mIgnoreConsumed = true;
        }
        result.mFocusable = view.hasFocusable();
    }
```

#### 布局小结与细节：

- **布局的过程中总会顺序调用dispatchLayoutStep1、2、3，三者的作用分别是：预布局，测量子项高度，确定布局子项位置变化和动画选择；根据第一步的结果进行布局动作；重置各种标志位，处理各子项的动画信息，并执行各动画回调。**

- Recycler的一级缓存是**mCacheViews**，其大小取决于`mViewCacheMax`参数（常规是2+N，N取决于LayoutManager的mPrefetchMaxCountObserved参数），当一个多余子项移出页面时，就会进入mCachedViews里，以便快速地重用（这个过程是预测性质的，根据一个deadlineNs来进行预判断应该缓存哪个ViewHolder，**Recycler**掌握着这个神奇的过程，包括布局时的计时）；第二级缓存就是**mRecyclerPool**，为RecycledViewPool类型，里面保存着一个SparseArray，以类型区分的形式保存ViewHolder数组，每种类型数组的上限来自ScrapData的mMaxScrap值，默认是5。当一级缓存满时，就会将ViewHolder放入二级缓存。

  > 理论上RecyclerView只需要一屏最大子项数量+2（即将消失和即将出现的子项）个ViewHolder就能完成整套循环（和ViewPager一样），但也许是列表的滚动速率更快，内容迭代更频繁的原因，这里设置两级冗余的缓存，以满足**少创建（二级缓存）和修改（一级缓存）View**的需求。
  >
  > 其实在上述两级缓存中间还有可以自定义的 `RecyclerView.ViewCacheExtension`拓展缓存，其适用于和当前位置无关的（一级缓存和在屏幕的位置有关），不受限View复用的情况。
  >
  > 总结一下，RecyclerView的缓存是这样的：在一个已经绘制完成的界面，所有子项都在mAttachedScrap集合里，伴随着滚动，最近离开RecyclerView的子项会进入mCacheViews里；新建的子项则可能来只mRecyclerPool；如果此时回滚，就可以直接取mCacheViews内的值；如果继续往下滚，则其内的值在到达上限后会进入mRecyclerPool，然后继续接纳新的从mAttachedScrap里来的子项。
  >
  > 如果是按照这个逻辑的话，mRecyclerPool应该基本不太会满，因为从一级缓存退下来的项和mAttachedScrap取用的应该是平衡的。但是存在一些情况——比如可见的多项子项进行更改（setAdapter或者notifyDatasetChanged），展示子项范围有非线性的变更（update操作涉及缓存对象）等，会将一级缓存内的ViewHolder全部重置，放入二级缓存。

- 增减子项会造成ViewHolder的position偏移（AdapterHelper在step1中完成），可能造成缺少一项屏外的ViewHolder（中间增加一项的情况），或者屏内某个位置的ViewHolder找不到对应项（中间减少一项，此时会尝试从缓存中获取），这些额外的项会在layoutChunk中被加入RecyclerView，以完成动画的展示。

- 每次走onCreateViewHolder时会记录下这个类型ViewHolder构造的时间，来计算构造平均值，这个平均值的作用是用来估算当页面变化时，是否有足够的时间构造一个ViewHolder（但是在布局中需要构造ViewHolder时，deadline相当于是不存在的，仅在GapWorker的任务中会用到这个逻辑，但这是干什么的呢？）。RecycledViewPool用ScrapData来保存不同类型ViewHolder的信息，包括其缓存上限、平均构造时间和平均绑定时间。

- 需要注意的各种类：**ChildHelper**（管理子项的数量、添加/移除子项相关的操作）、**AdapterHelper**（管理子项在Adapter中的位置、各种变更指令的解析）、**Recycler**（负责子项的循环使用、缓存管理、及ViewHolder本身的构造与绑定调用）。

#### RecyclerView的动画

- RecyclerView的子项动画配置来自ItemAnimator
- `animateMove`的默认实现是通过预设置translate，再执行平移动画将其拖到应在的位置来实现的。
- 单个动画的回调实现最后都是通过添加到延迟执行列表实现的。顺序是 清除动画-移动动画-添加动画。其中移动动画的执行是多个子项并行的（考虑屏幕里插入一个子项的表现）。
- 动画的实现过程是一个模板方法调用过程，在上面所说顺序的延迟执行方法中简单地调用了View的**animate**方法取得预定的动画对象，添加Recycler动画声明周期提示方法（动画开始、结束，**这些回调调用的目的是为了提醒Recycler真正清除被移除项的数据**）的回调，然后执行。
- 连续进行添加动作不会发生什么异常，因为移动动画在添加动画之前，而连续进行删除动画就会造成页面有一段空白和重叠，这个现象一方面是由于删除动画早于移动动画，下一个删除动作的到来终止了之前动作的移动动画，同时把删除的子项偏移量置为0，这导致了被删除项会瞬间回到更新后的位置导致重叠。