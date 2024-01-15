# 一、引言

内存泄露一直是Android开发中需要避免的问题，因此发现和定位内存泄露就是我们治理内存泄露问题的首要动作。目前市面上最流行的内存泄露排查组件非大名鼎鼎的***LeakCanary***莫属了，它能非常方便直观把内存泄漏处的引用链展示出来，有助于我们的快速定位。另外，其使用方式也十分简单友好。对于开发者而言，仅满足使用还是不够的，尽量知其所以然，于是就有了这篇源码的解析。

# 二、使用和原理

> 源码分析基于LeakCanary版本1.6.3

## 2.1 使用方式

LeakCanary可以检测具体Activity和Fragment是否存在内存泄露，也可以对想要观察的对象进行检测。它的使用十分简单，以检测Activity为例，以下几行代码即可。

```java
public class MyApplication extends Application {
    @Override
    public void onCreate() {
        super.onCreate();
        if (!LeakCanary.isInAnalyzerProcess(this)) {
            LeakCanary.install(this);
        }
    }
}
```

自定义Application类，调用`LeakCanary.install()`方法即可。

## 2.2 检测原理

*LeakCanary*的检测原理是：

**弱引用对象在其包装对象被回收后（弱引用对象创建时，传入了引用队列），该弱引用对象会被加到引用队列中（*ReferenceQueue*）。**

**通过在*ReferenceQueue*中检测是否有目标对象的弱引用对象存在，即可判断目标对象是否被回收。**

例如：

```java
......
Activity mActivity;
......
ReferenceQueue<Activity> mQueue = new ReferenceQueue<>();
WeakReference<Activity> mWeakReference = new WeakReference<>(mActivity, mQueue);
......
```

在创建目标对象`mActivity`的弱引用对象时，如果构造方法中传入了引用队列`mQueue`，那么当`mActivity`对象被回收时，`mWeakReference`对象将会被添加到引用队列`mQueue`中。

# 三、源码分析

了解了检测原理后，现在以检测*Activity*是否存在内存泄露为例，来详细分析下实现源码。

## 3.1、使用入口`install`方法

```java
public static @NonNull RefWatcher install(@NonNull Application application) {
    return refWatcher(application).listenerServiceClass(DisplayLeakService.class)
        .excludedRefs(AndroidExcludedRefs.createAppDefaults().build())
        .buildAndInstall();
  }
```

进行一些初始化的构造，重点在`buildAndInstall`方法

## 3.2、`buildAndInstall`方法

```java
public @NonNull RefWatcher buildAndInstall() {
    if (LeakCanaryInternals.installedRefWatcher != null) {
      throw new UnsupportedOperationException("buildAndInstall() should only be called once.");
    }
    // ①
    RefWatcher refWatcher = build();
    // ②
    if (refWatcher != DISABLED) {
      if (enableDisplayLeakActivity) {
        LeakCanaryInternals.setEnabledAsync(context, DisplayLeakActivity.class, true);
      }
      if (watchActivities) {
        ActivityRefWatcher.install(context, refWatcher);
      }
      if (watchFragments) {
        FragmentRefWatcher.Helper.install(context, refWatcher);
      }
    }
    // ③
    LeakCanaryInternals.installedRefWatcher = refWatcher;
    return refWatcher;
  }
```

① 调用`build`方法创建*RefWatcher*对象，**注意这里使用的是*AndroidRefWatcherBuilder***，是在`install`方法中调用`refWatcher`方法返回的。

② 这里主要进行一些自定义的设置，比如是否展示内存泄露的链路展示页面、是否检测*Activity*的内存泄露和是否检测*Fragment*的内存泄露。默认为否-是-是。所以会调用`ActivityRefWatcher.install()`方法和`FragmentRefWatcher.Helper.install`方法。

