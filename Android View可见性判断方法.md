在Android开发过程中，经常需要获取Window或某个View的可见性变化时机，以便在View的Visibility变化时进行相应的处理。目前，比较常用的判断View可见性时机的回调有

- *onWindowVisibilityChanged*
- *onVisibilityChanged*
- *OnAttachStateChangeListener*#*onViewAttachedToWindow*

# 一、onWindowVisibilityChanged

```java
 		/**
     * Called when the window containing has change its visibility
     * (between {@link #GONE}, {@link #INVISIBLE}, and {@link #VISIBLE}).  Note
     * that this tells you whether or not your window is being made visible
     * to the window manager; this does <em>not</em> tell you whether or not
     * your window is obscured by other windows on the screen, even if it
     * is itself visible.
     *
     * @param visibility The new visibility of the window.
     */
		protected void onWindowVisibilityChanged(@Visibility int visibility) {
        if (visibility == VISIBLE) {
            initialAwakenScrollBars();
        }
    }
```

由方法注释可知，它是在窗口可见性改变时调用，而且注意这只是在*Window*对*WindowManager*可见时调用，并不是告知你当前可见的*Window*是否被遮挡。

<span id="1">查看代码</span>，发现其调用位置有**3**处，添加时在`performTraversals`方法中(代码有省略)，*Activity* *onStop*生命周期会*remove*掉*DecorView*和对应的*Window*，在`removeView`方法中会调用`dispatchDetachedFromWindow`方法，该方法内又会调用`onWindowVisibilityChanged`

```java
private void performTraversals() {
        // cache mView since it is used so much below...
        final View host = mView;
				......

        final int viewVisibility = getHostVisibility();
        final boolean viewVisibilityChanged = !mFirst
                && (mViewVisibility != viewVisibility || mNewSurfaceNeeded
                // Also check for possible double visibility update, which will make current
                // viewVisibility value equal to mViewVisibility and we may miss it.
                || mAppVisibilityChanged);
        mAppVisibilityChanged = false;
        final boolean viewUserVisibilityChanged = !mFirst &&
                ((mViewVisibility == View.VISIBLE) != (viewVisibility == View.VISIBLE));

        ......
        if (mFirst) {
            ......

            // We used to use the following condition to choose 32 bits drawing caches:
            // PixelFormat.hasAlpha(lp.format) || lp.format == PixelFormat.RGBX_8888
            // However, windows are now always 32 bits by default, so choose 32 bits
            mAttachInfo.mUse32BitDrawingCache = true;
            mAttachInfo.mWindowVisibility = viewVisibility;
            mAttachInfo.mRecomputeGlobalAttributes = false;
            mLastConfigurationFromResources.setTo(config);
            mLastSystemUiVisibility = mAttachInfo.mSystemUiVisibility;
            // Set the layout direction if it has not been set before (inherit is the default)
            if (mViewLayoutDirectionInitial == View.LAYOUT_DIRECTION_INHERIT) {
                host.setLayoutDirection(config.getLayoutDirection());
            }
           /**
            *	①
            */
            host.dispatchAttachedToWindow(mAttachInfo, 0);
            mAttachInfo.mTreeObserver.dispatchOnWindowAttachedChange(true);
            dispatchApplyInsets(host);
        } else {
           	......
            }
        }

        if (viewVisibilityChanged) {
            mAttachInfo.mWindowVisibility = viewVisibility;
           /**
            *	②
            */
            host.dispatchWindowVisibilityChanged(viewVisibility);
            if (viewUserVisibilityChanged) {
                host.dispatchVisibilityAggregated(viewVisibility == View.VISIBLE);
            }
            if (viewVisibility != View.VISIBLE || mNewSurfaceNeeded) {
                endDragResizing();
                destroyHardwareResources();
            }
            if (viewVisibility == View.GONE) {
                // After making a window gone, we will count it as being
                // shown for the first time the next time it gets focus.
                mHasHadWindowFocus = false;
            }
        }
      }
```

```java
void dispatchDetachedFromWindow() {
        AttachInfo info = mAttachInfo;
        if (info != null) {
            int vis = info.mWindowVisibility;
            if (vis != GONE) {
                onWindowVisibilityChanged(GONE);// ③
                if (isShown()) {
                    // Invoking onVisibilityAggregated directly here since the subtree
                    // will also receive detached from window
                    onVisibilityAggregated(false);
                }
            }
        }

        onDetachedFromWindow();
        onDetachedFromWindowInternal();

        ......
        ListenerInfo li = mListenerInfo;
        final CopyOnWriteArrayList<OnAttachStateChangeListener> listeners =
                li != null ? li.mOnAttachStateChangeListeners : null;
        if (listeners != null && listeners.size() > 0) {
            // NOTE: because of the use of CopyOnWriteArrayList, we *must* use an iterator to
            // perform the dispatching. The iterator is a safe guard against listeners that
            // could mutate the list by calling the various add/remove methods. This prevents
            // the array from being modified while we iterate it.
            for (OnAttachStateChangeListener listener : listeners) {
                listener.onViewDetachedFromWindow(this);
            }
        }

        ......
    }
```

