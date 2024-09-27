# RecyclerView进阶（二）ItemDecoration之分组栏的吸顶效果



在[RecyclerView进阶（一）之分割线、添加Header和Footer](https://blog.csdn.net/qq_21830869/article/details/85126443)展示过一个类似微信通讯录的粘性头部效果。

![粘性头部效果](https://github.com/Aaron-DBJ/MyItemDecoration/tree/master/Images/stiky_header_decoration2.gif)

这篇文章就来实现一下这个效果。

像上面这种分组栏，它是通过ItemDecoration实现的；也就是说继承ItemDecoration并重写它的方法绘制出来的，注意一下：**分组栏本身不是itemView，它本质上是在itemView撑开的区域上画出来的图案**。

## 普通的分组栏

我们首先复习一下RecyclerView的背景，`ondraw（）`绘制的内容，`itemView`，`onDrawOver()`绘制的内容的层级关系，这样对后面分组栏的绘制有一个较好的理解；它们刚好是按顺序从最底层到最顶层。观察这个粘性头部的效果，发现给itemView分组后，始终有一个分组栏在最顶部，而且不能被itemView遮盖。说明它是`onDrawOver()`方法绘制出来。而且要使每组第一个itemView上有一个分组栏，就要判断出每组第一个itemView，然后将它的顶部撑开一定的区域来绘制分组栏。所以这种效果的思路就出来：**判断每组的第一个itemView，然后将它的顶部撑开一定的区域来绘制分组栏**

既然要进行分组，那么我们首先定义一个组的类，这里就以省份来分组。

``` java 
public class Province {
    private int provinceId;
    private String provinceName;
    //recyclerView中每个itemview的位置，也就是城市的位置序号
    private int cityPosition;
    private Bitmap provinceBitmap;
    private String provinceBackground;

    public Province(int provinceId, String provinceName) {
        this.provinceId = provinceId;
        this.provinceName = provinceName;
    }

    public Bitmap getProvinceBitmap() {
        return provinceBitmap;
    }

    public void setProvinceBitmap(Bitmap provinceBitmap) {
        this.provinceBitmap = provinceBitmap;
    }

    public String getProvinceBackground() {
        return provinceBackground;
    }

    public void setProvinceBackground(String provinceBackground) {
        this.provinceBackground = provinceBackground;
    }

    public boolean isFirstCityInProvince(){
        if(cityPosition == 0){
            return true;
        }
        return false;
    }

    public int getProvinceId() {
        return provinceId;
    }

    public void setProvinceId(int provinceId) {
        this.provinceId = provinceId;
    }

    public String getProvinceName() {
        return provinceName;
    }

    public void setProvinceName(String provinceName) {
        this.provinceName = provinceName;
    }

    public int getCityPosition() {
        return cityPosition;
    }

    public void setCityPosition(int position) {
        this.cityPosition = position;
    }
}
```

我们定义了省份的id和名字，还有城市的位置序号，城市就是在itemView中展示的内容。其中有一个重要方法，

``` java 
public boolean isFirstCityInProvince(){
        if(cityPosition == 0){
            return true;
        }
        return false;
    }
```

来判断当前itemView是不是该分组的第一项。有了分组的类，我们新建一个接口，里面提供一个方法来返回Province类的对象。

``` java
public interface OnProvinceListener{
    public Province getProvince(int position);
}
```

通过接口返回的Province对象，我们就可以知道分组的信息，方便我们绘制分组栏使用。

接下来我们开始继承ItemDecoration类重写它的方法，前面说过，分组栏要在最顶层，不能被itemView遮挡，所以在`onDrawOver()`方法中进行分组栏的绘制。

``` java
public class StickyHeaderDecoration extends RecyclerView.ItemDecoration{
    //定义分组栏的高度
    private static final int headerHeight = 200;
    //定义分割线的宽度
    private static final int divider = 20;
    //绘制分组栏背景的画笔
    private Paint mPaint;
    //绘制分组栏上文字的画笔
    private Paint textPaint;
    //接口对象，用于获取省份信息
    private OnProvinceListener onProvinceListener;
    private Province province;
    public StickyHeaderDecoration(OnProvinceListener onProvinceListener){
        this.onProvinceListener = onProvinceListener;
        //初始化2个Paint对象
        mPaint = new Paint(Paint.ANTI_ALIAS_FLAG);
        mPaint.setAntiAlias(true);

        textPaint = new Paint(Paint.ANTI_ALIAS_FLAG);
        textPaint.setAntiAlias(true);
        textPaint.setTextSize(60);
        textPaint.setTypeface(Typeface.SANS_SERIF);
        textPaint.setFakeBoldText(true);
    }
    
    @Override
    public void getItemOffsets(Rect outRect, View view, RecyclerView parent, RecyclerView.State state) {
        super.getItemOffsets(outRect, view, parent, state);
        //获取当前itemView的位置序号
        int index = parent.getChildAdapterPosition(view);
        //通过当前itemView的位置获取它所在分组的信息，如果它是该分组的第一个itemView
        //则在顶部撑开一个分组栏的高度用于绘制分组栏；否则撑开分割线的宽度，表示分割线
        Province province = onProvinceListener.getProvince(index);
        if (province.isFirstCityInProvince){
            outRect.top = headerHeight;
        }else {
            outRect.top = divider;
        }
    }
    
     @Override
    public void onDrawOver(Canvas c, RecyclerView parent, RecyclerView.State state) {
        super.onDrawOver(c, parent, state);
        //获取itemView的总个数
        int childCount = parent.getChildCount();
        //声明一个View对象
        View child;
        //循环遍历所有的itemView，找出每组的第一个itemView，
        //并在其上方撑开一段距离绘制分组栏
        for (int i = 0; i < childCount; i++){
            child = parent.getChildAt(i);
            int position = parent.getChildAdapterPosition(child);
            if(onProvinceListener != null){
                province = onProvinceListener.getProvince(position);
                int left = child.getLeft();
                int right = child.getRight();
                if(i != 0){
                    if(province.isFirstCityInProvince()){
                        int top = child.getTop() - headerHeight;
                        textPaint.setColor(Color.parseColor("#dddddd"));
                        int bottom = child.getTop();
                       	drawHeaderRect(c, province, left, top, right, bottom);
                    }
                }else{
                    //i==0时，整个列表中的第一个itemView,它的分组栏应该紧贴在屏幕最上方
                    int top = parent.getpaddingTop();
                    int bottom = top + headerHeight;
                    drawHeaderRect(c, province, left, top, right, bottom);
                }
            }
        }
    }
    private void drawHeaderRect(Canvas canvas, Province province, int left, int top, int right, int bottom){
        canvas.drawRect(left, top, right, bottom, mPaint);
        //这里+500，+120是为了让文字显示在分组栏中间，实际场景中不要采用这种硬编码方式
        canvas.drawRect(province.getProvinceName(), left + 500, top + 120, textPaint)
    }
}
```

在Activity中，代码如下：

``` java
...
private RecyclerView recyclerView ;
private List<String> mList;
private MyAdapter adapter；
private Province province;
private String[] provinces = {"四川省", "河南省", "江苏省", "山东省", "广东省"};
	@Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_sticky_header);
        recyclerView = findViewById(R.id.recycler_view);
        LinearLayoutManager manager = new LinearLayoutManager(this, LinearLayoutManager.VERTICAL, false);
        recyclerView.setLayoutManager(manager);
        adapter = new MyAdapter(mList, this);
        StickHeaderDecoration decoration = new StickyHeaderDecoration(new OnProvinceListener{
        	@Override
            public Province getProvince(int position){
            //这里我是分成每组12个itemView，实际应用中可按具体需求分组
            	int provinceId = position / 12；
            	int index = position % 12;
            	province = new Province(provinceId, provincesp[provinceId]);
                province.setCityPosition(index);
                return province;
            }
        });
        recyclerView.addItemDecoration(decoration);
    }
```

这样就基本实现了一个带分组的列表界面了，看下运行结果：

![](https://github.com/Aaron-DBJ/MyItemDecoration/tree/master/Images/header_rect.gif)

## 粘性头部分组栏

上面实现已经实现了基本的分组栏效果，之所以说基本，是他还有不完善的地方。比如上面的GIF图所示，下一个分组栏是直接覆盖上一个分组栏，如果我们想实现一个效果是，下面的分组栏将上面的分组栏慢慢推出屏幕，该如何做。我们先来分析一下这个“推出”过程。

![](C:\Users\jenny\Pictures\BlogPictures\last_item.png)

黑色边框代表手机屏幕，当前这个itemView是分组栏1的最后一个itemView，现在继续向上滑动，itemView的底边逐渐趋近于与分组栏1的底边重合，当某个时刻，二者完全重合，而且分组栏1和分组栏2相接，到了一个临界点。现在想要实现“推出”的效果，就要求继续往上滑动时，分组栏1的底边逐渐向上直至完全消失在屏幕，那么绘制的时候，分组栏1的顶边坐标在屏幕之上，这样画一个高度固定的矩形时才能实现“吸顶”的效果。

![](C:\Users\jenny\Pictures\BlogPictures\linjiezhangtai.png)

此时到达临界状态，该分组最后一个itemView的底边和分组栏1的底边重合。如果继续向上滑动，如下图

![](C:\Users\jenny\Pictures\BlogPictures\chaolinjei.png)

此时分组栏1已经一部分超出屏幕范围，所以要重新绘制分组栏1，它的top和bottom值都变化了。所以在`onDrawOver()`方法中，先判断当前itemView是否为该分组最后一个itemView；如果是则修改分组栏的top和bottom值，使之呈现分组栏也慢慢向上划出屏幕并被下一个分组栏顶出取代的效果。有了上述分析，我们来改造一下代码，首先为了判断当前itemView是否为该分组最后一个itemView，在Province类中增加一个成员函数和方法。

``` java
public class Province{
    ...
    private int provinceLength;
    public boolean isLastCityInProvince(){
        if (citiPosition >= 0 && cityPosition == provinceLength - 1){
            reture true;
        }
        reture false;
    }
    ...
}
```

然后在`onDrawOver（）`方法中修改代码

``` java
public class StickyHeaderDecoration extends RecyclerView.ItemDecoration{
    //定义分组栏的高度
    private static final int headerHeight = 200;
    //定义分割线的宽度
    private static final int divider = 20;
    //绘制分组栏背景的画笔
    private Paint mPaint;
    //绘制分组栏上文字的画笔
    private Paint textPaint;
    //接口对象，用于获取省份信息
    private OnProvinceListener onProvinceListener;
    private Province province;
    public StickyHeaderDecoration(OnProvinceListener onProvinceListener){
        this.onProvinceListener = onProvinceListener;
        //初始化2个Paint对象
        mPaint = new Paint(Paint.ANTI_ALIAS_FLAG);
        mPaint.setAntiAlias(true);

        textPaint = new Paint(Paint.ANTI_ALIAS_FLAG);
        textPaint.setAntiAlias(true);
        textPaint.setTextSize(60);
        textPaint.setTypeface(Typeface.SANS_SERIF);
        textPaint.setFakeBoldText(true);
    }
    
    @Override
    public void getItemOffsets(Rect outRect, View view, RecyclerView parent, RecyclerView.State state) {
        super.getItemOffsets(outRect, view, parent, state);
        //获取当前itemView的位置序号
        int index = parent.getChildAdapterPosition(view);
        //通过当前itemView的位置获取它所在分组的信息，如果它是该分组的第一个itemView
        //则在顶部撑开一个分组栏的高度用于绘制分组栏；否则撑开分割线的宽度，表示分割线
        Province province = onProvinceListener.getProvince(index);
        if (province.isFirstCityInProvince){
            outRect.top = headerHeight;
        }else {
            outRect.top = divider;
        }
    }
    
     @Override
    public void onDrawOver(Canvas c, RecyclerView parent, RecyclerView.State state) {
        super.onDrawOver(c, parent, state);
        //获取itemView的总个数
        int childCount = parent.getChildCount();
        //声明一个View对象
        View child;
        //循环遍历所有的itemView，找出每组的第一个itemView，
        //并在其上方撑开一段距离绘制分组栏
        for (int i = 0; i < childCount; i++){
            child = parent.getChildAt(i);
            int position = parent.getChildAdapterPosition(child);
            if(onProvinceListener != null){
                province = onProvinceListener.getProvince(position);
                int left = child.getLeft();
                int right = child.getRight();
                if(i != 0){
                    if(province.isFirstCityInProvince()){
                        int top = child.getTop() - headerHeight;
                        textPaint.setColor(Color.parseColor("#dddddd"));
                        int bottom = child.getTop();
                       	drawHeaderRect(c, province, left, top, right, bottom);
                    }
                }else{
                    //i==0时，整个列表中的第一个itemView,它的分组栏应该紧贴在屏幕最上方
                    int top = parent.getpaddingTop();
                    //这里加入了一个判断，在上推过程中改变分组栏的top和bottom值
                    if(pronvince.isLastCityInProvnice()){
                        int supposedTop = child.getBootom()-headerHeight;
                        if(supposedTop < top){
                            top = supposedTop;
                        }
                    }
                    int bottom = top + headerHeight;
                    drawHeaderRect(c, province, left, top, right, bottom);
                }
            }
        }
    }
...
}
```

运行看一下效果：

![](C:\Users\jenny\Pictures\BlogPictures\styiky_header.gif)

具体请看：[完整代码](https://github.com/Aaron-DBJ/MyItemDecoration)
