# 一、Fragment

## 1.1 生命周期

官方网上的Fragment声明周期。

![image-20211022174424557](/Users/mtdp/Library/Application Support/typora-user-images/image-20211022174424557.png)	

​	图 1 Fragment生命周期流程图

除了上述声明周期之外，也可以注册*FragmentLifecycleCallbacks*来插入更多*Fragment*状态的监听，具体有：

```java
public abstract static class FragmentLifecycleCallbacks {
        /**
         * Called right before the fragment's {@link Fragment#onAttach(Context)} method is called.
         * This is a good time to inject any required dependencies or perform other configuration
         * for the fragment before any of the fragment's lifecycle methods are invoked.
         *
         * @param fm Host FragmentManager
         * @param f Fragment changing state
         * @param context Context that the Fragment is being attached to
         */
        public void onFragmentPreAttached(@NonNull FragmentManager fm, @NonNull Fragment f,
                @NonNull Context context) {}

        /**
         * Called after the fragment has been attached to its host. Its host will have had
         * <code>onAttachFragment</code> called before this call happens.
         *
         * @param fm Host FragmentManager
         * @param f Fragment changing state
         * @param context Context that the Fragment was attached to
         */
        public void onFragmentAttached(@NonNull FragmentManager fm, @NonNull Fragment f,
                @NonNull Context context) {}

        /**
         * Called right before the fragment's {@link Fragment#onCreate(Bundle)} method is called.
         * This is a good time to inject any required dependencies or perform other configuration
         * for the fragment.
         *
         * @param fm Host FragmentManager
         * @param f Fragment changing state
         * @param savedInstanceState Saved instance bundle from a previous instance
         */
        public void onFragmentPreCreated(@NonNull FragmentManager fm, @NonNull Fragment f,
                @Nullable Bundle savedInstanceState) {}

        /**
         * Called after the fragment has returned from the FragmentManager's call to
         * {@link Fragment#onCreate(Bundle)}. This will only happen once for any given
         * fragment instance, though the fragment may be attached and detached multiple times.
         *
         * @param fm Host FragmentManager
         * @param f Fragment changing state
         * @param savedInstanceState Saved instance bundle from a previous instance
         */
        public void onFragmentCreated(@NonNull FragmentManager fm, @NonNull Fragment f,
                @Nullable Bundle savedInstanceState) {}

        /**
         * Called after the fragment has returned from the FragmentManager's call to
         * {@link Fragment#onActivityCreated(Bundle)}. This will only happen once for any given
         * fragment instance, though the fragment may be attached and detached multiple times.
         *
         * @param fm Host FragmentManager
         * @param f Fragment changing state
         * @param savedInstanceState Saved instance bundle from a previous instance
         */
        public void onFragmentActivityCreated(@NonNull FragmentManager fm, @NonNull Fragment f,
                @Nullable Bundle savedInstanceState) {}

        /**
         * Called after the fragment has returned a non-null view from the FragmentManager's
         * request to {@link Fragment#onCreateView(LayoutInflater, ViewGroup, Bundle)}.
         *
         * @param fm Host FragmentManager
         * @param f Fragment that created and owns the view
         * @param v View returned by the fragment
         * @param savedInstanceState Saved instance bundle from a previous instance
         */
        public void onFragmentViewCreated(@NonNull FragmentManager fm, @NonNull Fragment f,
                @NonNull View v, @Nullable Bundle savedInstanceState) {}

        /**
         * Called after the fragment has returned from the FragmentManager's call to
         * {@link Fragment#onStart()}.
         *
         * @param fm Host FragmentManager
         * @param f Fragment changing state
         */
        public void onFragmentStarted(@NonNull FragmentManager fm, @NonNull Fragment f) {}

        /**
         * Called after the fragment has returned from the FragmentManager's call to
         * {@link Fragment#onResume()}.
         *
         * @param fm Host FragmentManager
         * @param f Fragment changing state
         */
        public void onFragmentResumed(@NonNull FragmentManager fm, @NonNull Fragment f) {}

        /**
         * Called after the fragment has returned from the FragmentManager's call to
         * {@link Fragment#onPause()}.
         *
         * @param fm Host FragmentManager
         * @param f Fragment changing state
         */
        public void onFragmentPaused(@NonNull FragmentManager fm, @NonNull Fragment f) {}

        /**
         * Called after the fragment has returned from the FragmentManager's call to
         * {@link Fragment#onStop()}.
         *
         * @param fm Host FragmentManager
         * @param f Fragment changing state
         */
        public void onFragmentStopped(@NonNull FragmentManager fm, @NonNull Fragment f) {}

        /**
         * Called after the fragment has returned from the FragmentManager's call to
         * {@link Fragment#onSaveInstanceState(Bundle)}.
         *
         * @param fm Host FragmentManager
         * @param f Fragment changing state
         * @param outState Saved state bundle for the fragment
         */
        public void onFragmentSaveInstanceState(@NonNull FragmentManager fm, @NonNull Fragment f,
                @NonNull Bundle outState) {}

        /**
         * Called after the fragment has returned from the FragmentManager's call to
         * {@link Fragment#onDestroyView()}.
         *
         * @param fm Host FragmentManager
         * @param f Fragment changing state
         */
        public void onFragmentViewDestroyed(@NonNull FragmentManager fm, @NonNull Fragment f) {}

        /**
         * Called after the fragment has returned from the FragmentManager's call to
         * {@link Fragment#onDestroy()}.
         *
         * @param fm Host FragmentManager
         * @param f Fragment changing state
         */
        public void onFragmentDestroyed(@NonNull FragmentManager fm, @NonNull Fragment f) {}

        /**
         * Called after the fragment has returned from the FragmentManager's call to
         * {@link Fragment#onDetach()}.
         *
         * @param fm Host FragmentManager
         * @param f Fragment changing state
         */
        public void onFragmentDetached(@NonNull FragmentManager fm, @NonNull Fragment f) {}
    }
```

所以，如果考虑开发中自己注册的状态变化监听，那么其方法调用过程如下图

