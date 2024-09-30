# DiffUtil使用

DiffUtil是Recyclerview中提供的一个计算差量更新数据的工具，用于局部刷新列表的场景（例如仅有部分数据发生变化），避免无脑`notifyDataSetChanged`全量刷新列表，提高性能。

## 步骤

### 1、实现`DiffUtil.Callback`

源码如下

```java
    /**
     * A Callback class used by DiffUtil while calculating the diff between two lists.
     */
    public abstract static class Callback {
        /**
         * Returns the size of the old list.
         *
         * @return The size of the old list.
         */
        public abstract int getOldListSize();

        /**
         * Returns the size of the new list.
         *
         * @return The size of the new list.
         */
        public abstract int getNewListSize();

        /**
         * Called by the DiffUtil to decide whether two object represent the same Item.
         * <p>
         * For example, if your items have unique ids, this method should check their id equality.
         *
         * @param oldItemPosition The position of the item in the old list
         * @param newItemPosition The position of the item in the new list
         * @return True if the two items represent the same object or false if they are different.
         */
        public abstract boolean areItemsTheSame(int oldItemPosition, int newItemPosition);

        /**
         * Called by the DiffUtil when it wants to check whether two items have the same data.
         * DiffUtil uses this information to detect if the contents of an item has changed.
         * <p>
         * DiffUtil uses this method to check equality instead of {@link Object#equals(Object)}
         * so that you can change its behavior depending on your UI.
         * For example, if you are using DiffUtil with a
         * {@link RecyclerView.Adapter RecyclerView.Adapter}, you should
         * return whether the items' visual representations are the same.
         * <p>
         * This method is called only if {@link #areItemsTheSame(int, int)} returns
         * {@code true} for these items.
         *
         * @param oldItemPosition The position of the item in the old list
         * @param newItemPosition The position of the item in the new list which replaces the
         *                        oldItem
         * @return True if the contents of the items are the same or false if they are different.
         */
        public abstract boolean areContentsTheSame(int oldItemPosition, int newItemPosition);

        /**
         * When {@link #areItemsTheSame(int, int)} returns {@code true} for two items and
         * {@link #areContentsTheSame(int, int)} returns false for them, DiffUtil
         * calls this method to get a payload about the change.
         * <p>
         * For example, if you are using DiffUtil with {@link RecyclerView}, you can return the
         * particular field that changed in the item and your
         * {@link RecyclerView.ItemAnimator ItemAnimator} can use that
         * information to run the correct animation.
         * <p>
         * Default implementation returns {@code null}.
         *
         * @param oldItemPosition The position of the item in the old list
         * @param newItemPosition The position of the item in the new list
         * @return A payload object that represents the change between the two items.
         */
        @Nullable
        public Object getChangePayload(int oldItemPosition, int newItemPosition) {
            return null;
        }
    }
```

该接口有4个方法：获取旧数据集大小，获取新数据集大小，判断2个item是不是相同，判断2个item的内容是不是相同。

这个接口的作用就是让用户自己实现判断2个item是不是相同，相同就不需要更新。

### 2、实现`ListUpdateCallback`

源码

```java
public interface ListUpdateCallback {
    /**
     * Called when {@code count} number of items are inserted at the given position.
     *
     * @param position The position of the new item.
     * @param count    The number of items that have been added.
     */
    void onInserted(int position, int count);

    /**
     * Called when {@code count} number of items are removed from the given position.
     *
     * @param position The position of the item which has been removed.
     * @param count    The number of items which have been removed.
     */
    void onRemoved(int position, int count);

    /**
     * Called when an item changes its position in the list.
     *
     * @param fromPosition The previous position of the item before the move.
     * @param toPosition   The new position of the item.
     */
    void onMoved(int fromPosition, int toPosition);

    /**
     * Called when {@code count} number of items are updated at the given position.
     *
     * @param position The position of the item which has been updated.
     * @param count    The number of items which has changed.
     */
    void onChanged(int position, int count, @Nullable Object payload);
}
```