**这2个方法就是去设置检测Activity/*Fragment*对象是否泄露的时机**。

以`ActivityRefWatcher.install()`方法为例

```java
// ActivityRefWatcher.class
......
public static void install(@NonNull Context context, @NonNull RefWatcher refWatcher) {
    Application application = (Application) context.getApplicationContext();
    ActivityRefWatcher activityRefWatcher = new ActivityRefWatcher(application, refWatcher);
	// ②.1
    application.registerActivityLifecycleCallbacks(activityRefWatcher.lifecycleCallbacks);
  }

  private final Application.ActivityLifecycleCallbacks lifecycleCallbacks =
      new ActivityLifecycleCallbacksAdapter() {
        @Override public void onActivityDestroyed(Activity activity) {
            // ②.2
          refWatcher.watch(activity);
        }
      };
......
```

注册*Activity*的生命周期回调，***LeakCanary*注册了在`onDestroy`生命周期方法执行的时候，去检测当前*Activity*对象是否存在内存泄露**。

对于*Fragment*逻辑类似，是在`onFragmentDestroyed`和`onFragmentViewDestroyed`时检测

③ 将创建的*RefWatcher*对象赋值给*LeakCanaryInternals.installedRefWatcher*对象

## 3.3、检测逻辑入口`refWatcher.watch()`

代码如下

```java
// RefWatcher.class
......
public void watch(Object watchedReference) {
    ① 
    watch(watchedReference, "");
  }

public void watch(Object watchedReference, String referenceName) {
    if (this == DISABLED) {
      return;
    }
    checkNotNull(watchedReference, "watchedReference");
    checkNotNull(referenceName, "referenceName");
    final long watchStartNanoTime = System.nanoTime();
    // ②
    String key = UUID.randomUUID().toString();
    retainedKeys.add(key);
    // ③
    final KeyedWeakReference reference =
        new KeyedWeakReference(watchedReference, key, referenceName, queue);
	// ④
    ensureGoneAsync(watchStartNanoTime, reference);
  }
```

① 对于*Activity*和*Fragment*来说，传入的*referenceName*为空串

② 每次检测时，生成一个随机数并添加到集合中，这个随机数会被当做*KeyedWeakReference*对象的一个属性，所以可以指代被检测的*Activity*/*Fragment*对象。

③ 然后创建*KeyedWeakReference*对象，这里传入了引用队列。所以当被观测的*Activity*或*Fragment*被回收时，对应的引用对象将会被添加到引用队列中。

④ 开始内存泄露检测的具体逻辑

```java
// RefWatcher.class
......
private void ensureGoneAsync(final long watchStartNanoTime, final KeyedWeakReference reference) {
    // ⑤
    watchExecutor.execute(new Retryable() {
      @Override public Retryable.Result run() {
        return ensureGone(reference, watchStartNanoTime);
      }
    });
  }
```

⑤ *watchExecutor*是一个函数式接口，只有一个`execute`方法，看它的实现类*AndroidWatchExecutor*，调用

`execute`方法

```java
// AndroidWatchExecutor.class
@Override public void execute(@NonNull Retryable retryable) {
    if (Looper.getMainLooper().getThread() == Thread.currentThread()) {
      waitForIdle(retryable, 0);
    } else {
      postWaitForIdle(retryable, 0);
    }
  }

private void postWaitForIdle(final Retryable retryable, final int failedAttempts) {
    mainHandler.post(new Runnable() {
      @Override public void run() {
        waitForIdle(retryable, failedAttempts);
      }
    });
  }

  private void waitForIdle(final Retryable retryable, final int failedAttempts) {
    // This needs to be called from the main thread.
    Looper.myQueue().addIdleHandler(new MessageQueue.IdleHandler() {
      @Override public boolean queueIdle() {
        postToBackgroundWithDelay(retryable, failedAttempts);
        return false;
      }
    });
  }

private void postToBackgroundWithDelay(final Retryable retryable, final int failedAttempts) {
    long exponentialBackoffFactor = (long) Math.min(Math.pow(2, failedAttempts), maxBackoffFactor);
    long delayMillis = initialDelayMillis * exponentialBackoffFactor;
    // ⑥
    backgroundHandler.postDelayed(new Runnable() {
      @Override public void run() {
        Retryable.Result result = retryable.run();
        if (result == RETRY) {
          postWaitForIdle(retryable, failedAttempts + 1);
        }
      }
    }, delayMillis);
  }
.......
```

