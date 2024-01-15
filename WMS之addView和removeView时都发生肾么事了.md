[toc]

# 一、前言

众所周知，*Android*中是通过*WindowManger*来添加、删除和更新视图，也就是*WindowManager*提供的`addView`、`removeView`和`updateViewLayout`方法。但是在添加或移除*View*的代码逻辑是啥，在这期间做了哪些工作，调用`removeView`方法就真的立刻删除*view*了吗？本文正是针对这些问题，对相关系统源码做了一次梳理和分析。作为一次学习源码过程，希望对相关知识点做到既能用，也明白底层原理。

在详细梳理代码前，首先我们先明确一下*View*的这些操作方法在哪里声明的，实现是怎样的，调用链路是啥。

1、「增删改」的*View*操作方法都是在**ViewManager**接口中声明的

```
public interface ViewManager
{
    /**
     * Assign the passed LayoutParams to the passed View and add the view to the window.
     * <p>Throws {@link android.view.WindowManager.BadTokenException} for certain programming
     * errors, such as adding a second view to a window without removing the first view.
     * <p>Throws {@link android.view.WindowManager.InvalidDisplayException} if the window is on a
     * secondary {@link Display} and the specified display can't be found
     * (see {@link android.app.Presentation}).
     * @param view The view to be added to this window.
     * @param params The LayoutParams to assign to view.
     */
    public void addView(View view, ViewGroup.LayoutParams params);
    public void updateViewLayout(View view, ViewGroup.LayoutParams params);
    public void removeView(View view);
}
```

2、**WindowManagerImpl**是**WindowManager**接口的实现类，相关代码如下

```
		// 添加view
		@Override
    public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
        applyDefaultToken(params);
        mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
    }
		
		//更新布局
    @Override
    public void updateViewLayout(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
        applyDefaultToken(params);
        mGlobal.updateViewLayout(view, params);
    }

		//移除view
		@Override
    public void removeView(View view) {
        mGlobal.removeView(view, false);
    }

		//立即移除view
 		@Override
    public void removeViewImmediate(View view) {
        mGlobal.removeView(view, true);
    }
```

发现对于*WindowManagerImpl*来说，并不是它自己来处理，而是委托给**mGlobal**对象处理。*mGlobal*是啥呢，就是*WindowManagerGlobal*的对象。

3、**WindowManagerGlobal**才是真正处理*View*「增删改」的类

综上，`addView`和`removeView`的处理链路如下

![image-20211228180515868](/Users/mtdp/Library/Application Support/typora-user-images/image-20211228180515868.png)

# 二、源码分析

第一节总整体视角说明了添加移除View的参与者，以及调用链路。本小节，从代码角度出发，分析一下在`addView`和`removeView`时具体逻辑和所做工作。

## 2.1 addView

前文已知，*addView*的代码实现在*WindowManagerGlobal*中，在该类中，有些重要的缓存数据的结构，详见注释说明

```
		// 	存放要添加的view对象	
		private final ArrayList<View> mViews = new ArrayList<View>();
		// 存放窗口对应的ViewRootImpl对象
    private final ArrayList<ViewRootImpl> mRoots = new ArrayList<ViewRootImpl>();
   	// 存放将要移除的View对象，这个列表中的view属于未被移除但将要移除的状态
    private final ArraySet<View> mDyingViews = new ArraySet<View>();
```

### 1、WindowManagerGlobal#addView

代码如下（部分无关代码已省略）

