[toc]

# 一、前言

近来在开发时，经常使用到inflate方法加载视图布局，并且回调onFinishInflate方法进行一些初始化的操作。

顿时心血来潮，想要探究一下Layoutinflater的原理，怎么就把XML格式的布局文件加载为布局的实例对象，对于一些特殊标签，例如\<merge>，\<include>如何处理的，所以带着以下👇问题探究一下：

1. LayoutInflater源码解析
   - view的加载流程
   - 特殊标签\<merge>，\<include>的处理
   - view实际创建过程，从xml定义到内存的视图实例（onCreateView， createView）
2. onFinishInflate调用机制和时机
3. inflate方法中的root和attachRoot参数的作用

# 二、LayoutInflater源码解析

​	通常我们动态加载一个布局文件是通过LayoutInflater的inflate方法来完成，其实在Activity中，setContentView方法底层也是调用的该方法完成。相关代码在PhoneWindow类：

```java
		//PhoneWindow.java
		@Override
    public void setContentView(int layoutResID) {
        // Note: FEATURE_CONTENT_TRANSITIONS may be set in the process of installing the window
        // decor, when theme attributes and the like are crystalized. Do not check the feature
        // before this happens.
        if (mContentParent == null) {
            installDecor();
        } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            mContentParent.removeAllViews();
        }

        if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                    getContext());
            transitionTo(newScene);
        } else {
            mLayoutInflater.inflate(layoutResID, mContentParent);
        }
        mContentParent.requestApplyInsets();
        final Callback cb = getCallback();
        if (cb != null && !isDestroyed()) {
            cb.onContentChanged();
        }
        mContentParentExplicitlySet = true;
    }
```

好了话不多说，进入正题

## 2.1 View加载流程

在Android中，我们只需写好各种布局文件，使用Inflate方法去加载就好，通常使用LayoutInflater.from()方法获取LayoutInflater实例，然后调研inflate方法加载布局，它有如下重载方法

```java
public View inflate(@LayoutRes int resource, @Nullable ViewGroup root)
public View inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot)
public View inflate(XmlPullParser parser, @Nullable ViewGroup root)
public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot)
```

这里root和attachToRoot参数作用下面分析源码时详细说明。

既然是源码分析，也就不得不摆出大段大段代码，已将无用代码省略掉，尽量让大家看起显得简洁。inflate方法代码如下，相关说明注释在代码中：

```java
public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
        synchronized (mConstructorArgs) {
            Trace.traceBegin(Trace.TRACE_TAG_VIEW, "inflate");

            final Context inflaterContext = mContext;
            final AttributeSet attrs = Xml.asAttributeSet(parser);
            Context lastContext = (Context) mConstructorArgs[0];
            mConstructorArgs[0] = inflaterContext;
            View result = root;

            try {
                // Look for the root node.
                int type;
                while ((type = parser.next()) != XmlPullParser.START_TAG &&
                        type != XmlPullParser.END_DOCUMENT) {
                    // Empty
                }
								//没有开始标签，说明布局文件语法错误，解析不出来
                if (type != XmlPullParser.START_TAG) {
                    throw new InflateException(parser.getPositionDescription()
                            + ": No start tag found!");
                }

                final String name = parser.getName();
                ......
                ①// TAG_MERGE就是merge标签，条件体是merge的加载
                if (TAG_MERGE.equals(name)) {
                    if (root == null || !attachToRoot) {
                        throw new InflateException("<merge /> can be used only with a valid "
                                + "ViewGroup root and attachToRoot=true");
                    }
                    rInflate(parser, root, inflaterContext, attrs, false);
                } else {
                    // Temp is the root view that was found in the xml
                    ②final View temp = createViewFromTag(root, name, inflaterContext, attrs);
                    ViewGroup.LayoutParams params = null;
                    if (root != null) {
                        ......
                        // Create layout params that match root, if supplied
                        params = root.generateLayoutParams(attrs);
                        ③//这里也能看出，只有root不为null，attachToRoot才起作用
                        if (!attachToRoot) {
                            // Set the layout params for temp if we are not
                            // attaching. (If we are, we use addView, below)
                            temp.setLayoutParams(params);
                        }
                    }
										......
                    ④// Inflate all children under temp against its context.
                    rInflateChildren(parser, temp, attrs, true);
                    ......
                    // We are supposed to attach all the views we found (int temp)
                    // to root. Do that now.
                    if (root != null && attachToRoot) {
                        root.addView(temp, params);
                    }

                    // Decide whether to return the root that was passed in or the
                    // top view found in xml.
                    if (root == null || !attachToRoot) {
                        result = temp;
                    }
                }

            } catch (XmlPullParserException e) {
                ......
            }
            return result;
        }
    }
```