有4个方法，分别在插入、异常、移动和更新数据时调用，需要调用对应的更新列表方法：`notifyItemRangeInserted`、`notifyItemRangeRemoved`、`notifyItemMoved`和`notifyItemRangeChanged`来触发列表重新渲染。

Recyclerview中有一个默认实现的类: `AdapterListUpdateCallback`，源码如下

```java
public final class AdapterListUpdateCallback implements ListUpdateCallback {
    @NonNull
    private final RecyclerView.Adapter mAdapter;

    /**
     * Creates an AdapterListUpdateCallback that will dispatch update events to the given adapter.
     *
     * @param adapter The Adapter to send updates to.
     */
    public AdapterListUpdateCallback(@NonNull RecyclerView.Adapter adapter) {
        mAdapter = adapter;
    }

    /** {@inheritDoc} */
    @Override
    public void onInserted(int position, int count) {
        mAdapter.notifyItemRangeInserted(position, count);
    }

    /** {@inheritDoc} */
    @Override
    public void onRemoved(int position, int count) {
        mAdapter.notifyItemRangeRemoved(position, count);
    }

    /** {@inheritDoc} */
    @Override
    public void onMoved(int fromPosition, int toPosition) {
        mAdapter.notifyItemMoved(fromPosition, toPosition);
    }

    /** {@inheritDoc} */
    @Override
    public void onChanged(int position, int count, Object payload) {
        mAdapter.notifyItemRangeChanged(position, count, payload);
    }
}
```

### 3、调用DiffUtil.calculateDiff计算出DiffUtil.DiffResult

```java
    // ContactDiffCallback是DiffUtil.Callback的实现类
    ContactDiffCallback diffCallback = new ContactDiffCallback(oldData, newData);
    DiffUtil.DiffResult diffResult = DiffUtil.calculateDiff(diffCallback);
```

### 4、调用DiffUtil.DiffResult的`dispatchUpdatesTo`更新列表

```java
diffResult.dispatchUpdatesTo(new AdapterListUpdateCallback(mAdapter));
```

# 异步计算

根据DiffUtil的使用过程知道，需要先计算出待更新的数据，然后局部更新列表。这个过程是在主进程进行的，数据量较少时还可，当时当数据量很大时，这个过程无疑是比较耗时的，容易导致ANR。

索性，Recyclerview也提供了异步计算工具`AsyncListDiffer`，在子线程中处理数据计算过程，然后切到主线程更新UI。

## AsyncListDiffer使用步骤

1、实现`DiffUtil.ItemCallback`

源码

```java
public abstract static class ItemCallback<T> {
        /**
         * Called to check whether two objects represent the same item.
         * <p>
         * For example, if your items have unique ids, this method should check their id equality.
         * <p>
         * Note: {@code null} items in the list are assumed to be the same as another {@code null}
         * item and are assumed to not be the same as a non-{@code null} item. This callback will
         * not be invoked for either of those cases.
         *
         * @param oldItem The item in the old list.
         * @param newItem The item in the new list.
         * @return True if the two items represent the same object or false if they are different.
         * @see Callback#areItemsTheSame(int, int)
         */
        public abstract boolean areItemsTheSame(@NonNull T oldItem, @NonNull T newItem);

        /**
         * Called to check whether two items have the same data.
         * <p>
         * This information is used to detect if the contents of an item have changed.
         * <p>
         * This method to check equality instead of {@link Object#equals(Object)} so that you can
         * change its behavior depending on your UI.
         * <p>
         * For example, if you are using DiffUtil with a
         * {@link RecyclerView.Adapter RecyclerView.Adapter}, you should
         * return whether the items' visual representations are the same.
         * <p>
         * This method is called only if {@link #areItemsTheSame(T, T)} returns {@code true} for
         * these items.
         * <p>
         * Note: Two {@code null} items are assumed to represent the same contents. This callback
         * will not be invoked for this case.
         *
         * @param oldItem The item in the old list.
         * @param newItem The item in the new list.
         * @return True if the contents of the items are the same or false if they are different.
         * @see Callback#areContentsTheSame(int, int)
         */
        public abstract boolean areContentsTheSame(@NonNull T oldItem, @NonNull T newItem);

        /**
         * When {@link #areItemsTheSame(T, T)} returns {@code true} for two items and
         * {@link #areContentsTheSame(T, T)} returns false for them, this method is called to
         * get a payload about the change.
         * <p>
         * For example, if you are using DiffUtil with {@link RecyclerView}, you can return the
         * particular field that changed in the item and your
         * {@link RecyclerView.ItemAnimator ItemAnimator} can use that
         * information to run the correct animation.
         * <p>
         * Default implementation returns {@code null}.
         *
         * @see Callback#getChangePayload(int, int)
         */
        @SuppressWarnings({"unused"})
        @Nullable
        public Object getChangePayload(@NonNull T oldItem, @NonNull T newItem) {
            return null;
        }
    }
```

