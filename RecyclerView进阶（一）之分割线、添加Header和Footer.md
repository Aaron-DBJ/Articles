# RecyclerView进阶（一）之分割线、添加Header和Footer

如今越来越多的开发者开始使用RecyclerView，与传统的ListView相比，它有许多优势：有更多的布局方式，更好的动画效果，更加灵活容易扩展，有局部刷新的能力等等。但不是说这样就能而完全取代ListView，毕竟它添加分割线、Header和Footer十分方便，而且自带有item的点击事件。所以使用RecyclerView还是ListView还是看具体应用场景。



## RecyclerView之ItemDecoration

​	在RecyclerView中，没有`divider`属性来添加分割线；所以在开发中，RecyclerView使用这个类ItemDecoration来添加分割线，如下：

```java
...
ItemDecoration decoration = new DividerItemDecoration(this, DividerItemDecoration.VERTICAL);
recyclerView.addItemDecoration(decoration);
...
```

​	但它的能力远非如此，话不多说，先看两个效果：

![粘性头部的效果](C:\Users\jenny\Pictures\BlogPictures\stiky_header_decoration2.gif)

![HeaderFooter](C:\Users\jenny\Pictures\BlogPictures\header_footer.gif)

​	要实现这样的效果只需要继承RecyclerView的ItemDecoration，重写它里面的方法就好了。先看一下ItemDecoration的源码，有下面三个方法：

``` java
 public void getItemOffsets(Rect outRect, View view, RecyclerView parent, State state)
     
/**
         * Draw any appropriate decorations into the Canvas supplied to the RecyclerView.
         * Any content drawn by this method will be drawn after the item views are drawn
         * and will thus appear over the views.
         *
         * @param c Canvas to draw into
         * @param parent RecyclerView this ItemDecoration is drawing into
         * @param state The current state of RecyclerView.
         */
public void onDrawOver(Canvas c, RecyclerView parent, State state)
/**
         * Draw any appropriate decorations into the Canvas supplied to the RecyclerView.
         * Any content drawn by this method will be drawn after the item views are drawn
         * and will thus appear over the views.
         *
         * @param c Canvas to draw into
         * @param parent RecyclerView this ItemDecoration is drawing into
         * @param state The current state of RecyclerView.
         */
public void onDrawOver(Canvas c, RecyclerView parent, State state)
```

### OutRect

​	在讲上述三个方法前，先了解一下什么是`OutRect`。

![OutRect](C:\Users\jenny\Pictures\BlogPictures\outRect.png)

`OutRect`就是每个itemview下的矩形区域，默认情况下，`outRect`的left、top、right和bottom的值为0；itemview和OutRect是重合，就只能看见itemview。所以函数：

``` java
public void getItemOffsets(Rect outRect, View view, RecyclerView parent, State state)
```

的作用就是把`OutRect`撑开一定的区域，就可以使用`onDraw()` 和`onDrawOver()`在这些被撑开的区域进行绘画。

RecyclerView的背景、`onDraw`绘制的内容、`Item`、`onDrawOver`绘制的内容，各层级关系如下[^1]：

![](F:\Chrome下载\cengjiguanxi.png)

也就是说，onDraw绘制的内容可能会被itemView遮挡，itemView也可能被onDrawOver绘制的内容遮挡。

### 分割线

​	我们想给RecyclerView添加分割线，**思路是把每个itemView的顶部或底部撑开一定的宽度，注意到在主布局中背景色设置为黑色，itemView的背景色在item_view.xml中设置为白色。那么当itemView顶部或底部撑开一段距离后，背景色就露了出来，看上去就像一条条的分割线。**废话少说，开始动手写吧。我们创建一个新项目，添加RecyclerView的依赖后，在MainActivity中添加一个RecyclerView，布局文件为：