```
public void addView(View view, ViewGroup.LayoutParams params,
            Display display, Window parentWindow) {
        ......
        final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams) params;
        if (parentWindow != null) {
            parentWindow.adjustLayoutParamsForSubWindow(wparams);
        } else {
            // If there's no parent, then hardware acceleration for this view is
            // set from the application's hardware acceleration setting.
            final Context context = view.getContext();
            if (context != null
                    && (context.getApplicationInfo().flags
                            & ApplicationInfo.FLAG_HARDWARE_ACCELERATED) != 0) {
                wparams.flags |= WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED;
            }
        }

        ViewRootImpl root;
        View panelParentView = null;

        synchronized (mLock) {
            // Start watching for system property changes.
            if (mSystemPropertyUpdater == null) {
                mSystemPropertyUpdater = new Runnable() {
                    @Override public void run() {
                        synchronized (mLock) {
                            for (int i = mRoots.size() - 1; i >= 0; --i) {
                                mRoots.get(i).loadSystemProperties();
                            }
                        }
                    }
                };
                SystemProperties.addChangeCallback(mSystemPropertyUpdater);
            }
			// ①
            int index = findViewLocked(view, false);
            if (index >= 0) {
                if (mDyingViews.contains(view)) {
                    // Don't wait for MSG_DIE to make it's way through root's queue.
                    mRoots.get(index).doDie();
                } else {
                    throw new IllegalStateException("View " + view
                            + " has already been added to the window manager.");
                }
            }

            // If this is a panel window, then find the window it is being
            // attached to for future reference.
            if (wparams.type >= WindowManager.LayoutParams.FIRST_SUB_WINDOW &&
                    wparams.type <= WindowManager.LayoutParams.LAST_SUB_WINDOW) {
                final int count = mViews.size();
                for (int i = 0; i < count; i++) {
                    if (mRoots.get(i).mWindow.asBinder() == wparams.token) {
                        panelParentView = mViews.get(i);
                    }
                }
            }
			// ②
            root = new ViewRootImpl(view.getContext(), display);
            view.setLayoutParams(wparams);
			// ③
            mViews.add(view);
            mRoots.add(root);
            mParams.add(wparams);
            try {
              	// ④
                root.setView(view, wparams, panelParentView);
            } catch (RuntimeException e) {
                // BadTokenException or InvalidDisplayException, clean up.
                if (index >= 0) {
                    removeViewLocked(index, true);
                }
                throw e;
            }
        }
    }
```

下面👇详细说明下标注序号的代码作用。

①、首先是找到要添加的*view*的位置索引，对于新添加的*view*，由于以前不存在所以返回。同理，也就会跳过对将要移除*view*的处理。如果*view*的位置索引大于0，而且在存放将要移除*view*的列表中存在，那么会去调用*ViewRootImpl*的`doDie`方法移除*view*，该方法在*removeView*时会详细介绍。

②、代码执行到该处，会创建*ViewRootImpl*对象，在构造方法中，初始化变量如**mWindowSession**、**mWindow**、**mAttachInfo**等，还有一些标志变量如*mAdded*（*view*已添加）、*mFirst*（*view*首次添加）等。

③、缓存要添加的*view*对象、创建的*ViewRootImpl*对象等

④、调用*ViewRootImpl*的`setView`方法，`setView`方法是与*WMS*进行沟通的桥梁。

### 2、setView

*addView*方法做了一些前期准备工作，然后调用*setView*开始真正添加视图，*setView*的代码如下（有删减）。

```
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
        synchronized (this) {
            if (mView == null) {
                mView = view;//初始化mView

                mAttachInfo.mDisplayState = mDisplay.getState();
                mDisplayManager.registerDisplayListener(mDisplayListener, mHandler);

                mViewLayoutDirectionInitial = mView.getRawLayoutDirection();
                mFallbackEventHandler.setView(view);
                mWindowAttributes.copyFrom(attrs);
                if (mWindowAttributes.packageName == null) {
                    mWindowAttributes.packageName = mBasePackageName;
                }
                attrs = mWindowAttributes;
                setTag();

                ......
                mAdded = true;//view已添加
                int res; /* = WindowManagerImpl.ADD_OKAY; */

                // Schedule the first layout -before- adding to the window
                // manager, to make sure we do the relayout before receiving
                // any other events from the system.
                // ①
                requestLayout();
                if ((mWindowAttributes.inputFeatures
                        & WindowManager.LayoutParams.INPUT_FEATURE_NO_INPUT_CHANNEL) == 0) {
                    mInputChannel = new InputChannel();
                }
                mForceDecorViewVisibility = (mWindowAttributes.privateFlags
                        & PRIVATE_FLAG_FORCE_DECOR_VIEW_VISIBILITY) != 0;
                try {
                    mOrigWindowType = mWindowAttributes.type;
                    mAttachInfo.mRecomputeGlobalAttributes = true;
                    collectViewAttributes();
                    // ②
                    res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                            getHostVisibility(), mDisplay.getDisplayId(), mWinFrame,
                            mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                            mAttachInfo.mOutsets, mAttachInfo.mDisplayCutout, mInputChannel);
                } catch (RemoteException e) {
                    mAdded = false;
                    mView = null;
                    mAttachInfo.mRootView = null;
                    mInputChannel = null;
                    mFallbackEventHandler.setView(null);
                    unscheduleTraversals();
                    setAccessibilityFocus(null, null);
                    throw new RuntimeException("Adding window failed", e);
                } finally {
                    if (restore) {
                        attrs.restore();
                    }
                }
								......
                if (res < WindowManagerGlobal.ADD_OKAY) {
                    mAttachInfo.mRootView = null;
                    mAdded = false;
                    mFallbackEventHandler.setView(null);
                    unscheduleTraversals();
                    setAccessibilityFocus(null, null);
                    switch (res) {
                        ......
                    }
                   ......
                }
               ......
            }
        }
    }
```

