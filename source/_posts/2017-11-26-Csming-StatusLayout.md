---
title: StatusLayout
subtitle: 显示不同状态的布局；在一个应用中，通常都会有数据为空、网络断开、正在加载等一系列状态；于是我做了一个控件来统一管理这些状态；一来方便使用，二来方便风格统一；
date: 2017-11-26 14:58:50
author: "Csming"
catalog: true

tags:
     - 14级
     - Android
---

# StatusLayout

首先附上github项目地址；https://github.com/csming1995/statuslayout

之前看过很多网上已有的做法，大多都已经将状态都涵盖了；这样的做法，可能很难包裹所有的业务需求；

于是，突发奇想，是否能够提供给使用者更自由的使用方式；比如，提供给使用者自定义某状态布局，甚至自定义状态及布局的自由；

***

这是一个复杂度不太高，但是代码设计感比较强一点的开源库~；

先看一下源码；

```java
public class StatusLayout extends FrameLayout{
    private static final String TAG = "StatusLayout.FrameLayout";


    /**
     * DEFAULT EMPTY NET_ERROR 默认的三种状态
     * DEFAULT 为用户第一次使用该组件时指定的属性状态
     */
    private static final int DEFAULT = 1;
    private static final int EMPTY = 2;
    private static final int NET_ERROR = 3;
    //rivate static final int LOADING = 3;


    /**
     * 属性值
     */
    private String mInitMessage;
    private Drawable mInitImage;
    private String mInitStrInBtn;

    /**
     * Map 用键值对存储 状态-视图
     * List 用于存储子控件,即内容
     */
    private Map<Integer, View> mMapMessageViews;
    private List<View> mNormalViews;

    private LayoutInflater mLayoutInflater;


    /**
     * 默认页
     * 空数据页
     * 网络错误页
     */
    private LinearLayout mDefaultView;//默认页
    private LinearLayout mDefaultEmptyMessageView;
    private LinearLayout mDefaultNetErrorView;

    private Context mContext;

    public StatusLayout(Context context){
        this(context, null);
    }

    public StatusLayout(Context context, AttributeSet attrs){
        this(context, attrs, 0);
    }

    public StatusLayout(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        mContext = context;
        init(attrs);
    }


    /**
     * 一些初始化工作
     * 初始化DefaultView
     */

    private void init(AttributeSet attrs){
        if (null == mNormalViews) mNormalViews = new ArrayList<>();

        if (null == mMapMessageViews) mMapMessageViews = new HashMap<>();

        if (null == mLayoutInflater){
            mLayoutInflater = (LayoutInflater)mContext.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
        }

        TypedArray mValueArray = mContext.obtainStyledAttributes(attrs, R.styleable.StatusLayoutValue);

        mInitMessage = mValueArray.getString(R.styleable.StatusLayoutValue_attr_message);
        mInitImage = mValueArray.getDrawable(R.styleable.StatusLayoutValue_attr_image_src);
        mInitStrInBtn = mValueArray.getString(R.styleable.StatusLayoutValue_attr_str_btn);

        setEmptyMessageView();
        setNetErrorMessageView();
        setDefaultView(mInitMessage, mInitImage, mInitStrInBtn);
        mValueArray.recycle();

    }

    /**
     * 加载完布局后 使默认视图显示
     */
    @Override
    protected void onFinishInflate(){
        super.onFinishInflate();
        showDefaultView();
    }

    /**
     * 测量
     * @param widthMeasureSpec
     * @param heightMeasureSpec
     */
    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);
        measureChildren(widthMeasureSpec, heightMeasureSpec);
    }

    /**
     * 通过addView函数在被调用时,对child View进行初始化
     * 获取子控件信息
     * @param child
     * @param params
     */
    @Override
    public void addView(View child, ViewGroup.LayoutParams params) {
        super.addView(child, params);
        for(int i = 0; i < getChildCount(); i ++){
            mNormalViews.add(getChildAt(i));
        }
    }

    public void showDefaultView(){
        showStatusView(DEFAULT);
    }

    public void showEmptyMessageView(){
        showStatusView(EMPTY);
    }

    public void showNetErrorView(){
        showStatusView(NET_ERROR);
    }

    /**
     * 设置为有数据状态
     * 使当前View的子View显示
     * 子View为RecyclerView
     * @see #setContentView(boolean)
     */
    public void showNormalView(){
        hiddenStatusViews();
        setContentView(true);
    }

    /**
     * 设置子View的显示或隐藏状态
     * 子View存储于一个list中
     * @param isShown
     */
    private void setContentView(boolean isShown){
        if (isShown){
            for (View v : mNormalViews){
                v.setVisibility(VISIBLE);
            }
        }else {
            for (View v : mNormalViews){
                v.setVisibility(GONE);
            }
        }
    }

    /**
     * 无参调用的设置网络错误页
     * 用于内部调用
     */
    private void setNetErrorMessageView(){
        if (null == mDefaultNetErrorView){
            mDefaultNetErrorView = (LinearLayout)mLayoutInflater.inflate(R.layout.status_layout_net_error_message, null);
        }
        mMapMessageViews.put(NET_ERROR, mDefaultNetErrorView);
    }

    /**
     * 有参调用的设置网络错误页
     * 提供给外部使用者
     * @param message
     * @param image
     */
    public void setNetErrorMessageView(String message, Drawable image){
        if(null == mDefaultNetErrorView){
            mDefaultNetErrorView = (LinearLayout)mLayoutInflater.inflate(R.layout.status_layout_net_error_message, null);
        }else {
            mDefaultNetErrorView = (LinearLayout)mMapMessageViews.get(NET_ERROR);
        }

        TextView mTvNetError = (TextView)mDefaultNetErrorView.findViewById(R.id.tv_net_error_view);
        ImageView mIvNetError = (ImageView)mDefaultNetErrorView.findViewById(R.id.iv_net_error_view);
        if (null != message){
            mTvNetError.setText(message);
            mTvNetError.setVisibility(VISIBLE);
        }else {
            mTvNetError.setVisibility(GONE);
        }
        if (null != image){
            mIvNetError.setImageDrawable(image);
            mIvNetError.setVisibility(VISIBLE);
        }else {
            mIvNetError.setVisibility(GONE);
        }

        mMapMessageViews.put(NET_ERROR, mDefaultNetErrorView);
    }

    /**
     * 无参调用空数据页面
     * 用于内部调用
     */

    private void setEmptyMessageView(){
        if (null == mDefaultEmptyMessageView) {
            mDefaultEmptyMessageView = (LinearLayout)mLayoutInflater.inflate(R.layout.status_layout_empty_message, null);
        }
        mMapMessageViews.put(EMPTY, mDefaultEmptyMessageView);
    }

    public void setEmptyMessageView(int messageId, int imageId, int messageInBtnId){
        String messageInBtn = mContext.getString(messageInBtnId);
        setEmptyMessageView(messageId, imageId, messageInBtn);
    }

    public void setEmptyMessageView(int messageId, int imageId, String messageInBtn){
        Drawable image = ContextCompat.getDrawable(mContext, imageId);
        setEmptyMessageView(messageId, image, messageInBtn);
    }

    public void setEmptyMessageView(int messageId, Drawable image, String messageInBtn){
        String message = mContext.getString(messageId);
        setEmptyMessageView(message, image, messageInBtn);
    }

    /**
     * 有参调用设置空数据页
     * 提供给外部使用者
     * @param message
     * @param image
     * @param messageInBtn
     */

    public void setEmptyMessageView(String message, Drawable image, String messageInBtn){
        if(null == mDefaultEmptyMessageView){
            mDefaultEmptyMessageView = (LinearLayout)mLayoutInflater.inflate(R.layout.status_layout_empty_message, null);
        }else {
            mDefaultEmptyMessageView = (LinearLayout)mMapMessageViews.get(EMPTY);
        }

        TextView mTvEmpty = (TextView)mDefaultEmptyMessageView.findViewById(R.id.tv_empty_view);
        ImageView mIvEmpty = (ImageView)mDefaultEmptyMessageView.findViewById(R.id.iv_empty_view);
        Button mBtnEmpty = (Button)mDefaultEmptyMessageView.findViewById(R.id.btn_empty_view);
        if (null != message){
            mTvEmpty.setText(message);
            mTvEmpty.setVisibility(VISIBLE);
        }else {
            mTvEmpty.setVisibility(GONE);
        }
        if (null != image){
            mIvEmpty.setImageDrawable(image);
            mIvEmpty.setVisibility(VISIBLE);
        }else {
            mIvEmpty.setVisibility(GONE);
        }
        if (null != messageInBtn) {
            mBtnEmpty.setText(message);
            mBtnEmpty.setVisibility(VISIBLE);
        }else {
            mBtnEmpty.setVisibility(GONE);
        }
        mMapMessageViews.put(EMPTY, mDefaultEmptyMessageView);
    }
    /**
     * 有参调用 设置默认页
     * 用于内部使用
     * @param message
     * @param image
     * @param messageInBtn
     */
    private void setDefaultView(String message, Drawable image, String messageInBtn){
        if(null == mDefaultView){
            mDefaultView = (LinearLayout)mLayoutInflater.inflate(R.layout.status_layout_default_message, null);
        }else {
            mDefaultView = (LinearLayout)mMapMessageViews.get(DEFAULT);
        }

        TextView mTvDefault = (TextView)mDefaultView.findViewById(R.id.tv_default_view);
        ImageView mIvDefault = (ImageView)mDefaultView.findViewById(R.id.iv_default_view);
        Button mBtnDefault = (Button)mDefaultView.findViewById(R.id.btn_default_view);
        if (null != message){
            mTvDefault.setText(message);
            mTvDefault.setVisibility(VISIBLE);
        }else {
            mTvDefault.setVisibility(GONE);
        }
        if (null != image){
            mIvDefault.setImageDrawable(image);
            mIvDefault.setVisibility(VISIBLE);
        }else {
            mIvDefault.setVisibility(GONE);
        }
        if (null != messageInBtn) {
            mBtnDefault.setText(message);
            mBtnDefault.setVisibility(VISIBLE);
        }else {
            mBtnDefault.setVisibility(GONE);
        }
        mMapMessageViews.put(DEFAULT, mDefaultView);
    }

    /**
     * 外部添加状态
     * 若状态与已有状态碰撞
     * 跳出错误
     * @param key
     * @param view
     * @throws IllegalNumException
     */
    public void addStatus(int key, View view) throws IllegalNumException {
        if(1 == key||2 == key||3 == key) {
            throw new IllegalNumException();
        }
        mMapMessageViews.put(key, view);
    }

    /**
     * 显示指定状态页
     * 并将其他页面隐藏
     * 用于内部以及外部电泳
     * @param key
     */
    public void showStatusView(int key){
        setContentView(false);
        View mMessageView = mMapMessageViews.get(key);
        hiddenStatusViews();
        addView(mMessageView);
        mMessageView.setVisibility(VISIBLE);
    }

    /**
     * 隐藏mMapMessageViews的所有页面
     */

    private void hiddenStatusViews(){
        for (View v : mMapMessageViews.values()){
            removeView(v);
        }
    }

}

```