```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:background="#000000"
    android:orientation="vertical"
    tools:context=".MainActivity">

   <android.support.v7.widget.RecyclerView
       android:id="@+id/recycler_view"
       android:layout_width="match_parent"
       android:layout_height="match_parent">
   </android.support.v7.widget.RecyclerView>
</LinearLayout>
```

然后我们来写RecyclerView 的适配器。先定义一个itemVIew 的布局，命名为item_view，代码如下：

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:background="#ffffff"
    android:elevation="5dp">

    <TextView
        android:id="@+id/tv_item"
        android:text="35sp"
        android:gravity="center_vertical"
        android:layout_width="match_parent"
        android:layout_height="70dp" />
</LinearLayout>
```

我们只是演示一下效果，就简单定义一个TextView就好了。Adapter的代码如下：

```java
public class MyAdapter extends RecyclerView.Adapter<RecyclerView.ViewHolder> {
    private List<String> mList;
    private Context context;
	public MyAdapter(List<String> list, Context context){
        mList = list;
        this.context = context;
    }
    class ViewHolder extends RecyclerView.ViewHolder{
        public TextView textView;
        public ViewHolder(View itemView) {
            super(itemView);
            textView = itemView.findViewById(R.id.tv_item);
        }
    }
    @NonNull
    @Override
    public RecyclerView.ViewHolder onCreateViewHolder(@NonNull ViewGroup parent, int viewType) {
        View view = LayoutInflater.from(parent.getContext()).inflate(R.layout.item_view, parent, false);
        return new ViewHolder(view);
    }
    
     @Override
    public void onBindViewHolder(@NonNull RecyclerView.ViewHolder holder, final int position) {
            ((ViewHolder)holder).textView.setText(mList.get(position));
    }
    
    @Override
    public int getItemCount() {
        return mList.size();
    }
}
```

这里开始我们最主要的工作，就是继承ItemDecoration重写它的方法。

```java
public class DividerDecoration extends RecyclerView.ItemDecoration {
    private List<String> mList;
    public DividerDecoration(List<String> list) {
        super();
        mList = list;
    }
    @Override
    public void getItemOffsets(Rect outRect, View view, RecyclerView parent, RecyclerView.State state) {
        super.getItemOffsets(outRect, view, parent, state);
        outRect.top = 2;//单位是像素px
    }
    @Override
    public void onDraw(Canvas c, RecyclerView parent, RecyclerView.State state) {
        super.onDraw(c, parent, state);
    }

    @Override
    public void onDrawOver(Canvas c, RecyclerView parent, RecyclerView.State state) {
        super.onDrawOver(c, parent, state);
    }
}

```

最后在主函数中

```java
public class MainActivity extends AppCompatActivity {
    private List<String> mList;
    private RecyclerView recyclerView;
     @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        recyclerView = findViewById(R.id.recycler_view);
        MyAdapter adapter = new MyAdapter(mList, this);
        LinearLayoutManager manager = new LinearLayoutManager(this, LinearLayoutManager.VERTICAL, false);
        recyclerView.setLayoutManager(manager);
        recyclerView.setAdapter(adapter);
    }
```

效果如图所示：

![](C:\Users\jenny\Pictures\BlogPictures\divider.gif)

## 为RecyclerView添加Header和Footer

说了这么多理论的东西，我们动手来实际操作一下。

### Header和Footer

​	在ListView中，给我们提供了`addHeader()`和`AddFooter()`方法来添加头部和底部。但在RecyclerView中没有封装好的方法供我们调用，需要自己去写相应的方法。从ListView的源码可以知道，它添加Header和Footer的方法是在适配器Adapter中动态添加的；所以在RecyclerView中，我们也要在adapter中添加相应方法。

​	主要思路是：**根据适配器中的`getItemViewType()`方法，来判断当前的itemview的类型是内容列表项还是Header视图或者是Footer视图。对不同的类型加载各自的布局文件，从而实现添加Header和Footer的效果。**

首先创建Header和Footer的布局文件header.xml和footer.xml，这里只在header.xml中写一个TextView，footer.xml布局和header一样。

``` xml
<?xml version="1.0" encoding="utf-8"?>
<android.support.v7.widget.CardView xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    app:cardCornerRadius="0dp"
    app:cardBackgroundColor="#abcd23"
    android:layout_width="match_parent"
    android:layout_height="100dp">
    <TextView
        android:id="@+id/header_title"
        android:textStyle="bold"
        android:text="Header"
        android:layout_gravity="center"
        android:textSize="30sp"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content" />