调用位置已注释，首先看第一处①，应用启动时，*host*就是*DecorView*对象，它是一个*ViewGroup*，然后在*ViewGroup*类中查看。

```java
		/**
 		 * ViewGroup.class
 		 */
		@Override
    @UnsupportedAppUsage
    void dispatchAttachedToWindow(AttachInfo info, int visibility) {
        mGroupFlags |= FLAG_PREVENT_DISPATCH_ATTACHED_TO_WINDOW;
      	// [1]
        super.dispatchAttachedToWindow(info, visibility);
        mGroupFlags &= ~FLAG_PREVENT_DISPATCH_ATTACHED_TO_WINDOW;

        final int count = mChildrenCount;
        final View[] children = mChildren;
        for (int i = 0; i < count; i++) {
            final View child = children[i];
          	// [2]
            child.dispatchAttachedToWindow(info,
                    combineVisibility(visibility, child.getVisibility()));
        }
        final int transientCount = mTransientIndices == null ? 0 : mTransientIndices.size();
        for (int i = 0; i < transientCount; ++i) {
            View view = mTransientViews.get(i);
            view.dispatchAttachedToWindow(info,
                    combineVisibility(visibility, view.getVisibility()));
        }
    }
```

可以看到，`ViewGroup#dispatchAttachedToWindow`方法主要作用就是

[1] 调用自身的*dispatchAttachedToWindow*方法，处理自己*attach*到*Window*

[2] 向子*View*分发事件，让每个子*View*处理*attach*到*Window*的事件

由此可知，最终都会调用到`View#dispatchAttachedToWindow`方法：

```java
void dispatchAttachedToWindow(AttachInfo info, int visibility) {
        mAttachInfo = info;
        ......
        // Transfer all pending runnables.
        if (mRunQueue != null) {
            mRunQueue.executeActions(info.mHandler);
            mRunQueue = null;
        }
        performCollectViewAttributes(mAttachInfo, visibility);
        onAttachedToWindow();// [1]

        ListenerInfo li = mListenerInfo;
        final CopyOnWriteArrayList<OnAttachStateChangeListener> listeners =
                li != null ? li.mOnAttachStateChangeListeners : null;
        if (listeners != null && listeners.size() > 0) {
            // NOTE: because of the use of CopyOnWriteArrayList, we *must* use an iterator to
            // perform the dispatching. The iterator is a safe guard against listeners that
            // could mutate the list by calling the various add/remove methods. This prevents
            // the array from being modified while we iterate it.
            for (OnAttachStateChangeListener listener : listeners) {
                listener.onViewAttachedToWindow(this);// [2]
            }
        }

        int vis = info.mWindowVisibility;
        if (vis != GONE) {
            onWindowVisibilityChanged(vis);// [3]
            if (isShown()) {
                // Calling onVisibilityAggregated directly here since the subtree will also
                // receive dispatchAttachedToWindow and this same call
                onVisibilityAggregated(vis == VISIBLE);
            }
        }

        // Send onVisibilityChanged directly instead of dispatchVisibilityChanged.
        // As all views in the subtree will already receive dispatchAttachedToWindow
        // traversing the subtree again here is not desired.
        onVisibilityChanged(this, visibility);

       	......
    }
```

`View#dispatchAttachedToWindow`方法集中处理了**`onAttachedToWindow`、`onViewAttachedToWindow`、`onWindowVisibilityChanged`和`onVisibilityChanged`**这4种可见性变化相关的回调函数。