![image-20211022182154551](/Users/mtdp/Library/Application Support/typora-user-images/image-20211022182154551.png)

​											图 2 Fragment生命周期及FragmentLifecycleCallbacks流程图

## 1.2 生命周期执行顺序源码解析

### 1、Fragment的提交并展示

通常我们使用下列代码来提交并展示一个*Fragment*

```java
XXXFragment fragment = new XXXFragment();
Bundle bundle = new Bundle();
bundle.putString("key", "value");
fragment.setAurguments(bundle);
// 或者commitAllowStateLoss
getSupportFragmentManager.beginTransaction().add(R.id.xxx, fragment).commit(); 
```

添加Fragment视图的方式有很多，例如*add*()、*replace*()  

这里已add()方法为例，看下代码实现

```
// FragmentTransaction.java
......
/**
* Calls {@link #add(int, Fragment, String)} with a 0 containerViewId.
*/
@NonNull
public FragmentTransaction add(@NonNull Fragment fragment, @Nullable String tag)  {
	doAddOp(0, fragment, tag, OP_ADD);
	return this;
}
...... 
void doAddOp(int containerViewId, Fragment fragment, @Nullable String tag, int opcmd) {
        final Class<?> fragmentClass = fragment.getClass();
        final int modifiers = fragmentClass.getModifiers();
        if (fragmentClass.isAnonymousClass() || !Modifier.isPublic(modifiers)
                || (fragmentClass.isMemberClass() && !Modifier.isStatic(modifiers))) {
            throw new IllegalStateException("Fragment " + fragmentClass.getCanonicalName()
                    + " must be a public static class to be  properly recreated from"
                    + " instance state.");
        }
        if (tag != null) {
            if (fragment.mTag != null && !tag.equals(fragment.mTag)) {
                throw new IllegalStateException("Can't change tag of fragment "
                        + fragment + ": was " + fragment.mTag
                        + " now " + tag);
            }
            fragment.mTag = tag;
        }
        if (containerViewId != 0) {
            if (containerViewId == View.NO_ID) {
                throw new IllegalArgumentException("Can't add fragment "
                        + fragment + " with tag " + tag + " to container view with no id");
            }
            if (fragment.mFragmentId != 0 && fragment.mFragmentId != containerViewId) {
                throw new IllegalStateException("Can't change container ID of fragment "
                        + fragment + ": was " + fragment.mFragmentId
                        + " now " + containerViewId);
            }
            fragment.mContainerId = fragment.mFragmentId = containerViewId;
        }
        addOp(new Op(opcmd, fragment));
    }
......
void addOp(Op op) {
        mOps.add(op);
        op.mEnterAnim = mEnterAnim;
        op.mExitAnim = mExitAnim;
        op.mPopEnterAnim = mPopEnterAnim;
        op.mPopExitAnim = mPopExitAnim;
    }
```

代码看似很多，其实主要目的是将*OP_ADD*这一指令保存在列表中，然后接着看*commit*()

```java
// BackStackRecord.java

 @Override
public int commit() {
	return commitInternal(false);
}

int commitInternal(boolean allowStateLoss) {
	if (mCommitted) throw new IllegalStateException("commit already called");
	if (FragmentManagerImpl.DEBUG) {
		Log.v(TAG, "Commit: " + this);
		LogWriter logw = new LogWriter(TAG);
		PrintWriter pw = new PrintWriter(logw);
		dump("  ", pw);
		pw.close();	
  }
	mCommitted = true;
	if (mAddToBackStack) {
		mIndex = mManager.allocBackStackIndex(this);
	} else {
		mIndex = -1;
	}
  // ① 
	mManager.enqueueAction(this, allowStateLoss);
	return mIndex;
}
```

①：将提交Fragment事件添加到执行队列中等待执行。

继续深入查看代码，*mManager*是*FragmentManagerImpl*对象，执行它的*enqueueAction*()方法。

