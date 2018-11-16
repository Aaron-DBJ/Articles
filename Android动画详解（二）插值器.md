# Android动画详解（二）插值器

在上一篇[Android动画详解（一）补间动画](https://blog.csdn.net/qq_21830869/article/details/84073764)中我们提到过一个叫插值器的东西，看名字一头雾水完全不知道是什么神奇玩意。其实用人话翻译过来就是速度模型或者速度曲线的意思。为动画设置插值器就是设定动画的速度模型，就是设置它是怎么动的，先加速再加速呀、一直减速呀、匀速的运动啊。插值器不只是补间动画需要设置啊，后面要讲的属性动画一样有插值器。具体效果一看动图便知。

## 1、AccelerateDecelerateInterpolator

先加速运动再减速知道终点处，这也是默认的Interpolator，如果不setInterpolator(interpolator)，动画以该方式运动。

![先加速再减速](C:\Users\jenny\Pictures\BlogPictures\accelaratedecelarate.gif)

## 2、AccelerateInterpolator

一直加速前进，在终点处骤停。

![一直加速](C:\Users\jenny\Pictures\BlogPictures\accelerate.gif)

## 3、AnticipateInterpolator

先往回拉一小段距离，在先前运动。

![先往后拉一段距离再前进](C:\Users\jenny\Pictures\BlogPictures\anticipate.gif)

## 4、AnticipaOvershootInterpolator

先往回拉一小段距离，在先前运动，最后超出终点一小段距离再回到终点。

![](C:\Users\jenny\Pictures\BlogPictures\anticipateovershoot.gif)

## 5、BounceInterpolator

向前运动，在终点处回弹几下。

![](C:\Users\jenny\Pictures\BlogPictures\bounce.gif)

## 6、CycleInterpolator

CycleInterpolator(float cycles)，参数表示来回运动次数。在起点和终点之间来回运动，重复几次由它的cycles参数决定。

![](C:\Users\jenny\Pictures\BlogPictures\cycles.gif)

## 7、DecelerateInterpolator

初速度最大，然后一直加速运动到终点。

![](C:\Users\jenny\Pictures\BlogPictures\decelerate.gif)

## 8、LinearInterpolator

匀速运动。

![](C:\Users\jenny\Pictures\BlogPictures\linear.gif)

## 9、OvershootInterpator

它跟AnticipaOvershootInterpolator的区别是，刚开始的时候不需要往后拉一小段距离；相同之处是运动地超过终点一部分然后回到终点。

![](C:\Users\jenny\Pictures\BlogPictures\overshoot.gif)