* **首先是：** 这三个方法初始化了三种基本布局；这三个方法用于定义了每一种布局的默认状态下的文字及图片；首先在初始化的时候调用；
* 他们最终是将初始化后的布局，加入mMapMessageViews中保存；mMapMessageViews的键值对为：状态-布局；我们后面在显示的时候，将会从这个map中，通过状态key，获取对应的布局；

```java
public void setEmptyMessageView();
public void setEmptyMessageView(int messageId, int imageId, int messageInBtnId);
public void setEmptyMessageView(int messageId, int imageId, String messageInBtn);
public void setEmptyMessageView(int messageId, Drawable image, String messageInBtn);
public void setEmptyMessageView(String message, Drawable image, String messageInBtn);

public void setNetErrorMessageView();
public void setNetErrorMessageView(String message, Drawable image);

private void setDefaultView(String message, Drawable image, String messageInBtn);
```

* **然后：** 在onFinishInflate()布局加载完成后，先显示defaultView

```java
    @Override
    protected void onFinishInflate(){
        super.onFinishInflate();
        showDefaultView();
    }
```

* showDefaultView()方法，和其他的showXxx()方法一样:

```java
public void showDefaultView(){
        showStatusView(DEFAULT);
    }
```

最终调用的是showStatusView()这个方法；而showStatusView()方法，传入一个key，然后从mMapMessageViews中获取对应的布局，并隐藏其他布局，最后显示当前布局；

