[toc]

# 一、前言

众所周知，由于Android碎片化严重，屏幕尺寸适配可说是Android开发中非常繁琐也令人头疼的问题。本文主要是讲解一些和屏幕信息相关的API，以便开发中，如果涉及到UI的适配，能够准确找到所需要的API来满足需求。

# 二、屏幕信息API

## 2.1 基本单位

在开发中，设置空间布局的时候常用的一些单位有：px、dp、sp和dpi等，它们各自的含义如下。

- px：像素，就是构成图像的最小单位

- dp/dip：密度无关像素（Density Independent Pixels），它以160dp为基准，在屏幕密度160dp的机型上，1dp = 1px。以此类推

  | 机型dp | dp和px关系    |
  | ------ | ------------- |
  | 160    | 1 dp = 1 px   |
  | 240    | 1 dp = 1.5 px |
  | 320    | 1 dp = 2px    |
  | 480    | 1dp = 3px     |

- dpi：屏幕像素密度，注意不要和dp/dip混淆了，它表示每英寸的像素个数。通常所说的屏幕尺寸（xx英寸）其实就是屏幕对角线长度，所以dpi计算公式：dpi = 对角线上像素个数 / 屏幕大小

## 2.2 WindowManager

WindowManager作为窗口管理器，只是提供一个获取屏幕信息类Display的方法*getDefaultDisplay。*

## 2.3 Display

Display类提供有关逻辑显示的大小和密度的信息，注意是逻辑显示而不一定是物理显示设备，因为存在外界显示屏等情况。而且显示区域也有2种理解：

1. 应用程序显示区域

   由于顶部的系统状态栏和底部的导航栏存在，Android系统上APP的显示区域不会充满整个屏幕。除去这些系统装饰之外的区域才是应用程序显示区域，通常来说是小于物理屏幕显示区域的。应用程序显示区域可以使用***getSize***， ***getRectSize***和***getMetrics***去获取。

2. 实际显示区域

   这个区域就是整个屏幕的物理显示范围了，可以使用***getRealSize***和***getRealMetrics***方法来获取屏幕信息。

**1、getMetrics**

获取描述该显示器尺寸和密度的显示器指标，**注意**用此方法返回的尺寸不一定代表显示器的实际原始尺寸（本机分辨率）。它会排除一直可见的系统装饰（系统导航栏和状态栏），通常来说返回的是应用程序显示区域。

**2、getRealMetrics**

返回显示屏幕实际大小（包括系统性装饰）

在开发中需要根据需要来选用，确保实际UI效果符合预期。

## 2.4 DisplayMetrics

该类用于描述有关显示器的一般信息，例如大小，密度和字体缩放。它定义了许多变量来描述显示区域的宽高、dp、dpi和sp等信息。访问这些成员变量的方法：

```java
// ① 获取应用程序显示区域信息
DisplayMetrics metrics = new DisplayMetrics();
//调用👇方法后，屏幕信息就存储在metrics中，可以访问了
getWindowManager().getDefaultDisplay().getMetrics(metrics);

// ② 获取实际显示区域信息
DisplayMetrics realMetrics = new DisplayMetrics();
getWindowManager().getDefaultDisplay().getRealMetrics(realMetrics);
```