```java
 		// FragmentManagerImpl.java

		/**
     * Adds an action to the queue of pending actions.
     *
     * @param action the action to add
     * @param allowStateLoss whether to allow loss of state information
     * @throws IllegalStateException if the activity has been destroyed
     */
    public void enqueueAction(OpGenerator action, boolean allowStateLoss) {
        if (!allowStateLoss) {
            checkStateLoss();
        }
        synchronized (this) {
            if (mDestroyed || mHost == null) {
                if (allowStateLoss) {
                    // This FragmentManager isn't attached, so drop the entire transaction.
                    return;
                }
                throw new IllegalStateException("Activity has been destroyed");
            }
            if (mPendingActions == null) {
                mPendingActions = new ArrayList<>();
            }
            // ②
            mPendingActions.add(action);
            scheduleCommit();
        }
    }

		/**
     * Schedules the execution when one hasn't been scheduled already. This should happen
     * the first time {@link #enqueueAction(OpGenerator, boolean)} is called or when
     * a postponed transaction has been started with
     * {@link Fragment#startPostponedEnterTransition()}
     */
    @SuppressWarnings("WeakerAccess") /* synthetic access */
    void scheduleCommit() {
        synchronized (this) {
            boolean postponeReady =
                    mPostponedTransactions != null && !mPostponedTransactions.isEmpty();
          	// ③
            boolean pendingReady = mPendingActions != null && mPendingActions.size() == 1;
            if (postponeReady || pendingReady) {
                mHost.getHandler().removeCallbacks(mExecCommit);
              	// ④
                mHost.getHandler().post(mExecCommit);
                updateOnBackPressedCallbackEnabled();
            }
        }
    }

		Runnable mExecCommit = new Runnable() {
        @Override
        public void run() {
            execPendingActions();
        }
    };

		/**
     * Only call from main thread!
     */
    public boolean execPendingActions() {
        ensureExecReady(true);

        boolean didSomething = false;
        while (generateOpsForPendingActions(mTmpRecords, mTmpIsPop)) {
            mExecutingActions = true;
            try {
                removeRedundantOperationsAndExecute(mTmpRecords, mTmpIsPop);
            } finally {
                cleanupExec();
            }
            didSomething = true;
        }

        updateOnBackPressedCallbackEnabled();
        doPendingDeferredStart();
        burpActive();

        return didSomething;
    }

		private void removeRedundantOperationsAndExecute(ArrayList<BackStackRecord> records,
                                                     ArrayList<Boolean> isRecordPop) {
       ......
        // Force start of any postponed transactions that interact with scheduled transactions:
        executePostponedTransaction(records, isRecordPop);

        final int numRecords = records.size();
        int startIndex = 0;
        for (int recordNum = 0; recordNum < numRecords; recordNum++) {
            final boolean canReorder = records.get(recordNum).mReorderingAllowed;
            if (!canReorder) {
                // execute all previous transactions
                if (startIndex != recordNum) {
                    executeOpsTogether(records, isRecordPop, startIndex, recordNum);
                }
                // execute all pop operations that don't allow reordering together or
                // one add operation
                int reorderingEnd = recordNum + 1;
                if (isRecordPop.get(recordNum)) {
                    while (reorderingEnd < numRecords
                            && isRecordPop.get(reorderingEnd)
                            && !records.get(reorderingEnd).mReorderingAllowed) {
                        reorderingEnd++;
                    }
                }
              	// ⑤
                executeOpsTogether(records, isRecordPop, recordNum, reorderingEnd);
                startIndex = reorderingEnd;
                recordNum = reorderingEnd - 1;
            }
        }
        if (startIndex != numRecords) {
            executeOpsTogether(records, isRecordPop, startIndex, numRecords);
        }
    }

	private void executeOpsTogether(ArrayList<BackStackRecord> records,
                                    ArrayList<Boolean> isRecordPop, int startIndex, int endIndex) {
        ......
        if (!allowReordering) {
            FragmentTransition.startTransitions(this, records, isRecordPop, startIndex, endIndex,
                    false);
        }
        executeOps(records, isRecordPop, startIndex, endIndex);

        ......
        for (int recordNum = startIndex; recordNum < endIndex; recordNum++) {
            final BackStackRecord record = records.get(recordNum);
            final boolean isPop = isRecordPop.get(recordNum);
            if (isPop && record.mIndex >= 0) {
                freeBackStackIndex(record.mIndex);
                record.mIndex = -1;
            }
            record.runOnCommitRunnables();
        }
        if (addToBackStack) {
            reportBackStackChanged();
        }
    }

		private static void executeOps(ArrayList<BackStackRecord> records,
                                   ArrayList<Boolean> isRecordPop, int startIndex, int endIndex) {
        for (int i = startIndex; i < endIndex; i++) {
            final BackStackRecord record = records.get(i);
            final boolean isPop = isRecordPop.get(i);
            if (isPop) {
                record.bumpBackStackNesting(-1);
                // Only execute the add operations at the end of
                // all transactions.
                boolean moveToState = i == (endIndex - 1);
                record.executePopOps(moveToState);
            } else {
                record.bumpBackStackNesting(1);
                record.executeOps();
            }
        }
    }
```

②：首先初始化*mPendingActions，*并将事件添加到*mPendingActions*列表中。然后执行 *scheduleCommit()。*

③：在*scheduleCommit*方法中，由于*mPendingActions*已经初始化过，且大小为1。*pendingReady*标记位为*true，*所以执行步骤④

④：通过Handler发送添加*Fragment*的执行事件，开始创建*Fragment*对象，执行各生命周期方法等。

⑤：执行*executeOps*方法，然后执行 *record.executeOps()*

```java
// BackStackRecord.java
void executeOps() {
        final int numOps = mOps.size();
        for (int opNum = 0; opNum < numOps; opNum++) {
            final Op op = mOps.get(opNum);
            final Fragment f = op.mFragment;
            if (f != null) {
                f.setNextTransition(mTransition, mTransitionStyle);
            }
            switch (op.mCmd) {
                case OP_ADD:
                    f.setNextAnim(op.mEnterAnim);
                		// ⑥
                    mManager.addFragment(f, false);
                    break;
                case OP_REMOVE:
                    f.setNextAnim(op.mExitAnim);
                    mManager.removeFragment(f);
                    break;
                case OP_HIDE:
                    f.setNextAnim(op.mExitAnim);
                    mManager.hideFragment(f);
                    break;
                case OP_SHOW:
                    f.setNextAnim(op.mEnterAnim);
                    mManager.showFragment(f);
                    break;
                case OP_DETACH:
                    f.setNextAnim(op.mExitAnim);
                    mManager.detachFragment(f);
                    break;
                case OP_ATTACH:
                    f.setNextAnim(op.mEnterAnim);
                    mManager.attachFragment(f);
                    break;
                case OP_SET_PRIMARY_NAV:
                    mManager.setPrimaryNavigationFragment(f);
                    break;
                case OP_UNSET_PRIMARY_NAV:
                    mManager.setPrimaryNavigationFragment(null);
                    break;
                case OP_SET_MAX_LIFECYCLE:
                    mManager.setMaxLifecycle(f, op.mCurrentMaxState);
                    break;
                default:
                    throw new IllegalArgumentException("Unknown cmd: " + op.mCmd);
            }
            if (!mReorderingAllowed && op.mCmd != OP_ADD && f != null) {
                mManager.moveFragmentToExpectedState(f);
            }
        }
        if (!mReorderingAllowed) {
          // ⑦
            // Added fragments are added at the end to comply with prior behavior.
            mManager.moveToState(mManager.mCurState, true);
        }
    }
```

⑥：还记得在调用add方法时，将*OP_ADD*指令保存了下来，所以这里在会调用 *mManager.addFragment(f, false)，*该方法主要做一些状态和标记位的设置。

```java
// FragmentManagerImpl.java

public void addFragment(Fragment fragment, boolean moveToStateNow) {
        if (DEBUG) Log.v(TAG, "add: " + fragment);
        makeActive(fragment);
        if (!fragment.mDetached) {
            if (mAdded.contains(fragment)) {
                throw new IllegalStateException("Fragment already added: " + fragment);
            }
            synchronized (mAdded) {
                mAdded.add(fragment);
            }
            fragment.mAdded = true;
            fragment.mRemoving = false;
            if (fragment.mView == null) {
                fragment.mHiddenChanged = false;
            }
            if (isMenuAvailable(fragment)) {
                mNeedMenuInvalidate = true;
            }
            if (moveToStateNow) {
                moveToState(fragment);
            }
        }
    }
```