加载时，首先判断是否是\<merge>布局，是的话走到该布局的加载流程，否则先加载根视图，然后递归加载根视图下所有子视图。**②**处，**createViewFromTag**方法就是讲布局加载为实例对象的方法，此处暂不深究内部实现，下面有具体分析。加载完根视图后，代码走到**④**处，**rInflateChildren**开始递归加载所有子视图，进入方法内部：

```java
final void rInflateChildren(XmlPullParser parser, View parent, AttributeSet attrs,
            boolean finishInflate) throws XmlPullParserException, IOException {
        rInflate(parser, parent, parent.getContext(), attrs, finishInflate);
    }
```

可见，视图递归加载是通过**rInflate**方法来完成。

```java
void rInflate(XmlPullParser parser, View parent, Context context,
        AttributeSet attrs, boolean finishInflate) throws XmlPullParserException, IOException {

    final int depth = parser.getDepth();
    int type;
    boolean pendingRequestFocus = false;

    while (((type = parser.next()) != XmlPullParser.END_TAG ||
            parser.getDepth() > depth) && type != XmlPullParser.END_DOCUMENT) {

        if (type != XmlPullParser.START_TAG) {
            continue;
        }

        final String name = parser.getName();

        if (TAG_REQUEST_FOCUS.equals(name)) {
            pendingRequestFocus = true;
            consumeChildElements(parser);
        } else if (TAG_TAG.equals(name)) {
            parseViewTag(parser, parent, attrs);
        } else if (TAG_INCLUDE.equals(name)) {①
            if (parser.getDepth() == 0) {
                throw new InflateException("<include /> cannot be the root element");
            }
            parseInclude(parser, context, parent, attrs);
        } else if (TAG_MERGE.equals(name)) {②
            throw new InflateException("<merge /> must be the root element");
        } else {③
            final View view = createViewFromTag(parent, name, context, attrs);
            final ViewGroup viewGroup = (ViewGroup) parent;
            final ViewGroup.LayoutParams params = viewGroup.generateLayoutParams(attrs);
            rInflateChildren(parser, view, attrs, true);
            viewGroup.addView(view, params);
        }
    }

    if (pendingRequestFocus) {
        parent.restoreDefaultFocus();
    }

    if (finishInflate) {
        parent.onFinishInflate();④
    }
}
```

在循环体内，会根据标签名称来加载对应视图，由①知\<include>不能作为根元素，而 \<merge>必须为根节点。③处就是普通视图的加载逻辑，分别获取对应View的实例对象，父View的LayoutParams，然后继续递归加载子View。

\<include>布局的加载在parseInclude方法中，只能在ViewGroup中使用该标签且必须指定layout属性，否则会抛InflateException。除了layout属性，加载\<include>时，还会对id、visibility等属性做出处理，然后就是调用rInflateChildren方法去递归加载子布局。

## 2.2 View的创建

从XML中定义的View到内存中储存的对应View对象，是通过**createViewFromTag**方法来完成。具体代码👇：