</android.support.v7.widget.CardView>
```



在RecyclerView的Adapter中，先定义这3种类型、Header视图和Footer视图的数量；Adapter代码如下：

``` java
public class HeaderFooterAdapter extends RecyclerView.Adapter<RecyclerView.ViewHolder> {
    private List<String> mList;
    private Context context;
    /**
     * 定义3中itemview的类型
     */
    private static final int HEADER = 0;
    private static final int CONTENT = 1;
    private static final int FOOTER = 2;
    // Header和Footer的数量都为1。
    private int headerCount = 1;
    private int footerCount = 1;

    public HeaderFooterAdapter(List<String> list, Context context){
        mList = list;
        this.context = context;
    }

    public class HeaderViewHolder extends RecyclerView.ViewHolder{
        public HeaderViewHolder(View itemView) {
            super(itemView);
        }
    }

    public class FooterViewHolder extends RecyclerView.ViewHolder{
        public FooterViewHolder(View itemView) {
            super(itemView);
        }
    }

    public class ViewHolder extends RecyclerView.ViewHolder {
        TextView textView;
        public ViewHolder(View itemView) {
            super(itemView);
            textView = itemView.findViewById(R.id.item_view);
        }
    }
    /** 
     * 根据当前itemView的类型加载各自的视图
     */
    @NonNull
    @Override
    public RecyclerView.ViewHolder onCreateViewHolder(@NonNull ViewGroup parent, int viewType) {
        View view;
        if (viewType == HEADER){
            view = LayoutInflater.from(parent.getContext()).inflate(R.layout.header, parent, false);
            return new HeaderViewHolder(view);
        }else if (viewType == CONTENT){
            view = LayoutInflater.from(parent.getContext()).inflate(R.layout.item_card_view, parent, false);
            return new ViewHolder(view);
        }else if (viewType == FOOTER){
            view = LayoutInflater.from(parent.getContext()).inflate(R.layout.footer, parent, false);
            return new FooterViewHolder(view);
        }
        return null;
    }

    @Override
    public void onBindViewHolder(@NonNull RecyclerView.ViewHolder holder, int position) 	{
        if (holder instanceof HeaderViewHolder){
           
        }else if (holder instanceof ViewHolder){
            //因为第一项是header视图，所以内容视图从第二项开始，也就是从position=1的位置开始；
            //因为要从0获取list的内容，所以序号为position-headerCount。
            ((ViewHolder) holder).textView.setText(mList.get(position - headerCount));
        }else if (holder instanceof FooterViewHolder){

        }
    }

    @Override
    public int getItemCount() {
        return mList.size() + headerCount + footerCount;
    }
	/** 
 	 * 根据itemView的位置返回它的类型
 	 */
    @Override
    public int getItemViewType(int position) {
        if (headerCount != 0 && headerCount > position){
            return HEADER;
        }else if (footerCount != 0 && position >= (getItemCount() - 1)) {
            return FOOTER;
        }else {
            return CONTENT;
        }
    }
}
```

效果如下：

![](C:\Users\jenny\Pictures\BlogPictures\commonHF.gif)



### Header和Footer扩展

上面我们添加了头部视图和底部视图，但只是加载了一个简单的界面，我们也没给它绑定数据，其实header和footer也是itemview，那么itemview能做到的事，header和footer一样能完成。比如最开始的那张动态图，header就被设置成了一个轮播图，这也是很多APP上常见的布局。下面我们就来做做把header设置为一个轮播图。重复代码就不写了，只把新增的代码写下来，因为结构没变，只是添加一些代码内容。轮播图github上很多API，随便找一个就行。

首先还是header和footer的布局文件。去掉header.xml中的`TextView`，添加录播图。

``` xml
<?xml version="1.0" encoding="utf-8"?>
<android.support.v7.widget.CardView xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    app:cardElevation="5dp"
    app:cardCornerRadius="0dp"
    app:cardBackgroundColor="#986953"
    android:layout_width="match_parent"
    android:layout_height="180dp">
    <com.youth.banner.Banner
        android:id="@+id/banner"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"/>

