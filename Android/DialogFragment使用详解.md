[TOC]

# 一、前言

DialogFragment是用于展示弹窗的API，与Dialog不同，DialogFragment本质上是一个Fragment，也就具有Fragment所拥有的生命周期。在使用时，更容易通过生命周期回调来管理弹窗。对于复杂样式的弹窗，使用DialogFragment更加方便和高效。当然，任何API都并非没有问题和完美适用与任何情况，开发时根据具体场景和业务特征来选用。

# 二、DialogFragment使用

## 2.1 DialogFragment的创建

DialogFragment的使用也是非常简便，有2种方式来创建DialogFragment。

### 1、重写onCreateDialog()

在该方法内使用AlertDialog等创建dialog，适用于使用系统样式弹窗的情况

```java
public Dialog onCreateDialog(@Nullable Bundle savedInstanceState) {
        AlertDialog dialog = new AlertDialog.Builder(getContext())
                .setTitle("系统弹窗")
                .setMessage("信息")
                .setIcon(R.drawable.assign_set_question_ic_v2)
                .setNegativeButton("取消", new DialogInterface.OnClickListener() {
                    @Override
                    public void onClick(DialogInterface dialog, int which) {
                    }
                }).setPositiveButton("确定", new DialogInterface.OnClickListener() {
                    @Override
                    public void onClick(DialogInterface dialog, int which) {
                        Toast.makeText(getContext(), "确认", Toast.LENGTH_SHORT).show();
                    }
                }).create();
        return dialog;
    }
```

![](/Users/mtdp/Documents/BlogPicture/DialogFragment使用详解-1.png)

### 2、重写onCreateView()

同创建Fragment方式类似，在该方法内加载自定义布局，适用于使用自定义样式弹窗的情况。

```java
        @Override
    public View onCreateView(@NonNull LayoutInflater inflater, @Nullable ViewGroup container, @Nullable Bundle savedInstanceState) {
        return inflater.inflate(R.layout.fragment_base, container, false);
    }
```

![](/Users/mtdp/Documents/BlogPicture/DialogFragment使用详解-2.png)

**发现自定义弹窗的布局没有生效**，查看其布局文件

```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

   ......
</androidx.constraintlayout.widget.ConstraintLayout>
```

已经设置了弹窗宽高为MATCH_PARENT，可见**在布局文件中设置DialogFragment样式是无效的**，这也是使用DialogFragment需要注意的问题。下面👇将说明如何解决该问题。

上述2种方式创建好了弹窗，只需要创建DialogFragment对象，然后调用👇接口就可以显示出来了

```java
 public void show(FragmentManager manager, String tag)
```

本小节简要说明了创建和使用DialogFragment的基本姿势，需要注意的是，**如果同时重写这2种创建dialog的方法，那么在显示时以onCreateDialog为最终效果。**

## 2.2 DialogFragment布局不生效

上节提及过，如果使用自定义方式创建dialog，那么在布局文件中声明的样式是无效的。为了解决这个问题，需要在DialogFragment的onStart回调中获取Dialog的Window对象，通过Window对象来设置Dialog的布局和样式。例如

```java
        @Override
    public void onStart() {
        super.onStart();
        Dialog dialog = getDialog();
        if (dialog == null || dialog.getWindow() == null){
            return;
        }
        dialog.getWindow().setLayout(ViewGroup.LayoutParams.WRAP_CONTENT, 800);
        dialog.getWindow().setBackgroundDrawableResource(R.drawable.assign_set_bg);
    }
```

![](/Users/mtdp/Documents/BlogPicture/DialogFragment使用详解-3.png)

发现在onStart方法中设置后确实生效了，那么问题来了，为什么要在onStart方法中设置呢？其他地方比如onCreate、onViewCreated中可以吗?简单实验下，发现不可行。

分析下源码：

#### 1、setBackgroundDrawableResource

1. 先看setBackgroundDrawableResource，实际调用的是Window#setBackgroundDrawable，它是个抽象方法。众所周知，Window的唯一实现类是PhoneWindow，在其中找到对应方法
   
   ```java
       public abstract void setBackgroundDrawable(Drawable drawable);
   ```

