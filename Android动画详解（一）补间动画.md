# Android动画

## 一、概述

Android中常用到的动画有三种，分别是：帧动画、补间动画和属性动画。

1、帧(Frame)动画

​	帧动画的原理和电影一样，就是把一系列静态图片按一定顺序播放，利用人眼的视觉暂留效应使之呈现				动态效果。

2、补间(Tween)动画

​	补间动画是利用视图的平移、旋转、缩放和渐变来实现动画效果。

3、属性(Property)动画

​	虽然帧动画和补间动画可以实现很多动画效果，但是他们也有很大的局限性。比如它们只能在整个控件上去实现某个动画，而不能改变其属性。如果想让一个按钮(Button)的背景色由蓝变红，帧动画和补间动画就无能为力了。而且，也因为前两种动画不能改变属性，它们的动画只是让控件的显示位置改变，并不会改变控件本身的值。例如，一个按钮(Button)从位置A平移到位置B，点击位置B处的按钮没有任何响应，而点击位置A才会有反应。所以为了解决这种问题，属性动画应运而生。

## 二、补间动画

Animation有两种使用方式，一是xml资源文件方式来实现动画；二是完全由Java代码实现。

1、xml资源文件实现

​	在res/anim文件夹下创建xml资源文件，使用<alpha>、 <translate>、<rotate>和<scale>标签即可定义相应的动画效果。