```java
View createViewFromTag(View parent, String name, Context context, AttributeSet attrs,
            boolean ignoreThemeAttr) {
        if (name.equals("view")) {
            name = attrs.getAttributeValue(null, "class");
        }

        // Apply a theme wrapper, if allowed and one is specified.
        if (!ignoreThemeAttr) {
            final TypedArray ta = context.obtainStyledAttributes(attrs, ATTRS_THEME);
            final int themeResId = ta.getResourceId(0, 0);
            if (themeResId != 0) {
                context = new ContextThemeWrapper(context, themeResId);
            }
            ta.recycle();
        }

        try {
            View view = tryCreateView(parent, name, context, attrs);
						
            if (view == null) {
                final Object lastContext = mConstructorArgs[0];
                mConstructorArgs[0] = context;
                try {
                    if (-1 == name.indexOf('.')) {
                        view = onCreateView(context, parent, name, attrs);
                    } else {
                        view = createView(context, name, null, attrs);
                    }
                } finally {
                    mConstructorArgs[0] = lastContext;
                }
            }
            return view;
        } catch (InflateException e) {
           ......
    }
```

首先是添加Theme资源（若有），然后进入**tryCreateView**方法。

```java
public final View tryCreateView(@Nullable View parent, @NonNull String name,
        @NonNull Context context,
        @NonNull AttributeSet attrs) {
        if (name.equals(TAG_1995)) {①
            // Let's party like it's 1995!
            return new BlinkLayout(context, attrs);
        }
        View view;
        if (mFactory2 != null) {②
            view = mFactory2.onCreateView(parent, name, context, attrs);
        } else if (mFactory != null) {
            view = mFactory.onCreateView(name, context, attrs);
        } else {
            view = null;
        }
        if (view == null && mPrivateFactory != null) {
            view = mPrivateFactory.onCreateView(parent, name, context, attrs);
        }
        return view;
    }
```