2. PhoneWindow#setBackgroundDrawable
   
   ```java
           @Override
       public final void setBackgroundDrawable(Drawable drawable) {
           if (drawable != mBackgroundDrawable) {
               mBackgroundDrawable = drawable;
               if (mDecor != null) {
                   mDecor.setWindowBackground(drawable);
                   if (mBackgroundFallbackDrawable != null) {
                       mDecor.setBackgroundFallback(drawable != null ? null :
                               mBackgroundFallbackDrawable);
                   }
               }
           }
       }
   ```
   
   其中第6行，mDecor.setWindowBackground(drawable)，mDecor是个DecorView实例，如果mDecor不为null的话，就设置背景。对于DialogFragment来说，就是Dialog的DecorView实例。所以下一步我们只需**确认「mDecor」初始化的位置。**

3. 目标聚焦于找寻Dialog的DecorView实例初始化位置，阅读Dialog源码，发现Dialog的mDecor初识化位置正是在其show方法中。
   
   ```java
   public void show() {
           if (mShowing) {
               if (mDecor != null) {
                   if (mWindow.hasFeature(Window.FEATURE_ACTION_BAR)) {
                       mWindow.invalidatePanelMenu(Window.FEATURE_ACTION_BAR);
                   }
                   mDecor.setVisibility(View.VISIBLE);
               }
               return;
           }
   
           mCanceled = false;
   
           if (!mCreated) {
               dispatchOnCreate(null);
           } else {
               // Fill the DecorView in on any configuration changes that
               // may have occured while it was removed from the WindowManager.
               final Configuration config = mContext.getResources().getConfiguration();
               mWindow.getDecorView().dispatchConfigurationChanged(config);
           }
   
           onStart();
                 /**
             * ① mDecor初始化位置
           */
           mDecor = mWindow.getDecorView();
   
           if (mActionBar == null && mWindow.hasFeature(Window.FEATURE_ACTION_BAR)) {
               final ApplicationInfo info = mContext.getApplicationInfo();
               mWindow.setDefaultIcon(info.icon);
               mWindow.setDefaultLogo(info.logo);
               mActionBar = new WindowDecorActionBar(this);
           }
   
           WindowManager.LayoutParams l = mWindow.getAttributes();
           boolean restoreSoftInputMode = false;
           if ((l.softInputMode
                   & WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION) == 0) {
               l.softInputMode |=
                       WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION;
               restoreSoftInputMode = true;
           }
           mWindowManager.addView(mDecor, l);
           if (restoreSoftInputMode) {
               l.softInputMode &=
                       ~WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION;
           }
           mShowing = true;
           sendShowMessage();
       }
   ```
   
   接着反向推到DialogFragment又在何时调用了Dialog#show方法，很简单的定位一下位置，发现
   
   ```java
   // DialogFragment.class
           @Override
       public void onStart() {
           super.onStart();
   
           if (mDialog != null) {
               mViewDestroyed = false;
               mDialog.show();
           }
       }
   ```
   
   所以，正是在DialogFragment的onStart方法中调用了Dialog#show方法，初始化了Dialog#mDecor对象，才能使布局设置生效。
   
   #### 2、setLayout方法
   
   1. Window#setLayout
      
      ```java
      public void setLayout(int width, int height) {
              final WindowManager.LayoutParams attrs = getAttributes();
              attrs.width = width;
              attrs.height = height;
              dispatchWindowAttributesChanged(attrs);
      }
      ```
      
      只是给Window的属性赋值，既然设置了宽高数据，那么一定有一个方法是将所设置数据（宽高）应用到Window上。还是在PhoneWindow中，有个setAttributes方法
   
   2. PhoneWindow#setAttributes
      
      ```java
          @Override
          public void setAttributes(WindowManager.LayoutParams params) {
              super.setAttributes(params);
              if (mDecor != null) {
                  mDecor.updateLogTag(params);
              }
          }
      ```
      
      还是需要初始化了mDecor对象，才能使设置生效

**综上，动态设置DialogFragment的布局和样式必须在Dialog的DecorView实例初始化之后才能生效，即在onStart方法中去设置。**
