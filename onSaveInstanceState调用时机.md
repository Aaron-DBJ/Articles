## onSaveInstanceState调用时机

**结论1：当Activity没有finish生命周期，且当前版本在安卓3.0之前的，OnSaveInstanceState方法在onPause之前调用**

**结论2：安卓3.0之后至9.0之前，OnSaveInstanceState方法在onPause之后onStop之前调用**

**结论3：安卓9.0之后OnSaveInstanceState方法在onStop之后调用**

## 执行条件

源码里有3处执行时机，分别是

1.onPause之前调用

```java
  android.app.ActivityThread
  private Bundle performPauseActivity(ActivityClientRecord r, boolean finished, String reason,
            PendingTransactionActions pendingActions) {
            ......
            // Pre-Honeycomb apps always save their state before pausing
            final boolean shouldSaveState = !r.activity.mFinished && r.isPreHoneycomb();
            if (shouldSaveState) {
                关键代码
                callActivityOnSaveInstanceState(r);
            }
            内部通过Instrumentation,调用Acivity的performPause
            方法，进而执行onPause
            performPauseActivityIfNeeded(r, reason);
            ......
   }



    private boolean isPreHoneycomb() {
            return activity != null 
                      && activity.getApplicationInfo().targetSdkVersion < android.os.Build.VERSION_CODES.HONEYCOMB;
        }

```

2.onPause之后和onStop之前调用

```java
android.app.ActivityThread
    private void callActivityOnStop(ActivityClientRecord r, boolean saveState, String reason) {
        // Before P onSaveInstanceState was called before onStop, starting with P it's
        // called after. Before Honeycomb state was always saved before onPause.
        final boolean shouldSaveState = saveState && !r.activity.mFinished && r.state == null
                && !r.isPreHoneycomb();
        final boolean isPreP = r.isPreP();
        if (shouldSaveState && isPreP) {
            // 第二处
            callActivityOnSaveInstanceState(r);
        }

        try {
             执行onStop
            r.activity.performStop(r.mPreserveWindow, reason);
        } catch (SuperNotCalledException e) {
            throw e;
        } catch (Exception e) {
            if (!mInstrumentation.onException(r.activity, e)) {
                throw new RuntimeException(
                        "Unable to stop activity "
                                + r.intent.getComponent().toShortString()
                                + ": " + e.toString(), e);
            }
        }
        r.setState(ON_STOP);

        if (shouldSaveState && !isPreP) {
            // 第三处
            callActivityOnSaveInstanceState(r);
        }
    }
```

3.onStop之后调用

```java
android.app.ActivityThread
    private void callActivityOnStop(ActivityClientRecord r, boolean saveState, String reason) {
        // Before P onSaveInstanceState was called before onStop, starting with P it's
        // called after. Before Honeycomb state was always saved before onPause.
        final boolean shouldSaveState = saveState && !r.activity.mFinished && r.state == null
                && !r.isPreHoneycomb();
        final boolean isPreP = r.isPreP();
        if (shouldSaveState && isPreP) {
            // 第二处
            callActivityOnSaveInstanceState(r);
        }

        try {
             执行onStop
            r.activity.performStop(r.mPreserveWindow, reason);
        } catch (SuperNotCalledException e) {
            throw e;
        } catch (Exception e) {
            if (!mInstrumentation.onException(r.activity, e)) {
                throw new RuntimeException(
                        "Unable to stop activity "
                                + r.intent.getComponent().toShortString()
                                + ": " + e.toString(), e);
            }
        }
        r.setState(ON_STOP);

        if (shouldSaveState && !isPreP) {
            // 第三处
            callActivityOnSaveInstanceState(r);
        }
    }
```

可以间这3处调用都有一个非常重要的变量`shouldSaveState`，如果他为false，则无论什么版本都不会执行**OnSaveInstanceState**方法。我们都知道，只有**未经你许可关闭**Activity的时候，我们才会需要保存数据，并不是每次都会执行**OnSaveInstanceState**方法的。那可想而知，**shouldSaveState**就是什么叫**未经你许可**的体现了

而`shouldSaveState`又由一个关键变量`saveState`决定，它就是表明什么是未经用户许可的情况，它赋值为true的地方非常多，大概有下列场景

- 当用户按下HOME键时 

-  长按HOME键，选择运行其他的程序时 

- 按下电源按键（关闭屏幕显示）时 

- 从activity A中启动一个新的activity时 

- 屏幕方向切换 

总而言之，onSaveInstanceState的调用遵循一个重要原则，即当系统“未经你许可”时销毁了你的activity，则onSaveInstanceState会被系统调用，这是系统的责任，因为它必须要提供一个机会让你保存你的数据。

## onRestoreInstanceState什么时候执行

执行**OnRestoreInstanceState**方法是在Activity的**onStart**之后。



## 参考材料

[Android 11源码分析：onSaveInstanceState到底做了什么？你知道的调用时机真的是正确的吗？ - 掘金](https://juejin.cn/post/6995791487426363405)
