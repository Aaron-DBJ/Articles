# RecyclerView 的缓存机制

## 从缓存获取 ViewHolder 流程概览

![](https://raw.githubusercontent.com/Aaron-DBJ/ImageRepo/img/20240420210357.png)

### 1.1、四级缓存

Recycler缓存ViewHolder对象有4个等级，优先级从高到底依次为：

- **AttachedScrap**： **不参与滑动时的回收复用，只保存重新布局时从RecyclerView分离的item的无效、未移除、未更新的holder**。因为RecyclerView在`onLayout`的时候，会先把`children`全部移除掉，再重新添加进入，`mAttachedScrap`临时保存这些holder复用。

- **ChangedScrap**： `mChangedScrap`和`mAttachedScrap`类似，不参与滑动时的回收复用，只是用作临时保存的变量，它只会负责保存重新布局时发生变化的item的无效、未移除的holder，那么会重走adapter绑定数据的方法。

- **CachedViews** ： 用于保存最新被移除(`remove`)的ViewHolder，已经和RecyclerView分离的视图；它的作用是滚动的回收复用时如果需要新的ViewHolder时，精准匹配(根据`position/id`判断)是不是原来被移除的那个item；如果是，则直接返回ViewHolder使用，**不需要重新绑定数据**；如果不是则不返回，再去`mRecyclerPool`中找holder实例返回，并重新绑定数据。这一级的缓存是有容量限制的，最大数量为2。

- **ViewCacheExtension**： RecyclerView给开发者预留的缓存池，开发者可以自己拓展回收池，一般不会用到，用RecyclerView系统自带的已经足够了。

- **RecyclerPool**： 是一个终极回收站，真正存放着被标识废弃(其他池都不愿意回收)的ViewHolder的缓存池，如果上述`mAttachedScrap`、`mChangedScrap`、`mCachedViews`、`mViewCacheExtension`都找不到ViewHolder的情况下，就会从`mRecyclerPool`返回一个废弃的ViewHolder实例，但是这里的ViewHolder是已经被抹除数据的，没有任何绑定的痕迹，**需要重新绑定数据**。它是根据`itemType`来存储的，是以`SparseArray`嵌套一个ArraryList的形式保存ViewHolder的。
  
  ##### 1.1.1 mAttachedScrap
  
  `mAttachedScrap`存储的是当前屏幕中的ViewHolder，mAttachedScrap的对应数据结构是
  ArrayList，在调用`LayoutManager#onLayoutChildren`方法时对views进行布局，此时会将
  RecyclerView上的Views全部暂存到该集合中。
  **该缓存中的ViewHolder的特性是，如果和RV上的position或者itemId匹配上了那么可以直接拿来使用的，无需调用onBindViewHolder方法。**
  
  ##### 1.1.2 mChangedScrap
  
  `mChangedScrap`和`mAttachedScrap`属于同一级别的缓存，**不过mChangedScrap的调用场景是notifyItemChanged和notifyItemRangeChanged，只有发生变化的ViewHolder才会放入到mChangedScrap中**。`mChangedScrap`缓存中的ViewHolder是需要调用
  `onBindViewHolder`方法重新绑定数据的。
  
  ##### 1.1.3 mCachedViews
  
  `mCachedViews`缓存滑动时即将与RecyclerView分离的ViewHolder，按子View的position或
  id缓存，默认最多存放2个。mCachedViews对应的数据结构是ArrayList，但是该缓存对集合
  的大小是有限制的。
  **该缓存中ViewHolder的特性和mAttachedScrap中的特性是一样的，只要position或者**
  **itemId对应就无需重新绑定数据**。开发者可以调用setItemViewCacheSize(size)方法来改变
  缓存的大小，该层级缓存触发的一个常见的场景是滑动RecyclerView。当然调用notify()也会
  触发该缓存。
  
  ##### 1.1.4 ViewCacheExtension
  
  `ViewCacheExtension`是需要开发者自己实现的缓存，默认是空实现。基本上页面上的所有
  数据都可以通过它进行实现。
  
  ##### 1.1.5 RecyclerViewPool
  
  ViewHolder缓存池，本质上是一个SparseArray，其中key是ViewType(int类型)，value存放的是 ArrayList< ViewHolder>，默认每个ArrayList中最多存放5个ViewHolder。
  
  ### 1.2 四级缓存对比
  
  | 缓存级别 | 涉及对象                         | 说明                                                                                                                    | 是否重新创建视图View | 是否重新绑定数据 |
  | ---- | ---------------------------- | --------------------------------------------------------------------------------------------------------------------- | ------------ | -------- |
  | 一级缓存 | mAttachedScrap mChangedScrap | 缓存屏幕中可见范围的ViewHolder                                                                                                  | false        | false    |
  | 二级缓存 | mCachedViews                 | 缓存滑动时即将与RecyclerView分离的ViewHolder，按子View的position或id缓存                                                                | false        | false    |
  | 三级缓存 | mViewCacheExtension          | 开发者自行实现的缓存                                                                                                            |              |          |
  | 四级缓存 | mRecyclerPool                | ViewHolder缓存池，本质上是一个SparseArray，其中key是ViewType(int类型)，value存放的是 ArrayList< ViewHolder>，默认每个ArrayList中最多存放5个ViewHolder | false        | true     |
  
  ## 回收流程
  
  RecyclerView回收的入口有很多， 但是不管怎么样操作，RecyclerView 的回收或者复用必然涉及到add View 和 remove View 操作， 所以我们从onLayout的流程入手分析回收和复用的机制。
  首先，在LinearLayoutManager中，我们来到itemView布局入口的方法onLayoutChildren()，如下所示
  
  ```java
    @Override
    public void onLayoutChildren(RecyclerView.Recycler recycler, RecyclerView.State state) {
        if (mPendingSavedState != null || mPendingScrollPosition != RecyclerView.NO_POSITION) {
            if (state.getItemCount() == 0) {
                removeAndRecycleAllViews(recycler);//移除所有子View
                return;
            }
        }
        ensureLayoutState();
        mLayoutState.mRecycle = false;//禁止回收
        //颠倒绘制布局
        resolveShouldLayoutReverse();
        onAnchorReady(recycler, state, mAnchorInfo, firstLayoutDirection);
        //暂时分离已经附加的view，即将所有child detach并通过Scrap回收
        detachAndScrapAttachedViews(recycler);
    }
  ```
  
  在onLayoutChildren()布局的时候，先根据实际情况是否需要removeAndRecycleAllViews()移除所有的子View，哪些ViewHolder不可用；然后通过`detachAndScrapAttachedViews()`暂时分离已经附加的ItemView，并缓存到`mAttachedScrap`或`mChangedScrap`中。
  
  ```java
  // RecyclerView.java
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
                recycler.recycleViewHolderInternal(viewHolder);
            } else {
                detachViewAt(index);
                recycler.scrapView(view);
                mRecyclerView.mViewInfoStore.onViewDetached(viewHolder);
            }
        }
  ```
  
  如果viewHolder是**无效**、**未被移除**、**未被标记**（<mark>①</mark>）的则放到`recycleViewHolderInternal()`缓存起来，同时removeViewAt()移除了viewHolder。
  
  ```java
   void recycleViewHolderInternal(ViewHolder holder) {
           ·····
        if (forceRecycle || holder.isRecyclable()) {
            if (mViewCacheMax > 0
                    && !holder.hasAnyOfTheFlags(ViewHolder.FLAG_INVALID
                    | ViewHolder.FLAG_REMOVED
                    | ViewHolder.FLAG_UPDATE
                    | ViewHolder.FLAG_ADAPTER_POSITION_UNKNOWN)) {
                int cachedViewSize = mCachedViews.size();
                if (cachedViewSize >= mViewCacheMax && cachedViewSize > 0) {//如果超出容量限制，把第一个移除
                    recycleCachedViewAt(0);
                    cachedViewSize--;
                }
                     ·····
                mCachedViews.add(targetCacheIndex, holder);//mCachedViews回收
                cached = true;
            }
            if (!cached) {
                addViewHolderToRecycledViewPool(holder, true);//放到RecycledViewPool回收
                recycled = true;
            }
        }
    }
  ```
  
  `recycleViewHolderInternal()`中会优先缓存到`mCachedViews` 列表中，如果超出了mCachedViews的最大限制（默认大小为2），通过`recycleCachedViewAt()`将CacheView缓存的第一个数据添加到终极回收池RecycledViewPool后再移除掉，最后才会add()新的ViewHolder添加到`mCachedViews`中。
  如果不满足<mark>①</mark>条件，走到else分支。把当前屏幕所有的item与屏幕分离，将他们从RecyclerView的布局中拿下来，保存到`mAttachedScrap`或`mChangedScrap`中。等待重新布局时，再将ViewHolder重新一个个放到新的位置上去。
  **注意：`mAttachedScrap`和`mChangedScrap`只是保存从RecyclerView布局中当前屏幕显示的ViewHolder，不参与回收复用，单纯是为了先从RecyclerView中拿下来等下次再重新布局上去。**
  
  ## 复用流程
  
  RecyclerView对ViewHolder的复用是从`layoutChunk`方法里的 `LayoutState#next()`方法开始的。LayoutManager在布局itemView时，需要获取一个ViewHolder对象，如下所示。
  
        View next(RecyclerView.Recycler recycler) {
            if (mScrapList != null) {
                return nextViewFromScrapList();
            }
            final View view = recycler.getViewForPosition(mCurrentPosition);
            mCurrentPosition += mItemDirection;
            return view;
        }
  
  `next`方法调用RecyclerView的`getViewForPosition`方法来获取一个View，而`getViewForPosition`方法最终会调用到RecyclerView的`tryGetViewHolderForPositionByDeadline`方法，而RecyclerView真正复用的核心就在这里。
  
  ```java
  @Nullable
  ViewHolder tryGetViewHolderForPositionByDeadline(int position, boolean dryRun, long deadlineNs) {
    ViewHolder holder = null;
    // 0) 如果它是改变的废弃的ViewHolder，在scrap的mChangedScrap找
    if (mState.isPreLayout()) {
        holder = getChangedScrapViewForPosition(position);
        fromScrapOrHiddenOrCache = holder != null;
    }
    // 1)根据position分别在scrap的mAttachedScrap、mChildHelper、mCachedViews中查找
    if (holder == null) {
        holder = getScrapOrHiddenOrCachedHolderForPosition(position, dryRun);
    }
    if (holder == null) {
        final int type = mAdapter.getItemViewType(offsetPosition);
        // 2)根据id在scrap的mAttachedScrap、mCachedViews中查找
        if (mAdapter.hasStableIds()) {
            holder = getScrapOrCachedViewForId(mAdapter.getItemId(offsetPosition), type, dryRun);
        }
        if (holder == null && mViewCacheExtension != null) {
            //3)在ViewCacheExtension中查找，一般不用到，所以没有缓存
            final View view = mViewCacheExtension
                    .getViewForPositionAndType(this, position, type);
            if (view != null) {
                holder = getChildViewHolder(view);
            }
        }
        //4)在RecycledViewPool中查找
        holder = getRecycledViewPool().getRecycledView(type);
        if (holder != null) {
            holder.resetInternal();
            if (FORCE_INVALIDATE_DISPLAY_LIST) {
                invalidateDisplayListInt(holder);
            }
        }
    }
    //5)到最后如果还没有找到复用的ViewHolder，则新建一个
    holder = mAdapter.createViewHolder(RecyclerView.this, type);
  }
  ```
  
  可以看到，`tryGetViewHolderForPositionByDeadline()`方法分别去scrap、CacheView、ViewCacheExtension、RecycledViewPool中获取ViewHolder，如果没有则创建一个新的ViewHolder。
  
  #### 2.1 getChangedScrapViewForPosition
  
  一般情况下，当我们调用adapter的`notifyItemChanged()`方法，数据发生变化时，item缓存在`mChangedScrap`中，后续拿到的ViewHolder需要重新绑定数据。此时查找ViewHolder就会通过position和id分别在scrap的mChangedScrap中查找。
  
  ```java
  ViewHolder getChangedScrapViewForPosition(int position) {
      //通过position
      for (int i = 0; i < changedScrapSize; i++) {
          final ViewHolder holder = mChangedScrap.get(i);
          return holder;
      }
      // 通过id
      if (mAdapter.hasStableIds()) {
          final long id = mAdapter.getItemId(offsetPosition);
          for (int i = 0; i < changedScrapSize; i++) {
              final ViewHolder holder = mChangedScrap.get(i);
              return holder;
          }
      }
      return null;
  }
  ```
  
  #### 2.2 getScrapOrHiddenOrCachedHolderForPosition
  
  如果没有找到视图，根据position分别在scrap的`mAttachedScrap`、`mChildHelper`、`mCachedViews`中查找，涉及的方法如下。
  
  ```java
    ViewHolder getScrapOrHiddenOrCachedHolderForPosition(int position, boolean dryRun) {
        final int scrapCount = mAttachedScrap.size();
        // 首先从mAttachedScrap中查找，精准匹配有效的ViewHolder
        for (int i = 0; i < scrapCount; i++) {
            final ViewHolder holder = mAttachedScrap.get(i);
            return holder;
        }
        //接着在mChildHelper中mHiddenViews查找隐藏的ViewHolder
        if (!dryRun) {
            View view = mChildHelper.findHiddenNonRemovedView(position);
            if (view != null) {
                final ViewHolder vh = getChildViewHolderInt(view);
                scrapView(view);
                return vh;
            }
        }
        //最后从我们的一级缓存中mCachedViews查找。
        final int cacheSize = mCachedViews.size();
        for (int i = 0; i < cacheSize; i++) {
            final ViewHolder holder = mCachedViews.get(i);
            return holder;
        }
    }
  ```
  
  可以看到，`getScrapOrHiddenOrCachedHolderForPosition`查找ViewHolder的顺序如下：

- 首先，从`mAttachedScrap`中查找，精准匹配有效的ViewHolder；

- 接着，在mChildHelper中mHiddenViews查找隐藏的ViewHolder；

- 最后，从一级缓存中`mCachedViews`查找。
  
  #### 2.3 getScrapOrCachedViewForId
  
  如果在`getScrapOrHiddenOrCachedHolderForPosition`没有找到视图，则通过id在scrap的`mAttachedScrap`、`mCachedViews`中查找，代码如下。
  
  ```java
  ViewHolder getScrapOrCachedViewForId(long id, int type, boolean dryRun) {
     //在Scrap的mAttachedScrap中查找
     final int count = mAttachedScrap.size();
     for (int i = count - 1; i >= 0; i--) {
         final ViewHolder holder = mAttachedScrap.get(i);
          return holder;
      }
      //在一级缓存mCachedViews中查找
      final int cacheSize = mCachedViews.size();
      for (int i = cacheSize - 1; i >= 0; i--) {
          final ViewHolder holder = mCachedViews.get(i);
          return holder;
      }    
  }
  ```
  
  getScrapOrCachedViewForId()方法查找的顺序如下：

- 首先， 从mAttachedScrap中查找，精准匹配有效的ViewHolder；

- 接着， 从一级缓存中mCachedViews查找；
  
  #### 2.4 mViewCacheExtension
  
  `mViewCacheExtension`是由开发者定义的一层缓存策略，Recycler并没有将任何view缓存到这里。这里没有自定义缓存策略，那么就找不到对应的view。
  
  ```java
  if (holder == null && mViewCacheExtension != null) {
    final View view = mViewCacheExtension.getViewForPositionAndType(this, position, type);
    if (view != null) {
          holder = getChildViewHolder(view);
      }
  }
  ```
  
  #### 2.5 RecycledViewPool
  
  在ViewHolder的四级缓存中，我们有提到过RecycledViewPool，它是通过itemType把ViewHolder的List缓存到SparseArray中的，在`getRecycledViewPool().getRecycledView(type)`根据itemType从SparseArray获取ScrapData ，然后再从里面获取ArrayList<ViewHolder>，从而获取到ViewHolder。
  
  ```java
  @Nullable
  public ViewHolder getRecycledView(int viewType) {
    final ScrapData scrapData = mScrap.get(viewType);//根据viewType获取对应的ScrapData 
    if (scrapData != null && !scrapData.mScrapHeap.isEmpty()) {
        final ArrayList<ViewHolder> scrapHeap = scrapData.mScrapHeap;
        for (int i = scrapHeap.size() - 1; i >= 0; i--) {
            if (!scrapHeap.get(i).isAttachedToTransitionOverlay()) {
                return scrapHeap.remove(i);
            }
        }
    }
    return null;
  }
  ```
  
  #### 2.6 创建新的ViewHolder
  
  如果还没有获取到ViewHolder，则通过`mAdapter.createViewHolder()`创建一个新的ViewHolder返回。
  
  ```java
  // 如果还没有找到复用的ViewHolder，则新建一个
  holder = mAdapter.createViewHolder(RecyclerView.this, type);
  ```
  
  完整的流程图：
  ![](https://raw.githubusercontent.com/Aaron-DBJ/ImageRepo/img/20240420220227.png)
  [RecyclerView 回收复用](https://segmentfault.com/a/1190000040421118#item-2)
  [真正带你搞懂 RecyclerView 的缓存机制](https://zhuanlan.zhihu.com/p/80475040)
  [深入理解 RecyclerView 的回收复用缓存机制详解（匠心巨作-下） - 掘金](https://juejin.cn/post/6984974879296585764)
  
  ## 预加载机制
  
  [掌握这17张图，掌握RecyclerView中的“黑科技”预加载_提前下载recyclerview中的图片资源-CSDN博客](https://blog.csdn.net/chuyouyinghe/article/details/128802861)