```java
public void showStatusView(int key){
        setContentView(false);
        View mMessageView = mMapMessageViews.get(key);
        hiddenStatusViews();
        addView(mMessageView);
        mMessageView.setVisibility(VISIBLE);
    }
```

然后这里有一个setContentView()函数，他的意义在于，控制子布局的显示与隐藏；

因为，我们的布局，在有数据状态下，应该显示的是其子布局的内容；

例如：一个RecyclerView；

```xml
<com.csm.Component.StatusLayout
        android:id="@+id/statuslayout_demo"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        app:attr_message="@string/str_there_has_nothing"
        app:attr_image_src="@mipmap/ic_launcher">
        <android.support.v7.widget.RecyclerView
            android:id="@+id/rv_demo"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:src="@mipmap/ic_launcher_round"/>
    </com.csm.Component.StatusLayout>
```

那么调用setContentView()，就可以控制其子布局的显示与隐藏；主要在于遍历布局下的所有子布局，然后设置他们的显示隐藏；

```java
 /**
     * 设置子View的显示或隐藏状态
     * 子View存储于一个list中
     * @param isShown
     */
    private void setContentView(boolean isShown){
        if (isShown){
            for (View v : mNormalViews){
                v.setVisibility(VISIBLE);
            }
        }else {
            for (View v : mNormalViews){
                v.setVisibility(GONE);
            }
        }
    }
```

以上就是布局内容的显示部分；

***

然后，关键的，如何提供给使用者自定义状态及对应布局的逻辑，主要是维护了一个map,以及几种状态值；

```java
private Map<Integer, View> mMapMessageViews;

/**
     * DEFAULT EMPTY NET_ERROR 默认的三种状态
     * DEFAULT 为用户第一次使用该组件时指定的属性状态
     */
    private static final int DEFAULT = 1;
    private static final int EMPTY = 2;
    private static final int NET_ERROR = 3;
```

以上三种是默认值；

如果使用者需要自定义状态及布局，则只能定义除了这三个数字以外的数字；

为此，我特意编写了一个Exception类型：如果使用者自定义的key是1/2/3的话，则抛出错误；

```java

/**
 * Created by csm on 2017/7/7.
 */

public class IllegalNumException extends Exception {
    public IllegalNumException(){}

    public IllegalNumException(String gripe){
        super(gripe);
    }

    @Override
    public void printStackTrace(){
        super.printStackTrace();
        System.out.print("You can't choice 1,2,3 as your status key");
    }
}
```

***

该开源库已经上传到github上了；

https://github.com/csming1995/statuslayout

各种求star；