对于`onWindowVisibilityChanged`方法来说，首先会通过AttachInfo对象获取现在窗口（mWindowVisibility）可见性。`mWindowVisibility`变量的赋值也在[performTraversals](#1)方法中。

```java
......
final int viewVisibility = getHostVisibility();
// mViewVisibility在创建ViewRootImpl对象时，初始化值是GONE；在完成测量布局后赋值为viewVisibility
final boolean viewVisibilityChanged = !mFirst
                && (mViewVisibility != viewVisibility || mNewSurfaceNeeded
                // Also check for possible double visibility update, which will make current
                // viewVisibility value equal to mViewVisibility and we may miss it.
                || mAppVisibilityChanged);
......
mAttachInfo.mWindowVisibility = viewVisibility;
......

```

在*Window*被添加到屏幕上后(*mWindowSession.addToDisplay*)，`getHostVisibility()`就返回`Visible`。所以只要`mWindowVisibility`不为`GONE`就会调用`onWindowVisibilityChanged`方法。这就是它的第一种调用场景。

第二处调用位置在[②](#1)处，主要代码是

```java
	if (viewVisibilityChanged) {
            mAttachInfo.mWindowVisibility = viewVisibility;
           /**
            *	②
            */
            host.dispatchWindowVisibilityChanged(viewVisibility);
    	......
	}
```

在*DecorView*加载时，如果`mFirst==false`（非首次加载），很可能进行该条件体进行调用。

第三种情况③，例如打开一个新页面，老页面走到*onStop*声明周期方法，如③处，只要不是*GONE*就会调用。

综上，`onWindowVisibilityChanged`的调用：

- 每当一个页面打开或移除时（具体点说就是ViewRootImpl将DecovView添加到屏幕上展示），如果关联的*Window*可见（不等于*GONE*），则会调用
- 打开时，是在`View#dispatchAttachedToWindow`中进行调用，分离时在`View#dispatchDetachFromWindow`时调用，并传入默认参数*GONE*

# 二、onVisibilityChanged

`onVisibilityChanged`调用时机和`onWindowVisibilityChanged`非常类似，对于APP启动打开页面时，会处理重写该方法的View attach到Window的事件，此时默认传入的参数是*Visible*(值为0)。

```java
	// ViewRootImpl.class
	private void performTraversals() {
        ......
  	 	host.dispatchAttachedToWindow(mAttachInfo, 0);
  	    ......
	}

	// View.class
	void dispatchAttachedToWindow(AttachInfo info, int visibility) {
		......
        // Send onVisibilityChanged directly instead of dispatchVisibilityChanged.
        // As all views in the subtree will already receive dispatchAttachedToWindow
        // traversing the subtree again here is not desired.
        onVisibilityChanged(this, visibility);
       	......
    }
```

然后，每次调用`setVisibility`方法来控制视图的可见性时都会回调该方法。

```java
	// View.class
	public void setVisibility(@Visibility int visibility) {
        setFlags(visibility, VISIBILITY_MASK);
    }

	void setFlags(int flags, int mask) {
        ......

        if ((changed & VISIBILITY_MASK) != 0) {
            ......
            if (mAttachInfo != null) {
                dispatchVisibilityChanged(this, newVisibility);
				......
            }
        }        
        .......
    }

	protected void dispatchVisibilityChanged(@NonNull View changedView,
            @Visibility int visibility) {
        onVisibilityChanged(changedView, visibility);
    }
```

同样，在页面关闭时（或打开新页面覆盖旧页面），会执行*onStop*生命周期方法，其实是调用`ActivityThread#handleStopActivity`方法，然后会调用到`updateVisibility`方法

```java
// ActivityThread.class
private void updateVisibility(ActivityClientRecord r, boolean show) {
        View v = r.activity.mDecor;
        if (v != null) {
            if (show) {
                if (!r.activity.mVisibleFromServer) {
                    r.activity.mVisibleFromServer = true;
                    mNumVisibleActivities++;
                    if (r.activity.mVisibleFromClient) {
                        r.activity.makeVisible();
                    }
                }
                ......
            } else {
                if (r.activity.mVisibleFromServer) {
                    r.activity.mVisibleFromServer = false;
                    mNumVisibleActivities--;
                    v.setVisibility(View.INVISIBLE);
                }
            }
        }
    }
```

`mVisibleFromServer`默认是*false*，在*onResume*后赋值为*true*，此时`mVisibleFromServer`为*true*，进入条件体，首先将`mVisibleFromServer`设置为*false*，然后通过*DecorView*调用`setVisibility`方法来控制视图显示，并默认传输`View.INVISIBLE`。我们知道调用`setVisibility`就可能触发`onVisibilityChanged`的执行。

**总结**：

- 页面加载时，会在*View* *attach*到*Window*时（*dispatchAttachedToWindow*方法）调用`onVisibilityChanged`
- 通过`setVisibility`来改变View的可见性时会调用`onVisibilityChanged`
- 关闭页面时，在`handleStopActivity`方法中会调用`updateVisibility`方法，内部也会调用`setVisibility`

，并传入默认参数*INVISIBLE*。

# 三、*OnAttachStateChangeListener*#onViewAttachedToWindow

*OnAttachStateChangeListener*定义了2个接口方法，分别在*View Attach/Detach to Window*时候调用。要使用该接口首先需要注册这个监听器。

```java
public void addOnAttachStateChangeListener(OnAttachStateChangeListener listener) {
        ListenerInfo li = getListenerInfo();
        if (li.mOnAttachStateChangeListeners == null) {
            li.mOnAttachStateChangeListeners
                    = new CopyOnWriteArrayList<OnAttachStateChangeListener>();
        }
        li.mOnAttachStateChangeListeners.add(listener);
    }
```

调用位置很单纯，就在*View#dispatchAttachedToWindow*方法里，如果有注册过该监听器，就会调用

```java
void dispatchAttachedToWindow(AttachInfo info, int visibility) {
        ......
        onAttachedToWindow();

        ListenerInfo li = mListenerInfo;
        final CopyOnWriteArrayList<OnAttachStateChangeListener> listeners =
                li != null ? li.mOnAttachStateChangeListeners : null;
        if (listeners != null && listeners.size() > 0) {
            // NOTE: because of the use of CopyOnWriteArrayList, we *must* use an iterator to
            // perform the dispatching. The iterator is a safe guard against listeners that
            // could mutate the list by calling the various add/remove methods. This prevents
            // the array from being modified while we iterate it.
            for (OnAttachStateChangeListener listener : listeners) {
                listener.onViewAttachedToWindow(this);
            }
        }
		......
    }
```

