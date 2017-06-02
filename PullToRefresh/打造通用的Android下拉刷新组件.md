> 原文链接：http://blog.csdn.net/bboyfeiyu/article/details/39718861

[打造通用的Android下拉刷新组件(适用于ListView、GridView等各类View)](http://blog.csdn.net/bboyfeiyu/article/details/39718861)

## 前言

最近在做项目时，使用了一个开源的下拉刷新ListView组件，极其的不稳定，bug还多。稳定的组件又写得太复杂了，jar包较大。在我的一篇博客中也讲述过下拉刷新的实现，即Android打造(ListView、GridView等)通用的下拉刷新、上拉自动加载的组件。但是这种通过修改Margin的形式感觉不是特别的流畅，因此在这漫长的国庆长假又花了点时间用另外的原理实现了一遍，特此分享出来。

## 基本原理

原理就是自定义一个ViewGroup，将Header View, Content View, Footer View从上到下依次布局，如图1 (红色区域为屏幕的显示区域)。在初始时通过滚动，使得该组件在Y轴方向上滚动HeaderView的高度的距离，这样HeaderView就被隐藏掉了，如图2。而Content View的宽度和高度都是match_parent的，因此此时屏幕上只显示Content View， HeaderView 和 FooterView都被隐藏在屏幕外了。当组件被滚动到顶端时，如果用户继续下拉，那么拦截触摸事件，然后通过Scroller来滚动y轴的偏移量，实现逐步的显示HeaderView，从而到达下拉的效果，如图3。当用户滑动到最底部时会触发加载更多的操作。

|              图 1 (红色区域为屏幕)               |              图2  (红色区域为屏幕)               |               图 3(红色区域为屏幕)               |
| :--------------------------------------: | :--------------------------------------: | :--------------------------------------: |
| <img src="http://img.blog.csdn.net/20141001174054122?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYmJveWZlaXl1/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center" width="300" /> | <img src="http://img.blog.csdn.net/20141001174058833?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYmJveWZlaXl1/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center" width="300" /> | <img src="http://img.blog.csdn.net/20141001174104496?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYmJveWZlaXl1/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center" width="300" /> |

通过使用Scroller使得整个滚动更加的平滑，而使用Margin来实现的话需要自己来计算滚动时间和margin值，并不是很流畅，而且频繁的修改布局参数效率也不高。使用Scroller只是滚动位置，而没有修改布局参数，因此有点较为突出。

## Scroller的使用
为了更好的理解下拉刷的实现，我们先要了解Scroller的作用以及如何使用。这里我们将做一个简单的示例来说明。

Scroller是一个帮助View滚动的辅助类，在使用它之前用户需要通过startScroll来设置滚动的参数，即起始点坐标和x,y轴上要滚动的距离。Scroller它封装了滚动时间、要滚动的目标x轴和y轴，以及在每个时间内view应该滚动到的x,y轴的坐标点，这样用户就可以在有效的滚动周期内通过Scroller的getCurX()和getCurY()来获取当前时刻View应该滚动的位置，然后通过调用View的scrollTo或者ScrollBy方法进行滚动。那么如何判断滚动是否结束呢 ？ 我们只需要覆写View类的computeScroll方法，该方法会在View绘制的时候被调用，在里面调用Scroller的computeScrollOffset来判断滚动是否完成，如果返回true表明滚动未完成，否则滚动完成。上述说的scrollTo或者ScrollBy的调用就是在computeScrollOffset为true的情况下调用，并且最后还要调用目标view的postInvalidate()或者invalidate()以实现View的重绘。View的重绘又会导致computeScroll方法被调用，从而继续整个滚动过程，直至computeScrollOffset返回false， 即滚动结束。整个过程有点绕，我们看一个例子吧。
```java
public class ScrollLayout extends FrameLayout {  
  
    private String TAG = ScrollLayout.class.getSimpleName();  
  
  
    Scroller mScroller ;  
  
  
    public ScrollLayout(Context context) {  
        super(context);  
          
        mScroller = new Scroller(context) ;  
    }  
  
      
    // 该函数会在View重绘之时被调用  
    @Override  
    public void computeScroll() {  
        if ( mScroller.computeScrollOffset() ) {  
            // 滚动到此刻View应该滚动到的x,y坐标上.  
            this.scrollTo(mScroller.getCurrX(), mScroller.getCurrY());  
            // 请求重绘该View，从而又会导致computeScroll被调用，然后继续滚动，直到computeScrollOffset返回false  
            this.postInvalidate();  
        }  
    }  
  
  
    // 调用这个方法进行滚动，这里我们只滚动竖直方向，  
    public void scrollTo(int y) {  
            // 参数1和参数2分别为滚动的起始点水平、竖直方向的滚动偏移量  
            // 参数3和参数4为在水平和竖直方向上滚动的距离  
            mScroller.startScroll(getScrollX(), getScrollY(), 0, y);  
            this.invalidate();  
    }  
} 
```
滚动该视图的代码 : 
```java
ScrollLayout scrollView = new ScrollLayout(getContext()) ;  
scrollView.scrollTo(100);  
```
通过上面这段代码会让scrollView在y轴上向下滚动100个像素点。我们结合代码来分析一下。首先调用scrollTo(int y)方法，然后我们在该方法中通过mScroller.startScroll()方法来设置了滚动的参数，然后调用invalidate()方法使得该View重绘。重绘时会调用computeScroll方法，在该方法中通过mScroller.computeScrollOffset()判断滚动是否完成，如果返回true那代表没有滚动完成，此时把该View滚动到此刻View应该滚动到的x, y位置，这个位置通过mScroller的getCurX, getCurY获得。然后继续调用重绘方法，继续执行滚动过程，直至滚动完成。

了解了Scroller原理后，我们继续看通用的下拉刷新组件的实现吧。

## 下拉刷新实现
代码量不算多，但是也挺长的，我们这里只拿出重要的点来分析，完成的源码在博文最后会给出。以下是重要的代码段 : 
```java
/** 
 * @author mrsimple 
 */  
public abstract class RefreshLayoutBase<T extends View> extends ViewGroup implements  
        OnScrollListener {  
  
    /** 
     *  
     */  
    protected Scroller mScroller;  
  
    /** 
     * 下拉刷新时显示的header view 
     */  
    protected View mHeaderView;  
  
    /** 
     * 上拉加载更多时显示的footer view 
     */  
    protected View mFooterView;  
  
    /** 
     * 本次触摸滑动y坐标上的偏移量 
     */  
    protected int mYOffset;  
  
    /** 
     * 内容视图, 即用户触摸导致下拉刷新、上拉加载的主视图. 比如ListView, GridView等. 
     */  
    protected T mContentView;  
  
    /** 
     * 最初的滚动位置.第一次布局时滚动header的高度的距离 
     */  
    protected int mInitScrollY = 0;  
    /** 
     * 最后一次触摸事件的y轴坐标 
     */  
    protected int mLastY = 0;  
  
    /** 
     * 空闲状态 
     */  
    public static final int STATUS_IDLE = 0;  
  
    /** 
     * 下拉或者上拉状态, 还没有到达可刷新的状态 
     */  
    public static final int STATUS_PULL_TO_REFRESH = 1;  
  
    /** 
     * 下拉或者上拉状态 
     */  
    public static final int STATUS_RELEASE_TO_REFRESH = 2;  
    /** 
     * 刷新中 
     */  
    public static final int STATUS_REFRESHING = 3;  
  
    /** 
     * LOADING中 
     */  
    public static final int STATUS_LOADING = 4;  
  
    /** 
     * 当前状态 
     */  
    protected int mCurrentStatus = STATUS_IDLE;  
  
    /** 
     * 下拉刷新监听器 
     */  
    protected OnRefreshListener mOnRefreshListener;  
  
    /** 
     * header中的箭头图标 
     */  
    private ImageView mArrowImageView;  
    /** 
     * 箭头是否向上 
     */  
    private boolean isArrowUp;  
    /** 
     * header 中的文本标签 
     */  
    private TextView mTipsTextView;  
    /** 
     * header中的时间标签 
     */  
    private TextView mTimeTextView;  
    /** 
     * header中的进度条 
     */  
    private ProgressBar mProgressBar;  
    /** 
     *  
     */  
    private int mScreenHeight;  
    /** 
     *  
     */  
    private int mHeaderHeight;  
    /** 
     *  
     */  
    protected OnLoadListener mLoadListener;  
  
    /** 
     * @param context 
     */  
    public RefreshLayoutBase(Context context) {  
        this(context, null);  
    }  
  
    /** 
     * @param context 
     * @param attrs 
     */  
    public RefreshLayoutBase(Context context, AttributeSet attrs) {  
        this(context, attrs, 0);  
    }  
  
    /** 
     * @param context 
     * @param attrs 
     * @param defStyle 
     */  
    public RefreshLayoutBase(Context context, AttributeSet attrs, int defStyle) {  
        super(context, attrs);  
  
        // 初始化Scroller对象  
        mScroller = new Scroller(context);  
  
        // 获取屏幕高度  
        mScreenHeight = context.getResources().getDisplayMetrics().heightPixels;  
        // header 的高度为屏幕高度的 1/4  
        mHeaderHeight = mScreenHeight / 4;  
  
        // 初始化整个布局  
        initLayout(context);  
    }  
  
    /** 
     * 初始化整个布局 
     *  
     * @param context 
     */  
    private final void initLayout(Context context) {  
  
        // header view  
        setupHeaderView(context);  
  
        // 设置内容视图  
        setupContentView(context);  
        // 设置布局参数  
        setDefaultContentLayoutParams();  
        //  
        addView(mContentView);  
  
        // footer view  
        setupFooterView(context);  
  
    }  
  
    /** 
     * 初始化 header view 
     */  
    protected void setupHeaderView(Context context) {  
        mHeaderView = LayoutInflater.from(context).inflate(R.layout.pull_to_refresh_header, this,  
                false);  
        mHeaderView  
                .setLayoutParams(new ViewGroup.LayoutParams(LayoutParams.MATCH_PARENT,  
                        mHeaderHeight));  
        mHeaderView.setBackgroundColor(Color.RED);  
        // header的高度整个为1/4的屏幕高度，但是它只有100px是有效的显示区域，取余取余为paddingTop，这样是为了达到下拉的效果  
        mHeaderView.setPadding(0, mHeaderHeight - 100, 0, 0);  
        addView(mHeaderView);  
  
        // HEADER VIEWS  
        mArrowImageView = (ImageView) mHeaderView.findViewById(R.id.pull_to_arrow_image);  
        mTipsTextView = (TextView) mHeaderView.findViewById(R.id.pull_to_refresh_text);  
        mTimeTextView = (TextView) mHeaderView.findViewById(R.id.pull_to_refresh_updated_at);  
        mProgressBar = (ProgressBar) mHeaderView.findViewById(R.id.pull_to_refresh_progress);  
    }  
  
    /** 
     * 初始化Content View, 子类覆写. 
     */  
    protected abstract void setupContentView(Context context);  
  
  
    /** 
     * 与Scroller合作,实现平滑滚动。在该方法中调用Scroller的computeScrollOffset来判断滚动是否结束。如果没有结束， 
     * 那么滚动到相应的位置，并且调用postInvalidate方法重绘界面，从而再次进入到这个computeScroll流程，直到滚动结束。 
     */  
    @Override  
    public void computeScroll() {  
        if (mScroller.computeScrollOffset()) {  
            scrollTo(mScroller.getCurrX(), mScroller.getCurrY());  
            postInvalidate();  
        }  
    }  
  
    /* 
     * 在适当的时候拦截触摸事件，这里指的适当的时候是当mContentView滑动到顶部，并且是下拉时拦截触摸事件，否则不拦截，交给其child 
     * view 来处理。 
     * @see 
     * android.view.ViewGroup#onInterceptTouchEvent(android.view.MotionEvent) 
     */  
    @Override  
    public boolean onInterceptTouchEvent(MotionEvent ev) {  
  
        /* 
         * This method JUST determines whether we want to intercept the motion. 
         * If we return true, onTouchEvent will be called and we do the actual 
         * scrolling there. 
         */  
        final int action = MotionEventCompat.getActionMasked(ev);  
        // Always handle the case of the touch gesture being complete.  
        if (action == MotionEvent.ACTION_CANCEL || action == MotionEvent.ACTION_UP) {  
            // Do not intercept touch event, let the child handle it  
            return false;  
        }  
  
        switch (action) {  
  
            case MotionEvent.ACTION_DOWN:  
                mLastY = (int) ev.getRawY();  
                break;  
  
            case MotionEvent.ACTION_MOVE:  
                // int yDistance = (int) ev.getRawY() - mYDown;  
                mYOffset = (int) ev.getRawY() - mLastY;  
                // 如果拉到了顶部, 并且是下拉,则拦截触摸事件,从而转到onTouchEvent来处理下拉刷新事件  
                if (isTop() && mYOffset > 0) {  
                    return true;  
                }  
                break;  
  
        }  
  
        // Do not intercept touch event, let the child handle it  
        return false;  
    }  
  
    /** 
     * 是否已经到了最顶部,子类需覆写该方法,使得mContentView滑动到最顶端时返回true, 如果到达最顶端用户继续下拉则拦截事件; 
     *  
     * @return 
     */  
    protected abstract boolean isTop();  
  
    /** 
     * 是否已经到了最底部,子类需覆写该方法,使得mContentView滑动到最底端时返回true;从而触发自动加载更多的操作 
     *  
     * @return 
     */  
    protected abstract boolean isBottom();  
  
    /** 
     * 显示footer view 
     */  
    private void showFooterView() {  
        startScroll(mFooterView.getMeasuredHeight());  
        mCurrentStatus = STATUS_LOADING;  
    }  
  
    /** 
     * 设置滚动的参数 
     *  
     * @param yOffset 
     */  
    private void startScroll(int yOffset) {  
        mScroller.startScroll(getScrollX(), getScrollY(), 0, yOffset);  
        invalidate();  
    }  
  
    /* 
     * 在这里处理触摸事件以达到下拉刷新或者上拉自动加载的问题 
     * @see android.view.View#onTouchEvent(android.view.MotionEvent) 
     */  
    @Override  
    public boolean onTouchEvent(MotionEvent event) {  
  
        Log.d(VIEW_LOG_TAG, "@@@ onTouchEvent : action = " + event.getAction());  
        switch (event.getAction()) {  
            case MotionEvent.ACTION_DOWN:  
                mLastY = (int) event.getRawY();  
                break;  
  
            case MotionEvent.ACTION_MOVE:  
                int currentY = (int) event.getRawY();  
                mYOffset = currentY - mLastY;  
                if (mCurrentStatus != STATUS_LOADING) {  
                    //  
                    changeScrollY(mYOffset);  
                }  
  
                rotateHeaderArrow();  
                changeTips();  
                mLastY = currentY;  
                break;  
  
            case MotionEvent.ACTION_UP:  
                // 下拉刷新的具体操作  
                doRefresh();  
                break;  
            default:  
                break;  
  
        }  
  
        return true;  
    }  
  
    /** 
     * 修改y轴上的滚动值，从而实现header被下拉的效果 
     * @param distance 
     * @return 
     */  
    private void changeScrollY(int distance) {  
        // 最大值为 scrollY(header 隐藏), 最小值为0 ( header 完全显示).  
        int curY = getScrollY();  
        // 下拉  
        if (distance > 0 && curY - distance > getPaddingTop()) {  
            scrollBy(0, -distance);  
        } else if (distance < 0 && curY - distance <= mInitScrollY) {  
            // 上拉过程  
            scrollBy(0, -distance);  
        }  
  
        curY = getScrollY();  
        int slop = mInitScrollY / 2;  
        //  
        if (curY > 0 && curY < slop) {  
            mCurrentStatus = STATUS_RELEASE_TO_REFRESH;  
        } else if (curY > 0 && curY > slop) {  
            mCurrentStatus = STATUS_PULL_TO_REFRESH;  
        }  
    }  
  
  
  
    /** 
     * 刷新结束，恢复状态 
     */  
    public void refreshComplete() {  
        mCurrentStatus = STATUS_IDLE;  
        // 隐藏header view  
        mScroller.startScroll(getScrollX(), getScrollY(), 0, mInitScrollY - getScrollY());  
        invalidate();  
        updateHeaderTimeStamp();  
  
        // 200毫秒后处理arrow和progressbar,免得太突兀  
        this.postDelayed(new Runnable() {  
  
            @Override  
            public void run() {  
                mArrowImageView.setVisibility(View.VISIBLE);  
                mProgressBar.setVisibility(View.GONE);  
            }  
        }, 100);  
  
    }  
  
    /** 
     * 加载结束，恢复状态 
     */  
    public void loadCompelte() {  
        // 隐藏footer  
        startScroll(mInitScrollY - getScrollY());  
        mCurrentStatus = STATUS_IDLE;  
    }  
  
    /** 
     * 手指抬起时,根据用户下拉的高度来判断是否是有效的下拉刷新操作。如果下拉的距离超过header view的 
     * 1/2那么则认为是有效的下拉刷新操作，否则恢复原来的视图状态. 
     */  
    private void changeHeaderViewStaus() {  
        int curScrollY = getScrollY();  
        // 超过1/2则认为是有效的下拉刷新, 否则还原  
        if (curScrollY < mInitScrollY / 2) {  
            // 滚动到能够正常显示header的位置  
            mScroller.startScroll(getScrollX(), curScrollY, 0, mHeaderView.getPaddingTop()  
                    - curScrollY);  
            mCurrentStatus = STATUS_REFRESHING;  
            mTipsTextView.setText(R.string.pull_to_refresh_refreshing_label);  
            mArrowImageView.clearAnimation();  
            mArrowImageView.setVisibility(View.GONE);  
            mProgressBar.setVisibility(View.VISIBLE);  
        } else {  
            mScroller.startScroll(getScrollX(), curScrollY, 0, mInitScrollY - curScrollY);  
            mCurrentStatus = STATUS_IDLE;  
        }  
  
        invalidate();  
    }  
  
    /** 
     * 执行下拉刷新 
     */  
    private void doRefresh() {  
        changeHeaderViewStaus();  
        // 执行刷新操作  
        if (mCurrentStatus == STATUS_REFRESHING && mOnRefreshListener != null) {  
            mOnRefreshListener.onRefresh();  
        }  
    }  
  
    /** 
     * 执行下拉(自动)加载更多的操作 
     */  
    private void doLoadMore() {  
        if (mLoadListener != null) {  
            mLoadListener.onLoadMore();  
        }  
    }  
  
  
    /* 
     * 丈量视图的宽、高。宽度为用户设置的宽度，高度则为header, content view, footer这三个子控件的高度只和。 
     * @see android.view.View#onMeasure(int, int) 
     */  
    @Override  
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {  
  
        int width = MeasureSpec.getSize(widthMeasureSpec);  
  
        int childCount = getChildCount();  
  
        int finalHeight = 0;  
  
        for (int i = 0; i < childCount; i++) {  
            View child = getChildAt(i);  
            // measure  
            measureChild(child, widthMeasureSpec, heightMeasureSpec);  
            // 该view所需要的总高度  
            finalHeight += child.getMeasuredHeight();  
        }  
  
        setMeasuredDimension(width, finalHeight);  
    }  
  
    /* 
     * 布局函数，将header, content view, 
     * footer这三个view从上到下布局。布局完成后通过Scroller滚动到header的底部，即滚动距离为header的高度 + 
     * 本视图的paddingTop，从而达到隐藏header的效果. 
     * @see android.view.ViewGroup#onLayout(boolean, int, int, int, int) 
     */  
    @Override  
    protected void onLayout(boolean changed, int l, int t, int r, int b) {  
  
        int childCount = getChildCount();  
        int top = getPaddingTop();  
        for (int i = 0; i < childCount; i++) {  
            View child = getChildAt(i);  
            child.layout(0, top, child.getMeasuredWidth(), child.getMeasuredHeight() + top);  
            top += child.getMeasuredHeight();  
        }  
  
        // 计算初始化滑动的y轴距离  
        mInitScrollY = mHeaderView.getMeasuredHeight() + getPaddingTop();  
        // 滑动到header view高度的位置, 从而达到隐藏header view的效果  
        scrollTo(0, mInitScrollY);  
    }  
  
  
    /* 
     * 滚动监听，当滚动到最底部，且用户设置了加载更多的监听器时触发加载更多操作. 
     * @see android.widget.AbsListView.OnScrollListener#onScroll(android.widget. 
     * AbsListView, int, int, int) 
     */  
    @Override  
    public void onScroll(AbsListView view, int firstVisibleItem, int visibleItemCount,  
            int totalItemCount) {  
        // 用户设置了加载更多监听器，且到了最底部，并且是上拉操作，那么执行加载更多.  
        if (mLoadListener != null && isBottom() && mScroller.getCurrY() <= mInitScrollY  
                && mYOffset <= 0  
                && mCurrentStatus == STATUS_IDLE) {  
            showFooterView();  
            doLoadMore();  
        }  
    }  
  
} 
```
在构造函数中会调用initLayout来添加Header View, Content View, Footer View这三个区域的视图， 其中Content View就是我们的核心组件，比如ListView、GridView，这个区域的视图默认宽高都是match_parent的。Header的高度为屏幕宽度的1/4，但它的有效显示区域只有100像素，其他的都是paddingTop，这样就是的内容显示区域显示在最下面。这样当用户一直下拉时，首先会显示内容区域，继续下拉则会显示PaddingTop区域，此时就达到header view高度被拉伸的效果。如下图 : 

|                   图 4                    |                    图5                    |
| :--------------------------------------: | :--------------------------------------: |
| <img src="http://img.blog.csdn.net/20141001182355328?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYmJveWZlaXl1/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center" width="400" /> | <img src="http://img.blog.csdn.net/20141001182533895?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYmJveWZlaXl1/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center" width="400" /> |

不断下拉，y轴的偏移量不断减小，使得header越来越多的部分显示出来。只有白色的内容显示区域是有效的显示区，上面的绿色都是paddingTop区，这样就形成了被拉伸的效果。

添加这三个view之后，我们在onMeasure中对这几个子view进行丈量。使得该组件的宽度为用户设置的宽度，高度为header, content view, footer的高度之和。得到各个子视图的宽高和该组件的总宽高以后，会进行布局操作，即会调用onLayout方法。我们把这个几个视图从上到下排列。最后将该组件在y方向上滚动与header view的高度同样大小的像素值，使得header view隐藏掉，使得Content View完全显示出来。
```java
/* 
 * 丈量视图的宽、高。宽度为用户设置的宽度，高度则为header, content view, footer这三个子控件的高度只和。 
 * @see android.view.View#onMeasure(int, int) 
 */  
@Override  
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {  
  
    int width = MeasureSpec.getSize(widthMeasureSpec);  
  
    int childCount = getChildCount();  
  
    int finalHeight = 0;  
  
    for (int i = 0; i < childCount; i++) {  
        View child = getChildAt(i);  
        // measure  
        measureChild(child, widthMeasureSpec, heightMeasureSpec);  
        // 该view所需要的总高度  
        finalHeight += child.getMeasuredHeight();  
    }  
  
    setMeasuredDimension(width, finalHeight);  
}  
  
/* 
 * 布局函数，将header, content view, 
 * footer这三个view从上到下布局。布局完成后通过Scroller滚动到header的底部，即滚动距离为header的高度 + 
 * 本视图的paddingTop，从而达到隐藏header的效果. 
 * @see android.view.ViewGroup#onLayout(boolean, int, int, int, int) 
 */  
@Override  
protected void onLayout(boolean changed, int l, int t, int r, int b) {  
  
    int childCount = getChildCount();  
    int top = getPaddingTop();  
    for (int i = 0; i < childCount; i++) {  
        View child = getChildAt(i);  
        child.layout(0, top, child.getMeasuredWidth(), child.getMeasuredHeight() + top);  
        top += child.getMeasuredHeight();  
    }  
  
    // 计算初始化滑动的y轴距离  
    mInitScrollY = mHeaderView.getMeasuredHeight() + getPaddingTop();  
    // 滑动到header view高度的位置, 从而达到隐藏header view的效果  
    scrollTo(0, mInitScrollY);  
} 
```
然后就是下拉刷新触发点了。在onInterceptTouchEvent方法中，对于ACTION_MOVE事件我们会判断，如果已经滑到了Content View的顶部，并且还继续下拉，那么拦截触摸事件，使得事件转到onTouchEvent方法中处理。事件拦截的关键点如下 :
```java
case MotionEvent.ACTION_MOVE:  
          // int yDistance = (int) ev.getRawY() - mYDown;  
          mYOffset = (int) ev.getRawY() - mLastY;  
          // 如果拉到了顶部, 并且是下拉,则拦截触摸事件,从而转到onTouchEvent来处理下拉刷新事件  
          if (isTop() && mYOffset > 0) {  
              return true;  
          }  
          break;  
```
如果在onTouchEvent中我们根据用户当前触摸事件的y轴位置与上一次的y轴位置的偏移量来修改该组件在y轴上的滚动值,调用的方法为changeScrollY()函数，并且会修改header中的文本内容。当用户抬起手指时，会判断用户在y轴上滑动的距离是否大于header view的1/2, 如果大于header view的1/2那么为有效的下拉刷新，此时滚动到刚好显示header view的内容y轴位置，然后触发刷新操作，直到用户调用refreshCompete()位置，最后完全隐藏header。否则视为无效的下拉刷新操作，然后通过Scroller滚动来隐藏header view。

而加载更多操作为用户滑动到了最底部，并且继续上拉，那么会触发加载更多的操作。在操作在onScroll方法中被触发。

基本原理就是通过一个ViewGroup来组织header view, content view, footer view， 使它们从上到下排列，并且在初始化时滚动y轴，使得header 和 footer完全隐藏，只显示content view。用户下拉或者上拉时，通过判断是否显示header 或者 footer, 也是通过Scroller来滚动y轴的偏移量来实现HeaderView, Footer View的显示和隐藏，不需要修改margin值，这样效率更高，滚动也更平滑。当用户的上拉或者下拉操作满足了条件时，则会触发相应的操作，即下拉刷新、上拉加载更多。如有不明白的地方，就对比参考[Android打造(ListView、GridView等)通用的下拉刷新、上拉自动加载的组件](http://blog.csdn.net/bboyfeiyu/article/details/39253051)吧，原理都差不多。

## 下拉刷新的ListView
```java
/** 
 * @author mrsimple 
 */  
public class RefreshListView extends RefreshLayoutBase<ListView> {  
  
    /** 
     * @param context 
     */  
    public RefreshListView(Context context) {  
        this(context, null);  
    }  
  
    /** 
     * @param context 
     * @param attrs 
     */  
    public RefreshListView(Context context, AttributeSet attrs) {  
        this(context, attrs, 0);  
    }  
  
    /** 
     * @param context 
     * @param attrs 
     * @param defStyle 
     */  
    public RefreshListView(Context context, AttributeSet attrs, int defStyle) {  
        super(context, attrs, defStyle);  
    }  
  
    @Override  
    protected void setupContentView(Context context) {  
        mContentView = new ListView(context);  
        // 设置滚动监听器  
        mContentView.setOnScrollListener(this);  
  
    }  
  
    @Override  
    protected boolean isTop() {  
  
        // Log.d(VIEW_LOG_TAG,  
        // "### first pos = " + mContentView.getFirstVisiblePosition()  
        // + ", getScrollY= " + getScrollY());  
        return mContentView.getFirstVisiblePosition() == 0  
                && getScrollY() <= mHeaderView.getMeasuredHeight();  
    }  
  
    @Override  
    protected boolean isBottom() {  
        // Log.d(VIEW_LOG_TAG, "### last position = " +  
        // contentView.getLastVisiblePosition()  
        // + ", count = " + contentView.getAdapter().getCount());  
        return mContentView != null && mContentView.getAdapter() != null  
                && mContentView.getLastVisiblePosition() ==  
                mContentView.getAdapter().getCount() - 1;  
    }  
} 
```
需要下拉刷新的组件只需要实现isTop来判断是否滑动到最顶端、isBottom是否滑动到最底部，已经通过setupContentView设置mContentView对象即可。

## 使用示例
```java
final RefreshListView refreshLayout = new RefreshListView(this);  
String[] dataStrings = new String[20];  
for (int i = 0; i < dataStrings.length; i++) {  
    dataStrings[i] = "item - " +  
            i;  
}  
// 获取ListView, 这里的listview就是Content view  
refreshLayout.setAdapter(new ArrayAdapter<String>(this,  
        android.R.layout.simple_list_item_1, dataStrings));  
// 设置下拉刷新监听器  
refreshLayout.setOnRefreshListener(new OnRefreshListener() {  
  
    @Override  
    public void onRefresh() {  
        Toast.makeText(getApplicationContext(), "refreshing", Toast.LENGTH_SHORT)  
                .show();  
  
        refreshLayout.postDelayed(new Runnable() {  
  
            @Override  
            public void run() {  
                refreshLayout.refreshComplete();  
            }  
        }, 1500);  
    }  
});  
  
// 不设置的话到底部不会自动加载  
refreshLayout.setOnLoadListener(new OnLoadListener() {  
  
    @Override  
    public void onLoadMore() {  
        Toast.makeText(getApplicationContext(), "loading", Toast.LENGTH_SHORT)  
                .show();  
  
        refreshLayout.postDelayed(new Runnable() {  
  
            @Override  
            public void run() {  
                refreshLayout.loadCompelte();  
            }  
        }, 1500);  
    }  
});  
```
效果图 : 

![](http://img.blog.csdn.net/20141010130441974)

效果图中含有下拉刷新的ListView, GridView, TextView，可以看到即使实在模拟器中，下拉刷新的效果都挺流畅的。上拉加载更多在ListView中正常显示，GridView中在模拟器上没有触发，但是在真机上是正常的。

## 代码地址
github在此 [通用的下拉刷新组件](https://github.com/hehonghui/android_my_pull_refresh_view/tree/master/src/com/uit/pullrefresh/scroller)，前言中提到的版本也在该仓库中，两个版本所在的包不一样。这篇文章的在com/uit/pullrefresh/scroller包下，前言中提到的版本在com/uit/pullrefresh/base包下。