Transition 转场（过渡）动画效果

# 什么是Transition

> Transition是 Android的过渡框架，只需提供起始布局和结束布局，即可为界面中的各种运动添加动画效果。

可以用于同一页面各组件间的变化动效，也能用与Activity/Fragment切换时的过渡动画，以及共享元素的切换动画

# Transition使用

## Activity内的各个布局之间打造过渡动效

在两种布局之间添加动画效果的基本流程：

1. 为起始布局和结束布局创建 [`Scene`](https://developer.android.com/reference/android/transition/Scene?hl=zh-cn) 对象。
2. 创建一个 [`Transition`](https://developer.android.com/reference/android/transition/Transition?hl=zh-cn) 对象来定义所需的动画类型。
3. 调用 [`TransitionManager.go()`](https://developer.android.com/reference/android/transition/TransitionManager?hl=zh-cn#go(android.transition.Scene))，然后系统会运行动画以交换布局

![](https://raw.githubusercontent.com/Aaron-DBJ/ImageRepo/img/202504171511708.png)

**图 1.** 过渡框架如何创建动画

它的作用原理就是定义好变化的开始和结束场景(就是`Scene`)，然后结合`Transition`（Transition负责具体动画效果的视线）来实现动画过程。可以类比于属性动画，定义了一个动画的开始值和结束值，然后设置好插值器，就能实现动画效果。其实`Transition`内部也是通过属性动画来实现过渡动画效果的。

## 实现

`Transition`过渡动画的实现就是通过在需要变化的根布局中通过`addView`、`removeView`方式添加移除开始场景或结束场景，并结合属性动画来实现的。

例如我们调用`TransitionManager.go(scene);`来启动属性动画时，源码可以看出：

```java
public void enter() {

        // Apply layout change, if any
        if (mLayoutId > 0 || mLayout != null) {
            // empty out parent container before adding to it
            getSceneRoot().removeAllViews();

            if (mLayoutId > 0) {
                LayoutInflater.from(mContext).inflate(mLayoutId, mSceneRoot);
            } else {
                mSceneRoot.addView(mLayout);
            }
        }

        // Notify next scene that it is entering. Subclasses may override to configure scene.
        if (mEnterAction != null) {
            mEnterAction.run();
        }

        setCurrentScene(mSceneRoot, this);
    }
```

### 创建场景-Scene

##### 通过布局文件创建

场景创建通过 `Scene.getSceneForLayout()`方法实现，它需要传入3个参数：

- 需要实现动效的根布局sceneRoot

- 场景布局文件，就是开始或结束时的期望布局

- context

##### 通过代码创建

代码创建通过Scene的构造函数实现，需要传入根布局View和当前场景对应的View对象。

### 过渡动效

定义好了开始结束场景，接着就需要定义这2中场景的变化效果，就像属性动画中的插值器一样，需要定义2个属性值之间的变化过程，这就是`Transition`所做的工作。

#### 内置过渡动效

系统已经提前定义好了一些过渡动画，如下所示

| 类                    | XML标签                     | 作用                                                                                    |
| -------------------- | ------------------------- | ------------------------------------------------------------------------------------- |
| AutoTransition       | `<autoTransition/>`       | 默认过渡。按该顺序淡出视图、移动视图、调整大小以及淡入视图。                                                        |
| ChangeBounds         | `<changeBounds/>`         | 移动视图并调整其大小。                                                                           |
| ChangeClipBounds     | `<changeClipBounds/>`     | 捕获场景变化前后的 `View.getClipBounds()`，并在过渡期间为这些更改添加动画效果。                                   |
| ChangeImageTransform | `<changeImageTransform/>` | 捕获场景变化前后的 `ImageView` 矩阵，并在过渡期间为其添加动画效果。                                              |
| ChangeScroll         | `<changeScroll/>`         | 捕获场景更改前后目标的滚动属性，并为任何更改添加动画效果。                                                         |
| ChangeTransform      | `<changeTransform/>`      | 捕获场景更改前后视图的缩放、移动和旋转，并在过渡期间为这些更改添加动画效果。                                                |
| Explode              | `<explode/>`              | 跟踪起始场景和结束场景中目标视图的可见性变化，并将视图从场景边缘移入或移出。                                                |
| Fade                 | `<fade/>`                 | `fade_in` 淡入视图。<br>`fade_out` 淡出视图。<br>`fade_in_out`（默认）先执行 `fade_out` 后执行 `fade_in`。 |
| Slide                | `<slide/>`                | 跟踪起始场景和结束场景中目标视图的可见性变化，并将视图从场景边缘移入或移出。                                                |

📢**注意：不同动画必须满足条件才会生效，例如`ChangeTransform`一定是有`translation`、`rotate`或`scale`属性变化才有动画效果，在使用时需要注意。**

#### 从资源文件创建Transition

如需在资源文件中指定内置过渡，请按以下步骤操作：

- 将 `res/transition/` 目录添加到项目中。
- 在此目录中新建一个 XML 资源文件。
- 为其中一个内置过渡添加 XML 节点。

例如，以下资源文件指定了 `Fade` 过渡：

res/transition/fade_transition.xml

```xml
<fade xmlns:android="http://schemas.android.com/apk/res/android" />
```

从资源文件创建Transition实例，通过`TransitionInflater`类创建

```java
Transition fadeTransition =
        TransitionInflater.from(this).
        inflateTransition(R.transition.fade_transition);
```

如果要实现复杂动画效果，也可以将过渡动效进行组合，类似于集合动画一样。使用 `</transitionSet>`实现，例如：

```xml
<transitionSet xmlns:android="http://schemas.android.com/apk/res/android"
    android:transitionOrdering="sequential">
    <fade android:fadingMode="fade_out" />
    <changeBounds />
    <fade android:fadingMode="fade_in" />
</transitionSet>
```

#### 在代码中创建Transition

通过构造方法创建即可，例如

```java
Transition fadeTransition = new Fade();
```

### 启动过渡动画

使用`TransitionManager.go(sceneRoot)`或`TransitionManager.go(sceneRoot, transition)`启动

### beginDelayedTransition

使用scene时，需要提前定义好开始状态和结束状态时的布局文件，使用比较麻烦，并且有时仅需部分view具有动画效果，这个时候可以使用`beginDelayedTransition`方法。

`beginDelayedTransition`类似于先对即将发生变化的父布局进行注册，然后改变父布局内的View的尺寸、平移、旋转等属性，就会把注册的Transition效果应用到该View上。这一点从本方法的名字中的delayed也可见一点端倪。

使用示例：

```java
private void changeSize() {
        TransitionManager.beginDelayedTransition(constraintLayout);
        ConstraintLayout.LayoutParams layoutParams = (ConstraintLayout.LayoutParams) smallCircle.getLayoutParams();
        layoutParams.width *= 2;
        layoutParams.height *= 2;
        smallCircle.setLayoutParams(layoutParams);
    }
```

## Activity/Fragment过渡动画

> Activity 过渡 API 适用于 Android 5.0 (API 21) 及更高版本

**实现步骤**：

- 设置`Transition`，支持通过XML或者代码设置

- 使用`ActivityOptions`启动Activity

### 创建Transition

#### 从资源文件中创建Transition

前文已说明，不在赘述

#### 在代码中创建Transition

首先调用 `Window.requestFeature()`开启过渡动画

```java
getWindow().requestFeature(Window.FEATURE_ACTIVITY_TRANSITIONS);
```

然后设置过渡动画

有4种设置方式，比如A页面 打开 B页面，这4种方法分别对应：

`setEnterTransition`：B页面被打开时生效

`setReturnTransition`：B页面退出时生效

`setExitTransition`：A打开B，A退出时生效

`setReenterTransition`：A重新被打开（B关闭）时生效

### ActivityOptions方式启动Activity

```java
Intent intent = new Intent(xxx, yyy);
startActivity(intent, ActivityOptions.makeSceneTransitionAnimation(this).toBundle());
```

### 指定或移除页面过渡动画生效的布局内容

在过渡动画变化的过程中，默认是全部布局都会有动画效果，包括状态栏等系统组件，通常我们不想要这些系统组件也有动画效果时，或者只想指定某些布局有过渡效果。这个时候就可以通过`Target`来实现。

同样，有2种实现方式：

- XML文件：通过`<targets>`和`<Target>`标签的`android:targetId` 和`android:excludeId`属性来指定。前者表示只有targetId声明的组件会有过渡动画；后者表示这些组件不展示过渡动画。

- 代码方式：调用 `addTarget`或`removeTarget`

```java
<?xml version="1.0" encoding="utf-8"?>
<explode xmlns:android="http://schemas.android.com/apk/res/android"
    android:duration="500"
    android:interpolator="@android:interpolator/accelerate_decelerate">
    <targets>
        <target android:excludeId="@android:id/statusBarBackground" />
    </targets>
</explode>
```

## 共享元素过渡动画

共享元素动画是指可以在2个页面切换时，针对部分View控件设置一个相同的标志，这个标志叫做transition name，并且设置过渡动画效果。那么在页面切换时被标记为共享元素的View控件从A页面到B页面时会有一个过渡动画的效果。

共享元素需要成对设置，很好理解，动画必须有开始状态和结束状态，只需对需要有动画效果的view设置相同的transition name即可。

共享元素可以设置多对，只要保证每对之间的transition name相同即可。

### 实现方式

#### 标记共享元素View

有2种方式进行标记：

1. 在XML文件中设置 `android:transitionName="xxx"`

2. 代码中通过`setTransitionName`设置

#### 启动共享元素动画

还是通过`ActivityOptions`方式来启动页面，启动方式：

1、一个共享元素

```java
 public static ActivityOptions makeSceneTransitionAnimation(Activity activity,
            View sharedElement, String sharedElementName) {
        return makeSceneTransitionAnimation(activity, Pair.create(sharedElement, sharedElementName));
    }
```

2、多个共享元素

```java
 @SafeVarargs
    public static ActivityOptions makeSceneTransitionAnimation(Activity activity,
            Pair<View, String>... sharedElements) {
        ActivityOptions opts = new ActivityOptions();
        makeSceneTransitionAnimation(activity, activity.getWindow(), opts,
                activity.mExitTransitionListener, sharedElements);
        return opts;
    }
```