⑦：*mManager.moveToState*将fragment当前状态转化为下一个新状态

```java
/**
     * Changes the state of the fragment manager to {@code newState}. If the fragment manager
     * changes state or {@code always} is {@code true}, any fragments within it have their
     * states updated as well.
     *
     * @param newState The new state for the fragment manager
     * @param always If {@code true}, all fragments update their state, even
     *               if {@code newState} matches the current fragment manager's state.
     */
    void moveToState(int newState, boolean always) {
        if (mHost == null && newState != Fragment.INITIALIZING) {
            throw new IllegalStateException("No activity");
        }

        if (!always && newState == mCurState) {
            return;
        }

        mCurState = newState;

        // Must add them in the proper order. mActive fragments may be out of order
        final int numAdded = mAdded.size();
        for (int i = 0; i < numAdded; i++) {
            Fragment f = mAdded.get(i);
            moveFragmentToExpectedState(f);
        }

        // Now iterate through all active fragments. These will include those that are removed
        // and detached.
        for (Fragment f : mActive.values()) {
            if (f != null && (f.mRemoving || f.mDetached) && !f.mIsNewlyAdded) {
                moveFragmentToExpectedState(f);
            }
        }

        startPendingDeferredFragments();

        if (mNeedMenuInvalidate && mHost != null && mCurState == Fragment.RESUMED) {
            mHost.onSupportInvalidateOptionsMenu();
            mNeedMenuInvalidate = false;
        }
    }

 void moveFragmentToExpectedState(Fragment f) {
        if (f == null) {
            return;
        }
        if (!mActive.containsKey(f.mWho)) {
            if (DEBUG) {
                Log.v(TAG, "Ignoring moving " + f + " to state " + mCurState
                        + "since it is not added to " + this);
            }
            return;
        }
        int nextState = mCurState;
        if (f.mRemoving) {
            if (f.isInBackStack()) {
                nextState = Math.min(nextState, Fragment.CREATED);
            } else {
                nextState = Math.min(nextState, Fragment.INITIALIZING);
            }
        }
   			// ⑧
        moveToState(f, nextState, f.getNextTransition(), f.getNextTransitionStyle(), false);
        ......
    }
```

⑧：这个方法就是Fragment添加到展示过程中，各生命周期方法和注册的FragmentLifecycleCallbacks监听的具体执行位置。该方法源码如下