和`DiffUtil.Callback`类似，不过只需实现2个方法，用于判断item是否相同和item的内容是否相同。

### 2、创建`AsyncListDiffer`对象

```java
// ListItemBean是数据bean，AsyncListDiffer有2个参数：adapter和ItemCallback
AsyncListDiffer<ListItemBean> asyncListDiffer = new AsyncListDiffer<>(mAdapter, new ContactItemCallback());
```

### 3、提交数据

调用 `asyncListDiffer.submitList`方法提交更新后的数据，其内部计算完需要更新的数据后，通过`AdapterListUpdateCallback`去更新列表。

```java
 asyncListDiffer.submitList(newData);
```

**注意调用`submitList`传入的新数据list要和旧数据list是不同的实例对象，不然会被提前return，导致更新无效**

# 注意事项

- **尽量使用不可变数据对象。**
  
  DiffUtil 是通过比较两个数据对象的引用来判断它们是否相同的，因此如果我们在列表中使用可变数据对象，那么很容易出现引用相同但内容不同的情况，从而导致列表的更新出现问题。所以，在使用 DiffUtil 时，最好使用不可变数据对象，或者在可变数据对象中保证引用的唯一性。
- **注意数据对象的 equals() 方法的实现。**
  
  如果我们使用的是自定义的数据对象，那么需要确保它的 equals() 方法的实现是正确的，否则会导致 DiffUtil 计算的不准确。具体来说，equals() 方法应该正确地比较数据对象的所有字段。
- **尽量避免在列表中使用动态数据。**
  
  DiffUtil 的差异计算是基于静态数据的，如果我们在列表中使用了动态数据，比如时间戳、随机数等，那么可能会导致每次计算的结果不同，从而影响列表的更新效果。所以，如果需要在列表中使用动态数据，可以将其计算结果缓存下来，以避免影响列表的更新。
- **尽量避免对数据进行频繁的更新**。
  
  虽然 DiffUtil 可以非常高效地计算出数据集的差异，但是如果我们对数据进行频繁的更新，那么就会导致 DiffUtil 不断地进行计算，从而影响应用的性能。所以，在使用 DiffUtil 时，尽量避免对数据进行频繁的更新，可以将数据集的更新批量处理，或者使用合适的更新策略，如增量更新、局部更新等。
- **注意数据集的顺序。**
  
  DiffUtil 的差异计算是基于数据集的顺序的，如果数据集的顺序发生了变化，那么就需要重新计算差异。所以，在使用 DiffUtil 时，需要注意数据集的顺序，尽量避免频繁地对数据集进行排序等操作。
- **避免在 UI 线程中进行差异计算。**
  
  DiffUtil 的差异计算可能会比较耗时，如果在 UI 线程中进行计算，就会导致应用的卡顿，影响用户体验。所以，在使用 DiffUtil 时，最好将差异计算放在一个异步任务中进行，或者使用其他方式来避免阻塞 UI 线程。