| 属性                                                         | 功能                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| [`android:duration`](https://developer.android.google.cn/reference/android/view/animation/Animation.html#attr_android:duration) | 动画的持续时间，单位为毫秒                                   |
| [`android:fillAfter`](https://developer.android.google.cn/reference/android/view/animation/Animation.html#attr_android:fillAfter) | 是否保留动画结束时的状态，如果为true，则保留；反之则变回动画开始时的状态。 |
| [`android:fillBefore`](https://developer.android.google.cn/reference/android/view/animation/Animation.html#attr_android:fillBefore) | 如果为ture，则回到动画开始状态。                             |
| [`android:fillEnabled`](https://developer.android.google.cn/reference/android/view/animation/Animation.html#attr_android:fillEnabled) | 如果为ture，则回到动画开始状态。                             |
| [`android:interpolator`](https://developer.android.google.cn/reference/android/view/animation/Animation.html#attr_android:interpolator) | 设置插值器                                                   |
| [`android:repeatCount`](https://developer.android.google.cn/reference/android/view/animation/Animation.html#attr_android:repeatCount) | 设置动画重复次数                                             |
| [`android:repeatMode`](https://developer.android.google.cn/reference/android/view/animation/Animation.html#attr_android:repeatMode) | 设置动画的重复类型                                           |

（1）渐变动画

```xml
<alpha xmlns:android="http://schemas.android.com/apk/res/android"
    android:fromAlpha="0"
    android:toAlpha="1"
    android:duration="1500">
</alpha>
```

![渐变动画](C:\Users\jenny\Pictures\BlogPictures\alpha.gif)

（2）平移动画

```xml
<translate xmlns:android="http://schemas.android.com/apk/res/android"
    android:fromXDelta="0"
    android:toXDelta="200"
    android:fromYDelta="0"
    android:toYDelta="200"
    android:duration="1500">
</translate>
```

![渐变动画](C:\Users\jenny\Pictures\BlogPictures\translate.gif)



（3）旋转动画

​	旋转动画还常用到pivotX和pivotY属性，表示旋转起点的x坐标和y坐标，取值可以为数值、百分数和百分数p着三种。比如20，20%和20%p。如果是数值，如20，表示在原点（view的左上角）处加20px；如果是20%，则表示在当前view左上角加上自身宽度的20%；若是20%p，表示在当前view左上角加上父控件宽度的20%。

```xml
<rotate xmlns:android="http://schemas.android.com/apk/res/android"
    android:fromDegrees="0"
    android:toDegrees="500"
    android:duration="1500"
    android:pivotX="50%"
    android:pivotY="50%"
    >
</rotate>
```

![渐变动画](C:\Users\jenny\Pictures\BlogPictures\rotate.gif)

上图是将pivotX和pivotY设置为50%。即控件中心点为缩放的起始位置。如果设为其他值看看是什么效果，下图是设置为20%的效果。

![渐变动画](C:\Users\jenny\Pictures\BlogPictures\pivot20%.gif)

不设置fillbefore和fillafter属性默认情况是动画结束后控件就会回到初始状态，如果设置了fillAfter为true，那么就会保留动画结束时的状态，如下图：

![android:fillAfter="true"](C:\Users\jenny\Pictures\BlogPictures\fillAfter.gif)

（4）缩放动画

缩放动画的pivotX和pivotY表示的含义和旋转动画相似，代表缩放起点的x坐标和y坐标的值。

```xml
<scale xmlns:android="http://schemas.android.com/apk/res/android"
    android:fromXScale="0"
    android:fromYScale="0"
    android:toXScale="1"
    android:toYScale="1"
    android:pivotY="50%"
    android:pivotX="50%"
    android:duration="1500"
    >
</scale>
```

![渐变动画](C:\Users\jenny\Pictures\BlogPictures\scale.gif)



定义好了xml资源文件后，在Android代码中定义Animation对象，使用AnimationUtils.loadAnimation方法将xml文件加载到Animation对象，然后控件调用startAnimation方法来实现动画效果。

```java
ImageView imageView;
...
Animation animation = AnimationUtils.loadAnimation(this, R.anim.animation);
imageView.startAnimation(animation);
...
```

2、Java代码实现

补间动画对应的Java类是Animation，它有几个子类AlphaAnimation、RotateAnimation、ScaleAnimation和TranslateAnimation，分别对应渐变、旋转、缩放和平移动画。

Animation的常用方法：

| Public methods |                                                              |                       |
| -------------- | ------------------------------------------------------------ | --------------------- |
| `void`         | `cancel()`                                                   | 取消动画              |
| `void`         | `reset()`                                                    | 重置动画              |
| `int`          | `getBackgroundColor()`                                       | 返回动画的背景颜色    |
| `boolean`      | `isFillEnabled()`                                            | fillEnabled是否为true |
| `long`         | `getDuration()`                                              | 获取动画持续时间      |
| `boolean`      | `getFillAfter()`                                             | fillAfter 是否为true  |
| `void`         | `setRepeatMode(int repeatMode)`                              | 设置动画重复类型      |
| `Interpolator` | `getInterpolator()`                                          | 获取动画的插值器      |
| `int`          | `getRepeatCount()`                                           | 获取动画的重复次数    |
| `int`          | `getRepeatMode()`                                            | 获取动画的重复类型    |
| `void`         | `setAnimationListener(Animation.AnimationListener listener)` | 设置动画监听器        |
| `void`         | `setBackgroundColor(int bg)`                                 | 设置动画背景颜色      |
| `void`         | `start()`                                                    | 开始动画              |
| `void`         | `setRepeatCount(int repeatCount)`                            | 设置动画重复次数      |
| `void`         | `setDuration(long durationMillis)`                           | 设置动画持续时间      |
| `void`         | `setFillAfter(boolean fillAfter)`                            | 设置fillAfter属性     |
| `void`         | `setFillBefore(boolean fillBefore)`                          | 设置fillBefore属性    |
| `void`         | `setFillEnabled(boolean fillEnabled)`                        | 设置fillEnabled属性   |
| `void`         | `setInterpolator(Context context, int resID)`                | 设置插值器            |
| `void`         | `setInterpolator(Interpolator i)`                            | 设置插值器            |

（1）AlphaAnimation

构造方法有：

| Public constructors                                 |
| :-------------------------------------------------- |
| AlphaAnimation(Context context, AttributeSet attrs) |
| AlphaAnimation(float fromAlpha, float toAlpha)      |

使用时：

```Java
ImageView imageView;
...
Animation animation = new AlphaAniamtion(0, 1);
animation.setDuration(2000);
animation.setfillAfter(true);
aniamtion.setInterpolator(new LinearInterpolator());
imageView.startAnimation(animation);
...
```

这里设置了插值器，什么是插值器先不管它，后面再讲。

这里呈现了一个渐变动画的Java代码实现过程，平移、旋转和缩放动画也极其类似。

（2）RotateAnimation

| 构造方法                                                     |
| ------------------------------------------------------------ |
| `RotateAnimation(float fromDegrees, float toDegrees)`        |
| `RotateAnimation(float fromDegrees, float toDegrees, float pivotX, float pivotY)` |
| `RotateAnimation(float fromDegrees, float toDegrees, int pivotXType, float pivotXValue,int pivotYType, float pivotYValue)` |
| `RotateAnimation(Context context, AttributeSet attrs)`       |

虽说着四种动画的实现方式及其类似，但还是有些许不同。不同之处在上面的构造函数中就出来了，旋转动画中多了几个参数:

- *pivotXType*
- *pivotXValue*
- *pivotYType*
- *pivotYValue*

其实这几个参数也不是新东西，前面讲过变化的起点坐标pivotX和pivotY，它们各有三种取值，数值，百分数和百分数p。所以pivotXType和pivotYType就代表这3中取值，在Java代码中，这3中类型分别是：Animation.ABSOLUTE、Animation.RELATIVE_TO_SELF和Animation.RELATIVE_TO_PARENT，这里你就知道百分数p的这个「p」代表parent了。

使用时：

```Java
ImageView imageView;
...
Animation animation = new RotateAnimation(0, 500, Animation.RELATIVE_TO_SELF, 0.5f,
                 Animation.RELATIVE_TO_SELF, 0.5f);
animation.setDuration(2000);
animation.setfillAfter(true);
aniamtion.setInterpolator(new LinearInterpolator());
imageView.startAnimation(animation);
...
```

（3）ScaleAnimation

| 构造函数                                                     |
| :----------------------------------------------------------- |
| `ScaleAnimation(float fromX, float toX, float fromY, float toY)` |
| `ScaleAnimation(float fromX, float toX, float fromY, float toY, float pivotX, float pivotY)` |
| `ScaleAnimation(float fromX, float toX, float fromY, float toY, int pivotXType, float pivotXValue, int pivotYType, float pivotYValue)` |
| `ScaleAnimation(Context context, AttributeSet attrs)`        |

缩放动画和旋转动画的构造函数都有设置动画变化起点的参数，方法也一样就不赘述了。

（4）TranslateAnimation

| Public constructors                                          |
| ------------------------------------------------------------ |
| `TranslateAnimation(Context context, AttributeSet attrs)`    |
| `TranslateAnimation(float fromXDelta, float toXDelta, float fromYDelta, float toYDelta)` |
| `TranslateAnimation(int fromXType, float fromXValue, int toXType, float toXValue, int fromYType, float fromYValue, int toYType, float toYValue)` |

使用方法和前3者都一样，不废话了。



补间动画就先讲到这里，由于本人知识水平有限，如有错误和疏漏之处还望指正。

## 示例代码

[示例代码](https://github.com/Aaron-DBJ/MyView)

## 参考资料

[自定义控件三部曲之动画篇（三）—— 代码生成alpha、scale、translate、rotate、set及插值器动画](https://blog.csdn.net/harvic880925/article/details/40117115)