```java
@SuppressWarnings("ReferenceEquality")
    void moveToState(Fragment f, int newState, int transit, int transitionStyle,
                     boolean keepActive) {
        // Fragments that are not currently added will sit in the onCreate() state.
        if ((!f.mAdded || f.mDetached) && newState > Fragment.CREATED) {
            newState = Fragment.CREATED;
        }
        if (f.mRemoving && newState > f.mState) {
            if (f.mState == Fragment.INITIALIZING && f.isInBackStack()) {
                // Allow the fragment to be created so that it can be saved later.
                newState = Fragment.CREATED;
            } else {
                // While removing a fragment, we can't change it to a higher state.
                newState = f.mState;
            }
        }
        // Defer start if requested; don't allow it to move to STARTED or higher
        // if it's not already started.
        if (f.mDeferStart && f.mState < Fragment.STARTED && newState > Fragment.ACTIVITY_CREATED) {
            newState = Fragment.ACTIVITY_CREATED;
        }
        // Don't allow the Fragment to go above its max lifecycle state
        // Ensure that Fragments are capped at CREATED instead of ACTIVITY_CREATED.
        if (f.mMaxState == Lifecycle.State.CREATED) {
            newState = Math.min(newState, Fragment.CREATED);
        } else {
            newState = Math.min(newState, f.mMaxState.ordinal());
        }
        if (f.mState <= newState) {
            // For fragments that are created from a layout, when restoring from
            // state we don't want to allow them to be created until they are
            // being reloaded from the layout.
            if (f.mFromLayout && !f.mInLayout) {
                return;
            }
            if (f.getAnimatingAway() != null || f.getAnimator() != null) {
                // The fragment is currently being animated...  but!  Now we
                // want to move our state back up.  Give up on waiting for the
                // animation, move to whatever the final state should be once
                // the animation is done, and then we can proceed from there.
                f.setAnimatingAway(null);
                f.setAnimator(null);
                moveToState(f, f.getStateAfterAnimating(), 0, 0, true);
            }
            switch (f.mState) {
                case Fragment.INITIALIZING:
                    if (newState > Fragment.INITIALIZING) {
                        if (DEBUG) Log.v(TAG, "moveto CREATED: " + f);
                        if (f.mSavedFragmentState != null) {
                            f.mSavedFragmentState.setClassLoader(mHost.getContext()
                                    .getClassLoader());
                            f.mSavedViewState = f.mSavedFragmentState.getSparseParcelableArray(
                                    FragmentManagerImpl.VIEW_STATE_TAG);
                            Fragment target = getFragment(f.mSavedFragmentState,
                                    FragmentManagerImpl.TARGET_STATE_TAG);
                            f.mTargetWho = target != null ? target.mWho : null;
                            if (f.mTargetWho != null) {
                                f.mTargetRequestCode = f.mSavedFragmentState.getInt(
                                        FragmentManagerImpl.TARGET_REQUEST_CODE_STATE_TAG, 0);
                            }
                            if (f.mSavedUserVisibleHint != null) {
                                f.mUserVisibleHint = f.mSavedUserVisibleHint;
                                f.mSavedUserVisibleHint = null;
                            } else {
                                f.mUserVisibleHint = f.mSavedFragmentState.getBoolean(
                                        FragmentManagerImpl.USER_VISIBLE_HINT_TAG, true);
                            }
                            if (!f.mUserVisibleHint) {
                                f.mDeferStart = true;
                                if (newState > Fragment.ACTIVITY_CREATED) {
                                    newState = Fragment.ACTIVITY_CREATED;
                                }
                            }
                        }

                        f.mHost = mHost;
                        f.mParentFragment = mParent;
                        f.mFragmentManager = mParent != null
                                ? mParent.mChildFragmentManager : mHost.mFragmentManager;

                        // If we have a target fragment, push it along to at least CREATED
                        // so that this one can rely on it as an initialized dependency.
                        if (f.mTarget != null) {
                            if (mActive.get(f.mTarget.mWho) != f.mTarget) {
                                throw new IllegalStateException("Fragment " + f
                                        + " declared target fragment " + f.mTarget
                                        + " that does not belong to this FragmentManager!");
                            }
                            if (f.mTarget.mState < Fragment.CREATED) {
                                moveToState(f.mTarget, Fragment.CREATED, 0, 0, true);
                            }
                            f.mTargetWho = f.mTarget.mWho;
                            f.mTarget = null;
                        }
                        if (f.mTargetWho != null) {
                            Fragment target = mActive.get(f.mTargetWho);
                            if (target == null) {
                                throw new IllegalStateException("Fragment " + f
                                        + " declared target fragment " + f.mTargetWho
                                        + " that does not belong to this FragmentManager!");
                            }
                            if (target.mState < Fragment.CREATED) {
                                moveToState(target, Fragment.CREATED, 0, 0, true);
                            }
                        }

                        dispatchOnFragmentPreAttached(f, mHost.getContext(), false);
                        f.performAttach();
                        if (f.mParentFragment == null) {
                            mHost.onAttachFragment(f);
                        } else {
                            f.mParentFragment.onAttachFragment(f);
                        }
                        dispatchOnFragmentAttached(f, mHost.getContext(), false);

                        if (!f.mIsCreated) {
                            dispatchOnFragmentPreCreated(f, f.mSavedFragmentState, false);
                            f.performCreate(f.mSavedFragmentState);
                            dispatchOnFragmentCreated(f, f.mSavedFragmentState, false);
                        } else {
                            f.restoreChildFragmentState(f.mSavedFragmentState);
                            f.mState = Fragment.CREATED;
                        }
                    }
                    // fall through
                case Fragment.CREATED:
                    // We want to unconditionally run this anytime we do a moveToState that
                    // moves the Fragment above INITIALIZING, including cases such as when
                    // we move from CREATED => CREATED as part of the case fall through above.
                    if (newState > Fragment.INITIALIZING) {
                        ensureInflatedFragmentView(f);
                    }

                    if (newState > Fragment.CREATED) {
                        if (DEBUG) Log.v(TAG, "moveto ACTIVITY_CREATED: " + f);
                        if (!f.mFromLayout) {
                            ViewGroup container = null;
                            if (f.mContainerId != 0) {
                                if (f.mContainerId == View.NO_ID) {
                                    throwException(new IllegalArgumentException(
                                            "Cannot create fragment "
                                                    + f
                                                    + " for a container view with no id"));
                                }
                                container = (ViewGroup) mContainer.onFindViewById(f.mContainerId);
                                if (container == null && !f.mRestored) {
                                    String resName;
                                    try {
                                        resName = f.getResources().getResourceName(f.mContainerId);
                                    } catch (Resources.NotFoundException e) {
                                        resName = "unknown";
                                    }
                                    throwException(new IllegalArgumentException(
                                            "No view found for id 0x"
                                                    + Integer.toHexString(f.mContainerId) + " ("
                                                    + resName
                                                    + ") for fragment " + f));
                                }
                            }
                            f.mContainer = container;
                            f.performCreateView(f.performGetLayoutInflater(
                                    f.mSavedFragmentState), container, f.mSavedFragmentState);
                            if (f.mView != null) {
                                f.mInnerView = f.mView;
                                f.mView.setSaveFromParentEnabled(false);
                                if (container != null) {
                                    container.addView(f.mView);
                                }
                                if (f.mHidden) {
                                    f.mView.setVisibility(View.GONE);
                                }
                                f.onViewCreated(f.mView, f.mSavedFragmentState);
                                dispatchOnFragmentViewCreated(f, f.mView, f.mSavedFragmentState,
                                        false);
                                // Only animate the view if it is visible. This is done after
                                // dispatchOnFragmentViewCreated in case visibility is changed
                                f.mIsNewlyAdded = (f.mView.getVisibility() == View.VISIBLE)
                                        && f.mContainer != null;
                            } else {
                                f.mInnerView = null;
                            }
                        }

                        f.performActivityCreated(f.mSavedFragmentState);
                        dispatchOnFragmentActivityCreated(f, f.mSavedFragmentState, false);
                        if (f.mView != null) {
                            f.restoreViewState(f.mSavedFragmentState);
                        }
                        f.mSavedFragmentState = null;
                    }
                    // fall through
                case Fragment.ACTIVITY_CREATED:
                    if (newState > Fragment.ACTIVITY_CREATED) {
                        if (DEBUG) Log.v(TAG, "moveto STARTED: " + f);
                        f.performStart();
                        dispatchOnFragmentStarted(f, false);
                    }
                    // fall through
                case Fragment.STARTED:
                    if (newState > Fragment.STARTED) {
                        if (DEBUG) Log.v(TAG, "moveto RESUMED: " + f);
                        f.performResume();
                        dispatchOnFragmentResumed(f, false);
                        f.mSavedFragmentState = null;
                        f.mSavedViewState = null;
                    }
            }
        } else if (f.mState > newState) {
            switch (f.mState) {
                case Fragment.RESUMED:
                    if (newState < Fragment.RESUMED) {
                        if (DEBUG) Log.v(TAG, "movefrom RESUMED: " + f);
                        f.performPause();
                        dispatchOnFragmentPaused(f, false);
                    }
                    // fall through
                case Fragment.STARTED:
                    if (newState < Fragment.STARTED) {
                        if (DEBUG) Log.v(TAG, "movefrom STARTED: " + f);
                        f.performStop();
                        dispatchOnFragmentStopped(f, false);
                    }
                    // fall through
                case Fragment.ACTIVITY_CREATED:
                    if (newState < Fragment.ACTIVITY_CREATED) {
                        if (DEBUG) Log.v(TAG, "movefrom ACTIVITY_CREATED: " + f);
                        if (f.mView != null) {
                            // Need to save the current view state if not
                            // done already.
                            if (mHost.onShouldSaveFragmentState(f) && f.mSavedViewState == null) {
                                saveFragmentViewState(f);
                            }
                        }
                        f.performDestroyView();
                        dispatchOnFragmentViewDestroyed(f, false);
                        if (f.mView != null && f.mContainer != null) {
                            // Stop any current animations:
                            f.mContainer.endViewTransition(f.mView);
                            f.mView.clearAnimation();
                            AnimationOrAnimator anim = null;
                            // If parent is being removed, no need to handle child animations.
                            if (f.getParentFragment() == null || !f.getParentFragment().mRemoving) {
                                if (mCurState > Fragment.INITIALIZING && !mDestroyed
                                        && f.mView.getVisibility() == View.VISIBLE
                                        && f.mPostponedAlpha >= 0) {
                                    anim = loadAnimation(f, transit, false,
                                            transitionStyle);
                                }
                                f.mPostponedAlpha = 0;
                                if (anim != null) {
                                    animateRemoveFragment(f, anim, newState);
                                }
                                f.mContainer.removeView(f.mView);
                            }
                        }
                        f.mContainer = null;
                        f.mView = null;
                        // Set here to ensure that Observers are called after
                        // the Fragment's view is set to null
                        f.mViewLifecycleOwner = null;
                        f.mViewLifecycleOwnerLiveData.setValue(null);
                        f.mInnerView = null;
                        f.mInLayout = false;
                    }
                    // fall through
                case Fragment.CREATED:
                    if (newState < Fragment.CREATED) {
                        if (mDestroyed) {
                            // The fragment's containing activity is
                            // being destroyed, but this fragment is
                            // currently animating away.  Stop the
                            // animation right now -- it is not needed,
                            // and we can't wait any more on destroying
                            // the fragment.
                            if (f.getAnimatingAway() != null) {
                                View v = f.getAnimatingAway();
                                f.setAnimatingAway(null);
                                v.clearAnimation();
                            } else if (f.getAnimator() != null) {
                                Animator animator = f.getAnimator();
                                f.setAnimator(null);
                                animator.cancel();
                            }
                        }
                        if (f.getAnimatingAway() != null || f.getAnimator() != null) {
                            // We are waiting for the fragment's view to finish
                            // animating away.  Just make a note of the state
                            // the fragment now should move to once the animation
                            // is done.
                            f.setStateAfterAnimating(newState);
                            newState = Fragment.CREATED;
                        } else {
                            if (DEBUG) Log.v(TAG, "movefrom CREATED: " + f);
                            boolean beingRemoved = f.mRemoving && !f.isInBackStack();
                            if (beingRemoved || mNonConfig.shouldDestroy(f)) {
                                boolean shouldClear;
                                if (mHost instanceof ViewModelStoreOwner) {
                                    shouldClear = mNonConfig.isCleared();
                                } else if (mHost.getContext() instanceof Activity) {
                                    Activity activity = (Activity) mHost.getContext();
                                    shouldClear = !activity.isChangingConfigurations();
                                } else {
                                    shouldClear = true;
                                }
                                if (beingRemoved || shouldClear) {
                                    mNonConfig.clearNonConfigState(f);
                                }
                                f.performDestroy();
                                dispatchOnFragmentDestroyed(f, false);
                            } else {
                                f.mState = Fragment.INITIALIZING;
                            }

                            f.performDetach();
                            dispatchOnFragmentDetached(f, false);
                            if (!keepActive) {
                                if (beingRemoved || mNonConfig.shouldDestroy(f)) {
                                    makeInactive(f);
                                } else {
                                    f.mHost = null;
                                    f.mParentFragment = null;
                                    f.mFragmentManager = null;
                                    if (f.mTargetWho != null) {
                                        Fragment target = mActive.get(f.mTargetWho);
                                        if (target != null && target.getRetainInstance()) {
                                            // Only keep references to other retained Fragments
                                            // to avoid developers accessing Fragments that
                                            // are never coming back
                                            f.mTarget = target;
                                        }
                                    }
                                }
                            }
                        }
                    }
            }
        }

        if (f.mState != newState) {
            Log.w(TAG, "moveToState: Fragment state for " + f + " not updated inline; "
                    + "expected state " + newState + " found " + f.mState);
            f.mState = newState;
        }
    }
```