如果在主线程，就调用`waitForIdle`方法，否则调用`postWaitForIdle`，最终都是调用`waitForIdle`方法。

使用*IdleHandler*在消息队列空闲的时候执行任务`postToBackgroundWithDelay`，且只执行一次。

⑥ 延迟5s子线程中执行*runnable*，会执行到`ensureGone`方法

## 3.4 泄露判定ensureGone方法

`ensureGone`方法主要是进行是否真的存在内存泄露的判定逻辑执行，通过2次判定来最终确定是否有内存泄露的问题。

```java
// RefWatcher.class
......
Retryable.Result ensureGone(final KeyedWeakReference reference, final long watchStartNanoTime) {
    long gcStartNanoTime = System.nanoTime();
    long watchDurationMs = NANOSECONDS.toMillis(gcStartNanoTime - watchStartNanoTime);
	// ①
    removeWeaklyReachableReferences();

    if (debuggerControl.isDebuggerAttached()) {
      // The debugger can create false leaks.
      return RETRY;
    }
    // ②
    if (gone(reference)) {
      return DONE;
    }
    // ③
    gcTrigger.runGc();
    // ④
    removeWeaklyReachableReferences();
    // ⑤
    if (!gone(reference)) {
      long startDumpHeap = System.nanoTime();
      long gcDurationMs = NANOSECONDS.toMillis(startDumpHeap - gcStartNanoTime);

      File heapDumpFile = heapDumper.dumpHeap();
      if (heapDumpFile == RETRY_LATER) {
        // Could not dump the heap.
        return RETRY;
      }
      long heapDumpDurationMs = NANOSECONDS.toMillis(System.nanoTime() - startDumpHeap);

      HeapDump heapDump = heapDumpBuilder.heapDumpFile(heapDumpFile).referenceKey(reference.key)
          .referenceName(reference.name)
          .watchDurationMs(watchDurationMs)
          .gcDurationMs(gcDurationMs)
          .heapDumpDurationMs(heapDumpDurationMs)
          .build();

      heapdumpListener.analyze(heapDump);
    }
    return DONE;
  }

private boolean gone(KeyedWeakReference reference) {
    return !retainedKeys.contains(reference.key);
  }

private void removeWeaklyReachableReferences() {
    // WeakReferences are enqueued as soon as the object to which they point to becomes weakly
    // reachable. This is before finalization or garbage collection has actually happened.
    KeyedWeakReference ref;
    while ((ref = (KeyedWeakReference) queue.poll()) != null) {
      retainedKeys.remove(ref.key);
    }
  }
```

① 从引用队列中取出对象，如果不为空，就删除集合保存的隐射该对象的随机数。如果对象被回收了，就会在引用队列中存在，那么就会删除集合中的随机数，也就意味着没有内存泄露。

② 判断*retainedKeys*集合中是否还有该随机数，如果没有了，说明引用队列有该对象，该对象被回收了，没有内存泄露。

③ 如果*retainedKeys*集合还有该随机数，意味着引用队列中没有该对象，该对象没能被回收，于是手动触发一次*GC*。

④ *GC*后，再次移除*retainedKeys*集合中指代被检测对象的随机数。如果*GC*后被观测对象可以回收，那么引用队列就会存在，并且能够移除*retainedKeys*集合中的*key*。

⑤ *retainedKeys*集合还有该随机数，说明对象不能被回收，发生了内存泄露。就开始导出堆文件，进行分析，展示泄露引用链路页面等。

## 3.5 其他

*LeakCanary*默认是检测*Activity*/*Fragment*的内存泄露，其实根据上文所述，检测内存泄露核心方法就是`refWatcher.watch`方法，所以只要创建了*RefWatcher*对象，理论上可以检测任意对象的内存泄露问题。

# 4、总结

本文介绍了*LeakCanary*的使用方法和检测原理。核心原理是：**利用被回收对象，在用弱引用包装时传入的引用队列，弱引用对象会在被包装对象被回收时放入引用队列中，通过检测其在引用队列的存在与否，来判定对象是否被回收。**

另外，对其源码进行了分析，剖析了内存检测的时机，检测的具体逻辑等。