①、调用`requestLayout`方法

```
 @Override
    public void requestLayout() {
        if (!mHandlingLayoutInLayoutRequest) {
            checkThread();
            mLayoutRequested = true;
            scheduleTraversals();
        }
    }
```

检查当前线程是否是UI线程，然后调用`scheduleTraversals`方法，最终调用到`performTraversals`方法，这就是*View*测量、布局和绘制的起点。

②、*mWindowSession.addToDisplay*，*mWindowSession*最终是由WMS创建并返回，是一个单例，即一个进程只有这么一个*mWindowSession*对象。*APP*进程就是通过*mWindowSession*跟WMS通讯的。

关于*mWindowSession*涉及到*APP*进程和WMS的交互和*binder*进程间通信，详细原理和实现这里暂不详述，后续会整理和输出相关文档。

## 2.2 removeView

移除视图时，使用方式和添加*View*类似，调用`WindowManagerImpl#removeView`或`removeViewImmediate`方法，二者区别下面会详细说道。

### 1、移除View

先看`removeView`方法，还是由*WindowManagerGlobal*具体实现该方法，代码如下。

```
// ①
public void removeView(View view, boolean immediate) {
        if (view == null) {
            throw new IllegalArgumentException("view must not be null");
        }
        synchronized (mLock) {
            // ②
            int index = findViewLocked(view, true);
            View curView = mRoots.get(index).getView();
          	// ③
            removeViewLocked(index, immediate);
            if (curView == view) {
                return;
            }
            throw new IllegalStateException("Calling with view " + view
                    + " but the ViewAncestor is attached to " + curView);
        }
    }
```

①、调用`WindowManagerImpl#removeView`方法时，参数*immediate*为*false*。

②、还是先计算要移除*view*的位置索引，然后通过该索引找到*ViewRootImpl*中缓存的*view*。2个*view*相同则正常结束，否则抛出异常。

③、调用`removeViewLocked`移除*view*。

### 2、removeViewLocked

```
private void removeViewLocked(int index, boolean immediate) {
  		// ①
        ViewRootImpl root = mRoots.get(index);
        View view = root.getView();

        if (view != null) {
            InputMethodManager imm = InputMethodManager.getInstance();
            if (imm != null) {
                imm.windowDismissed(mViews.get(index).getWindowToken());
            }
        }
  		// ②
        boolean deferred = root.die(immediate);
        if (view != null) {
            view.assignParent(null);
            if (deferred) {
              	// ③
                mDyingViews.add(view);
            }
        }
    }
```

①、获取要移除*view*对应的*ViewRootImpl*对象，拿到*ViewRootImpl*对象后，获取*mView*成员变量，它缓存着要移除的*view*对象。*mView*的赋值是在`addView`时完成。

②、调用`ViewRootImpl#die`方法移除*view*

③、对于*immediate*为*false*的情况，调用`die`方法后会返回*true*，所以*deferred* = *true*，表示延迟移除*view*。**此时将待移除的view加入mDyingViews列表缓存**。

### 3、die

代码如下👇

```
/**
     * @param immediate True, do now if not in traversal. False, put on queue and do later.
     * @return True, request has been queued. False, request has been completed.
     */
    boolean die(boolean immediate) {
        // Make sure we do execute immediately if we are in the middle of a traversal or the damage
        // done by dispatchDetachedFromWindow will cause havoc on return.
      	// ①
        if (immediate && !mIsInTraversal) {
            doDie();
            return false;
        }

        if (!mIsDrawing) {
            destroyHardwareRenderer();
        } else {
            Log.e(mTag, "Attempting to destroy the window while drawing!\n" +
                    "  window=" + this + ", title=" + mWindowAttributes.getTitle());
        }
      	// ②
        mHandler.sendEmptyMessage(MSG_DIE);
        return true;
    }
```

由方法注释大致可知，如果参数*immediate*为*true*，且*view*不处于测量、布局和绘制中。（*mIsInTraversal*在`performTraversals`方法中赋值为*true*），立刻移除该*view*。否则将其放入队列稍后执行。所以

①、对于`removeView`方法，*immediate*为*false*，不会走到`doDie`方法

②、发送一个消息（*MSG_DIE*）异步处理*view*的移除，查看*Handler*的代码

```
 @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                ......
                case MSG_DIE:
                    doDie();
                    break;
                ......
            }
        }
    }
```

### 4、doDie

根据`die`方法的代码分析，`removeView`时会发送一个消息异步处理移除*View*，*Handler*处理时，调用了`doDie`方法。代码如下👇