代码很长，但还是比较容易看出，在*INITIALIZING*状态下，先后执行了*onFragmentPreAttach*、*onAttach*、*onFragmentAttched*、*onFragmentPreCreate*、*onCreate*等

在*CREATE*状态下，执行了*onCreateView*、*onViewCreated*、*onActivityCreated*等方法。

# 二、DialogFragment

## 2.1 DialogFragment生命周期

查看*DialogFragment*的继承关系，其实它继承自*Fragment*，所以具备*Fragment*的特性和生命周期方法。同时，*DialogFragment*也具有一些*Dialog*的特点，反应在生命周期方法调用流程上就是，在*onCreateView*前，会调用*onCreateDialog*方法。

 ![image-20211022182038497](/Users/mtdp/Library/Application Support/typora-user-images/image-20211022182038497.png

![image-20211022182118490](/Users/mtdp/Library/Application Support/typora-user-images/image-20211022182118490.png)

​																图 3 DialogFragment生命周期流程图

看下*Fragment*执行*onCreateView*的源码：

```java
					void moveToState(Fragment f, int newState, int transit, int transitionStyle,
                     boolean keepActive) {
       			......
            switch (f.mState) {
                case Fragment.INITIALIZING:
                    		......
                    // fall through
                case Fragment.CREATED:
                    ......

                    if (newState > Fragment.CREATED) {
                        if (DEBUG) Log.v(TAG, "moveto ACTIVITY_CREATED: " + f);
                        		......
                            f.mContainer = container;
                      			// ⑩
                            f.performCreateView(f.performGetLayoutInflater(
                                    f.mSavedFragmentState), container, f.mSavedFragmentState);
                           	......
                        }

                        f.performActivityCreated(f.mSavedFragmentState);
                        dispatchOnFragmentActivityCreated(f, f.mSavedFragmentState, false);
                        if (f.mView != null) {
                            f.restoreViewState(f.mSavedFragmentState);
                        }
                        f.mSavedFragmentState = null;
                    }
                    // fall through
                case Fragment.ACTIVITY_CREATED:
                    if (newState > Fragment.ACTIVITY_CREATED) {
                        if (DEBUG) Log.v(TAG, "moveto STARTED: " + f);
                        f.performStart();
                        dispatchOnFragmentStarted(f, false);
                    }
                    // fall through
                case Fragment.STARTED:
                    if (newState > Fragment.STARTED) {
                        if (DEBUG) Log.v(TAG, "moveto RESUMED: " + f);
                        f.performResume();
                        dispatchOnFragmentResumed(f, false);
                        f.mSavedFragmentState = null;
                        f.mSavedViewState = null;
                    }
            }
        }
```

⑩：在*performCreateView*时，需要传入*LayoutInflater*对象作为参数。*LayoutInflater*对象通过*performGetLayoutInflater*方法获取，查看方法*performGetLayoutInflater*。

```java
LayoutInflater performGetLayoutInflater(@Nullable Bundle savedInstanceState) {
        LayoutInflater layoutInflater = onGetLayoutInflater(savedInstanceState);
        mLayoutInflater = layoutInflater;
        return mLayoutInflater;
    }
```

实际是调用了*onGetLayoutInflater*方法，对于*DialogFragment*来说，它重写了该方法，并在其中调用了*onCreateDialog，*源码如下

```java
@Override
    @NonNull
    public LayoutInflater onGetLayoutInflater(@Nullable Bundle savedInstanceState) {
        if (!mShowsDialog) {
            return super.onGetLayoutInflater(savedInstanceState);
        }

        mDialog = onCreateDialog(savedInstanceState);

        if (mDialog != null) {
            setupDialog(mDialog, mStyle);

            return (LayoutInflater) mDialog.getContext().getSystemService(
                    Context.LAYOUT_INFLATER_SERVICE);
        }
        return (LayoutInflater) mHost.getContext().getSystemService(
                Context.LAYOUT_INFLATER_SERVICE);
    }
```

# 三、BottomSheetDialogFragment

*BottomSheetDialogFragment*继承自*DialogFragment*，也就是说它也具有*DialogFragment*的一些特性（或者更具体点说是*Fragment*的特性）。*BottomSheetDialogFragment*作为底部弹出式的视图，我们通常会将它和*BottomSheetBehavior*结合使用，来完成更复杂的交互或其他功能。

## 3.1 BottomSheetBehavior

*BottomSheetBehavior*使用还是比较简单，创建*BottomSheetBehavior*对象，和需要联动的View进行绑定即可。示例代码如下：

```java
...... 				
mBehavior = BottomSheetBehavior.from((View) view.getParent());
mBehavior.addBottomSheetCallback(bottomSheetCallback);
mBehavior.setPeekHeight(ScreenUtils.getScreenHeight(getContext()));
......
```

**注意**：*BottomSheetBehavior.from*方法要求传入的*View*的*LayoutParams*必须是*CoordinatorLayout*的*LayoutParams。*但这不意味着我们创建的*View*的根布局是*CoordinatorLayout*，它会自动给我们包一层，这个下面会详述。

通过*BottomSheetBehavior*，我们可以添加一些滑动和*Fling*的监听事件，或者设置*BottomSheetDialogFragment*打开时的初始高度等。

**问题：**

1）调用*BottomSheetBehavior.from*方法时，传入的是*view.getParent()*，因为这个view的根布局不是*CoordinatorLayout，如果传入view，*会报异常

```java
java.lang.IllegalArgumentException: The view is not a child of CoordinatorLayout
        at com.google.android.material.bottomsheet.BottomSheetBehavior.from(BottomSheetBehavior.java:1634)
```

\2) 如果传入*view.getParent()*（**也就是前面提到过的*****BottomSheetDialogFragment*****会给我们的*****view*****包一层*****CoordinatorLayout的*****布局**），无论是在*onCreateView*或*onViewCreated*生命周期方法中去创建*BottomSheetBehavior*对象，会报错并提示