</android.support.v7.widget.CardView>
```

然后修改adapter:

``` java 
public class HeaderFooterAdapter extends RecyclerView.Adapter<RecyclerView.ViewHolder> {
   ...

    public HeaderFooterAdapter(List<String> list, Context context){
		...
    }

    public class HeaderViewHolder extends RecyclerView.ViewHolder{
        Banner banner;
        public HeaderViewHolder(View itemView) {
            super(itemView);
            banner = itemView.findVIewById(R.id.banner);
        }
    }

    public class FooterViewHolder extends RecyclerView.ViewHolder{
        ...
    }

    public class ViewHolder extends RecyclerView.ViewHolder {
        ...
    }
    /** 
     * 根据当前itemView的类型加载各自的视图
     */
    @NonNull
    @Override
    public RecyclerView.ViewHolder onCreateViewHolder(@NonNull ViewGroup parent, int viewType) {
        View view;
        if (viewType == HEADER){
            view = LayoutInflater.from(parent.getContext()).inflate(R.layout.header, parent, false);
            return new HeaderViewHolder(view);
        }else if (viewType == CONTENT){
            view = LayoutInflater.from(parent.getContext()).inflate(R.layout.item_card_view, parent, false);
            return new ViewHolder(view);
        }else if (viewType == FOOTER){
            view = LayoutInflater.from(parent.getContext()).inflate(R.layout.footer, parent, false);
            return new FooterViewHolder(view);
        }
        return null;
    }

    @Override
    public void onBindViewHolder(@NonNull RecyclerView.ViewHolder holder, int position) 	{
        if (holder instanceof HeaderViewHolder){
            ((HeaderViewHolder) holder).banner.setDelayTime(4000)
                    .setBannerStyle(BannerConfig.CIRCLE_INDICATOR_TITLE_INSIDE)
                    .setImageLoader(new MyImageLoader())
                    .setImages(imageList)
                    .setIndicatorGravity(BannerConfig.CENTER)
                    .setBannerTitles(titleList)
                    .isAutoPlay(true)
                    .setBannerAnimation(Transformer.DepthPage)
                    .setOnBannerListener(new OnBannerListener() {
                        @Override
                        public void OnBannerClick(int position) {
                            Toast.makeText(context, "第"+ position + "项", Toast.LENGTH_SHORT).show();
                        }
                    }).start();
        }else if (holder instanceof ViewHolder){
            //因为第一项是header视图，所以内容视图从第二项开始，也就是从position=1的位置开始；
            //因为要从0获取list的内容，所以序号为position-headerCount。
            ((ViewHolder) holder).textView.setText(mList.get(position - headerCount));
        }else if (holder instanceof FooterViewHolder){

        }
    }

    @Override
    public int getItemCount() {
        return mList.size() + headerCount + footerCount;
    }
	/** 
 	 * 根据itemView的位置返回它的类型
 	 */
    @Override
    public int getItemViewType(int position) {
		...
    }
}
```

效果如下：

![](C:\Users\jenny\Pictures\BlogPictures\bannerHF.gif)



由于个人水平有限，如有错误和纰漏之处，还望不吝指教。

[^1]: 图片引用自[层级图](http://www.jcodecraeer.com/a/anzhuokaifa/2017/0615/8079.html)，侵删