①处是1个彩蛋，blink意为眨眼、闪烁，相关渊源感兴趣的可以自己查查。从②处起，增加了几个接口去拦截创建视图的流程，开发者可以自定义**Factory2**和**Factory**接口方法**onCreateView** 去拦截加载流程，针对不同业务场景实现想要的效果。有个例子，当在一些特殊纪念日（例如9.18），可能需要APP界面显示灰色。我们不可能去替换所有图片，然后改变各个View的颜色。但我们知道，Activity的根布局是content view，它是FrameLayout。我们只用写一套显示增加了灰度效果的FrameLayout，然后在加载时重写Factory2和Factory接口方法onCreateView 去拦截，就能实现相应的效果。具体例子可看大佬的文章：[App 黑白化实现探索，有一行代码实现的方案吗？](https://blog.csdn.net/lmj623565791/article/details/105319752)

正常情况下，没有拦截View创建过程，会进入到

```java
					......					
					if (view == null) {
                final Object lastContext = mConstructorArgs[0];
                mConstructorArgs[0] = context;
                try {
                    if (-1 == name.indexOf('.')) {①
                        view = onCreateView(context, parent, name, attrs);②
                    } else {
                        view = createView(context, name, null, attrs);③
                    }
                } finally {
                    mConstructorArgs[0] = lastContext;
                }
            }
					return view;
					......
```

①处判断是否有「.」号，通常是自定义View（自定义 View的形式为<package_name>.customViewName），自定义View直接调用③处方法createView，代码如下👇(部分代码已省略)：

```java
public final View createView(@NonNull Context viewContext, @NonNull String name,
            @Nullable String prefix, @Nullable AttributeSet attrs)
            throws ClassNotFoundException, InflateException {
        Objects.requireNonNull(viewContext);
        Objects.requireNonNull(name);
        Constructor<? extends View> constructor = sConstructorMap.get(name);
        if (constructor != null && !verifyClassLoader(constructor)) {
            constructor = null;
            sConstructorMap.remove(name);
        }
        Class<? extends View> clazz = null;

        try {
            Trace.traceBegin(Trace.TRACE_TAG_VIEW, name);

            if (constructor == null) {
                // Class not found in the cache, see if it's real, and try to add it
                clazz = Class.forName(prefix != null ? (prefix + name) : name, false,
                        mContext.getClassLoader()).asSubclass(View.class);

                if (mFilter != null && clazz != null) {
                    boolean allowed = mFilter.onLoadClass(clazz);
                    if (!allowed) {
                        failNotAllowed(name, prefix, viewContext, attrs);
                    }
                }
                constructor = clazz.getConstructor(mConstructorSignature);
                constructor.setAccessible(true);
                sConstructorMap.put(name, constructor);
            } else {
                // If we have a filter, apply it to cached constructor
                if (mFilter != null) {
                    // Have we seen this name before?
                    Boolean allowedState = mFilterMap.get(name);
                    if (allowedState == null) {
                        // New class -- remember whether it is allowed
                        clazz = Class.forName(prefix != null ? (prefix + name) : name, false,
                                mContext.getClassLoader()).asSubclass(View.class);

                        boolean allowed = clazz != null && mFilter.onLoadClass(clazz);
                        mFilterMap.put(name, allowed);
                        if (!allowed) {
                            failNotAllowed(name, prefix, viewContext, attrs);
                        }
                    } else if (allowedState.equals(Boolean.FALSE)) {
                        failNotAllowed(name, prefix, viewContext, attrs);
                    }
                }
            }

            Object lastContext = mConstructorArgs[0];
            mConstructorArgs[0] = viewContext;
            Object[] args = mConstructorArgs;
            args[1] = attrs;

            try {
                final View view = constructor.newInstance(args);①
                if (view instanceof ViewStub) {②
                    // Use the same context when inflating ViewStub later.
                    final ViewStub viewStub = (ViewStub) view;
                    viewStub.setLayoutInflater(cloneInContext((Context) args[0]));③
                }
                return view;
            } finally {
                mConstructorArgs[0] = lastContext;
            }
        } catch (NoSuchMethodException e) {
        	......
        }
    }
```

可以看出，View的创建是通过反射，调用View的构造方法完成的（①）。对于ViewStub，会为其设置LayoutInflater对象。

至此，inflate方法加载View的流程和相关代码已梳理一遍。

**总结一下**：

- View的加载首先加载根布局，然后递归加载各个子View。
- 对于\<merge>，\<include>，\<ViewStub>等标签有特殊的处理。
- View对象的创建实际通过反射方式，调用该View的构造方法完成。

# 三、onFinishInflate调用机制和时机

在自定义View时，通常在onFinishInflate回调中进行一些初识化操作或者判断View是否创建完成，该方法的调用位置在rInflate方法中：

```java
void rInflate(XmlPullParser parser, View parent, Context context,
            AttributeSet attrs, boolean finishInflate) throws XmlPullParserException, IOException {
				......
        while (((type = parser.next()) != XmlPullParser.END_TAG ||
                parser.getDepth() > depth) && type != XmlPullParser.END_DOCUMENT) {

            if (type != XmlPullParser.START_TAG) {
                continue;
            }

            final String name = parser.getName();
            if (TAG_REQUEST_FOCUS.equals(name)) {
                ......
            } else {
                final View view = createViewFromTag(parent, name, context, attrs);
                final ViewGroup viewGroup = (ViewGroup) parent;
                final ViewGroup.LayoutParams params = viewGroup.generateLayoutParams(attrs);
                rInflateChildren(parser, view, attrs, true);
                viewGroup.addView(view, params);
            }
        }

        if (pendingRequestFocus) {
            parent.restoreDefaultFocus();
        }

        if (finishInflate) {
            parent.onFinishInflate();
        }
    }
```

加载完视图后finishInflate为true，调用parent.onFinishInflate()。例如我们有如下布局文件：

```xml
<LinearLayout>
  <TextView>
    ......
  </TextView>
</LinearLayout>
```

首先加载根节点\<LinearLayout>，然后进入rInflate，进入循环体，遇到\<TextView>开始解析，创建TextView对象并添加到父视图中（LinearLayout）。然后递归创建子视图，rInflateChildren会把finishInflate参数设为true，然后还是会走到rInflate中，由于是结束标签，不会进入循环体。最后会调用onFinishInflate，所以如果一个View重写了onFinishInflate方法，那么在该View创建完成后便会调用该方法。

写个demo来验证一下，自定义一个简单View布局，重写onFinishInflate，打印一句话。

```java 
public class customTextView extends AppCompatTextView {
    public customTextView(Context context) {
        super(context);
    }

    public customTextView(Context context, AttributeSet attrs) {
        super(context, attrs);
    }

    public customTextView(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
    }

    @Override
    protected void onFinishInflate() {
        super.onFinishInflate();
        Log.d(InflateTestView.TAG, "customTextView onFinishInflate");
    }

    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
    }
}
```

```java
public class InflateTestView extends LinearLayout {
    public static final String TAG = "InflateTestView";
    public InflateTestView(Context context) {
        this(context, null);
    }

    public InflateTestView(Context context, @Nullable AttributeSet attrs) {
        this(context, attrs, 0);
    }

    public InflateTestView(Context context, @Nullable AttributeSet attrs, int defStyleAttr) 		{
        super(context, attrs, defStyleAttr);
    }
  
    @Override
    protected void onFinishInflate() {
        super.onFinishInflate();
        Log.d(TAG, "onFinishInflate");
    }

}
```

主界面的布局文件：

```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@+id/main_layout"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    tools:context=".MainActivity">
    <Button
        android:id="@+id/button"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Inflate测试"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintHorizontal_bias="0.3"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent"
        app:layout_constraintVertical_bias="0.499" />
</LinearLayout>
```

`inflateView`布局文件：

```xml
<com.example.servicedemo.InflateTestView
    android:id="@+id/inflate_view"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:orientation="vertical"
    android:background="@android:color/holo_orange_light"
    xmlns:android="http://schemas.android.com/apk/res/android">
    <com.example.servicedemo.customTextView
        android:text="customTextView in inflateView"
        android:textSize="20sp"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"/>
</com.example.servicedemo.InflateTestView>
```

点击按钮，动态加载`InflateTestView`视图到`Activity`。

```java
......
   View inflateTestView =  LayoutInflater.from(this).inflate(R.layout.inflate_view, null);
   LinearLayout.LayoutParams params = new LinearLayout.LayoutParams(ViewGroup.LayoutParams.WRAP_CONTENT, ViewGroup.LayoutParams.WRAP_CONTENT);
                mainLayout.addView(inflateTestView, params);
......
```

然后看下打印信息

![image-20201101222600134](/Users/mtdp/Library/Application Support/typora-user-images/image-20201101222600134.png)

customTextView和InflateTestView都重写了onFinishInflate方法，所以在加载布局文件时，创建对应View对象后就会调用onFinishInflate。

**总结**：只要重写了onFinishInflate，那么通过inflate加载布局时，每个View对象创建完成后都会调用onFinishInflate

# 四、root和attachRoot参数的作用

关于root和attachRoot参数的作用，还是要到代码中去找，在inflate方法中：

```java
...... 										
// Temp is the root view that was found in the xml
                    final View temp = createViewFromTag(root, name, inflaterContext, attrs);
                    ViewGroup.LayoutParams params = null;
                    if (root != null) {
                        ......
                        // Create layout params that match root, if supplied
                        params = root.generateLayoutParams(attrs);
                        if (!attachToRoot) {
                            // Set the layout params for temp if we are not
                            // attaching. (If we are, we use addView, below)
                            temp.setLayoutParams(params);
                        }
                    }
                    // Inflate all children under temp against its context.
                    rInflateChildren(parser, temp, attrs, true);
                    // We are supposed to attach all the views we found (int temp)
                    // to root. Do that now.
                    if (root != null && attachToRoot) {
                        root.addView(temp, params);
                    }

                    // Decide whether to return the root that was passed in or the
                    // top view found in xml.
                    if (root == null || !attachToRoot) {
                        result = temp;
                    }
                }
......
```

temp是root view对象，如果root参数不为null，那么会生成该View的LayoutParams对象（params）；此时，如果attachToRoot参数为false，那么会将params设置为根视图的layout params。

如果root参数不为null且attachToRoot参数为true，就把当前视图添加到root所代表的View中。