```java
java.lang.NullPointerException: Attempt to invoke virtual method 'android.view.ViewGroup$LayoutParams android.view.View.getLayoutParams()' on a null object reference
        at com.google.android.material.bottomsheet.BottomSheetBehavior.from(BottomSheetBehavior.java:1632)
```

说明此时我们的*view.getParent()*还没创建好，所以出现空指针。那么就需要确定下view.*getParent()*创建的时机。

**排查：**

*BottomSheetDialogFragment*是*DialogFragment*的子类，那么会依次执行*DialogFragment*的生命周期方法，因此排查就是依次看下这些方法执行了什么逻辑。

- 根据图3，在*onCreateDialog*方法新建了***BottomSheetDialog***对象，并且创建了*Dialog*的*window*对象以及设置了*WindowManager*对象。

- *onCreateView*仅创建了*View*对象，*onViewCreated*是空方法，这2个方法看起来比较正常。

- *onActivityCreated*给*Dialog*设置了需要展示的*View*和一些监听事件，代码如下

  ```java
  		// DialogFragment.java
  
  		@Override
      public void onActivityCreated(@Nullable Bundle savedInstanceState) {
          super.onActivityCreated(savedInstanceState);
  
          if (!mShowsDialog) {
              return;
          }
  
          View view = getView();
          if (view != null) {
              if (view.getParent() != null) {
                  throw new IllegalStateException(
                          "DialogFragment can not be attached to a container view");
              }
            	// ①
              mDialog.setContentView(view);
          }
          final Activity activity = getActivity();
          if (activity != null) {
              mDialog.setOwnerActivity(activity);
          }
          mDialog.setCancelable(mCancelable);
          mDialog.setOnCancelListener(this);
          mDialog.setOnDismissListener(this);
          if (savedInstanceState != null) {
              Bundle dialogState = savedInstanceState.getBundle(SAVED_DIALOG_STATE_TAG);
              if (dialogState != null) {
                  mDialog.onRestoreInstanceState(dialogState);
              }
          }
      }
  ```

  ①：看下*Dialog*的*setContentView*方法，由于*BottomSheetDialogFragment*在*onCreateDialog*方法创建的是***BottomSheetDialog***对象，而***BottomSheetDialog***类有重写了*setContentView*方法，所以查看***BottomSheetDialog#setContentView***方法源码

  ```java
  	// BottomSheetDialog.java
  
   @Override
    public void setContentView(@LayoutRes int layoutResId) {
      super.setContentView(wrapInBottomSheet(layoutResId, null, null));
    }
  	
  	private View wrapInBottomSheet(
        int layoutResId, @Nullable View view, @Nullable ViewGroup.LayoutParams params) {
      ensureContainerAndBehavior();
      CoordinatorLayout coordinator = (CoordinatorLayout) container.findViewById(R.id.coordinator);
      if (layoutResId != 0 && view == null) {
        view = getLayoutInflater().inflate(layoutResId, coordinator, false);
      }
  
      FrameLayout bottomSheet = (FrameLayout) container.findViewById(R.id.design_bottom_sheet);
      bottomSheet.removeAllViews();
      // ②
      if (params == null) {
        bottomSheet.addView(view);
      } else {
        bottomSheet.addView(view, params);
      }
      // We treat the CoordinatorLayout as outside the dialog though it is technically inside
      coordinator
          .findViewById(R.id.touch_outside)
          .setOnClickListener(
              new View.OnClickListener() {
                @Override
                public void onClick(View view) {
                  if (cancelable && isShowing() && shouldWindowCloseOnTouchOutside()) {
                    cancel();
                  }
                }
              });
      // Handle accessibility events
      ViewCompat.setAccessibilityDelegate(
          bottomSheet,
          new AccessibilityDelegateCompat() {
            @Override
            public void onInitializeAccessibilityNodeInfo(
                View host, @NonNull AccessibilityNodeInfoCompat info) {
              super.onInitializeAccessibilityNodeInfo(host, info);
              if (cancelable) {
                info.addAction(AccessibilityNodeInfoCompat.ACTION_DISMISS);
                info.setDismissable(true);
              } else {
                info.setDismissable(false);
              }
            }
  
            @Override
            public boolean performAccessibilityAction(View host, int action, Bundle args) {
              if (action == AccessibilityNodeInfoCompat.ACTION_DISMISS && cancelable) {
                cancel();
                return true;
              }
              return super.performAccessibilityAction(host, action, args);
            }
          });
      bottomSheet.setOnTouchListener(
          new View.OnTouchListener() {
            @Override
            public boolean onTouch(View view, MotionEvent event) {
              // Consume the event and prevent it from falling through
              return true;
            }
          });
      return container;
    }
  ```

  ②：重点就在*wrapInBottomSheet*方法，虽然我们构造自己的*View*时没有使用*CoordinatorLayout*，但在该方法中，系统有个默认的布局文件*R.layout.design_bottom_sheet_dialog*（见下方代码）*，wrapInBottomSheet*方法就是将我们的布局添加到默认布局中(**id为design_bottom_sheet的FrameLayout**)*，*我们的布局的父视图就是*CoordinatorLayout*了，从而不必我们自己去设置*CoordinatorLayout*为根布局。

  ```xml
  // R.layout.design_bottom_sheet_dialog
  
  <?xml version="1.0" encoding="utf-8"?>
  <!--
    ~ Copyright (C) 2015 The Android Open Source Project
    ~
    ~ Licensed under the Apache License, Version 2.0 (the "License");
    ~ you may not use this file except in compliance with the License.
    ~ You may obtain a copy of the License at
    ~
    ~      http://www.apache.org/licenses/LICENSE-2.0
    ~
    ~ Unless required by applicable law or agreed to in writing, software
    ~ distributed under the License is distributed on an "AS IS" BASIS,
    ~ WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
    ~ See the License for the specific language governing permissions and
    ~ limitations under the License.
  -->
  <FrameLayout
      xmlns:android="http://schemas.android.com/apk/res/android"
      xmlns:app="http://schemas.android.com/apk/res-auto"
      xmlns:tools="http://schemas.android.com/tools"
      android:id="@+id/container"
      android:layout_width="match_parent"
      android:layout_height="match_parent"
      android:fitsSystemWindows="true">
  
      <android.support.design.widget.CoordinatorLayout
          android:id="@+id/coordinator"
          android:layout_width="match_parent"
          android:layout_height="match_parent"
          android:fitsSystemWindows="true">
  
          <View
              android:id="@+id/touch_outside"
              android:layout_width="match_parent"
              android:layout_height="match_parent"
              android:importantForAccessibility="no"
              android:soundEffectsEnabled="false"
              tools:ignore="UnusedAttribute"/>
  
          <FrameLayout
              android:id="@+id/design_bottom_sheet"
              style="?attr/bottomSheetStyle"
              android:layout_width="match_parent"
              android:layout_height="wrap_content"
              android:layout_gravity="center_horizontal|top"
              app:layout_behavior="@string/bottom_sheet_behavior"/>
  
      </android.support.design.widget.CoordinatorLayout>
  
  </FrameLayout>
  ```

**总结**

1. 创建*BottomSheetBehavior*对象时，**如果传入的View参数不是某个*****CoordinatorLayout*****的直接子View，则需要在onActivityCreated生命周期方法之后创建**（如*onActivityCreated*、*onStart*等）**。**
2. 如果传入的*View*参数不是某个*CoordinatorLayout*的直接子*View*，还需要在布局文件中给其设置*layout_behavior*属性，否则会报「he view is not associated with BottomSheetBehavior」异常。