```
void doDie() {
        checkThread();
	 			......
        synchronized (this) {
            if (mRemoved) {// 首次移除，mRemoved为false
                return;
            }
            mRemoved = true;
            // ①
            if (mAdded) {
                dispatchDetachedFromWindow();
            }
			// ②
            if (mAdded && !mFirst) {
                destroyHardwareRenderer();

                if (mView != null) {
                    int viewVisibility = mView.getVisibility();
                    boolean viewVisibilityChanged = mViewVisibility != viewVisibility;
                    if (mWindowAttributesChanged || viewVisibilityChanged) {
                        // If layout params have been changed, first give them
                        // to the window manager to make sure it has the correct
                        // animation info.
                        try {
                            if ((relayoutWindow(mWindowAttributes, viewVisibility, false)
                                    & WindowManagerGlobal.RELAYOUT_RES_FIRST_TIME) != 0) {
                                mWindowSession.finishDrawing(mWindow);
                            }
                        } catch (RemoteException e) {
                        }
                    }

                    mSurface.release();
                }
            }

            mAdded = false;
        }
  		// ③
        WindowManagerGlobal.getInstance().doRemoveView(this);
    }
```

①、在调用`addView`方法时，*mAdded*变量赋为*true*，故会调用`dispatchDetachedFromWindow`方法。

```
void dispatchDetachedFromWindow() {
        mFirstInputStage.onDetachedFromWindow();
        if (mView != null && mView.mAttachInfo != null) {
            mAttachInfo.mTreeObserver.dispatchOnWindowAttachedChange(false);
            // ①
            mView.dispatchDetachedFromWindow();
        }

        mAccessibilityInteractionConnectionManager.ensureNoConnection();
        mAccessibilityManager.removeAccessibilityStateChangeListener(
                mAccessibilityInteractionConnectionManager);
        mAccessibilityManager.removeHighTextContrastStateChangeListener(
                mHighContrastTextManager);
        removeSendWindowContentChangedCallback();

        destroyHardwareRenderer();

        setAccessibilityFocus(null, null);

        mView.assignParent(null);
        mView = null;
        mAttachInfo.mRootView = null;

        mSurface.release();
				......
        try {
          	// ②
            mWindowSession.remove(mWindow);
        } catch (RemoteException e) {
        }

        // Dispose the input channel after removing the window so the Window Manager
        // doesn't interpret the input channel being closed as an abnormal termination.
        if (mInputChannel != null) {
            mInputChannel.dispose();
            mInputChannel = null;
        }

        mDisplayManager.unregisterDisplayListener(mDisplayListener);

        unscheduleTraversals();
    }
```

该方法就是真正移除*view*之处，包含资源的释放和重置。`dispatchDetachedFromWindow`会调用`onDetachedFromWindow`，我们可以重写`onDetachedFromWindow`实现一些资源释放的工作。然后在`mWindowSession.remove(mWindow)`;处就移除*view*了。

②、*mFirst*变量在`addView`方法中创建*ViewRootImpl*对象时初始化为*true*，在`performTraversals`完成*View*布局后（`performLayout`）和绘制前（`performDraw`），还原为*false*。对于成功绘制并展示的*view*，*mFirst*为*false*，那么就会进入条件体内。

③、调用`WindowManagerGlobal#doRemove`方法，代码如下

```
void doRemoveView(ViewRootImpl root) {
        synchronized (mLock) {
            final int index = mRoots.indexOf(root);
            if (index >= 0) {
                mRoots.remove(index);
                mParams.remove(index);
                final View view = mViews.remove(index);
                mDyingViews.remove(view);
            }
        }
        if (ThreadedRenderer.sTrimForeground && ThreadedRenderer.isAvailable()) {
            doTrimForeground();
        }
    }
```

该方法就是移除*mRoots*、*mParams*、*mViews*、*mDyingViews*等列表保存的数据。

## 2.3 小结

通过对*removeView*流程的梳理和说明，我们明白了*view*移除过程所做的操作。简单总结一下就是

1. **removeView时并非立即将view销毁，而是将其放入一个列表进行缓存，然后通过handler发送移除view的消息MSG_DIE到消息队列，等待处理。**
2. 如果需要立即移除*view*，可以调用`removeViewImmdiate`方法。
3. 在`addView`时，如果*view*已经添加过了（位置索引大于等于0），那么会先判断一下是否在待移除*view*列表中（*mDyingViews*）。**如果在，则不再等待MSG_DIE消息被执行，而是主动调用doDie方法立刻移除。否则，抛出view已添加的异常（**new IllegalStateException("View has already been added to the window manager.") **）。**
