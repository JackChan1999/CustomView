## 前言
下拉刷新组件在开发中使用率是非常高的，基本上联网的APP都会采用这种方式。对于开发效率而言，使用获得大家认可的开源库必然是效率最高的，但是不重复发明轮子的前提是你得自己知道轮子是怎么发明出来的，并且自己能够实现这些功能。否则只是知道其原理，并没有去实践那也就是纸上谈兵了。做程序猿，动手做才会遇到真正的问题，否则就只是自以为是的认为自己懂了。今天这篇文章就是以自己重复发明轮子这个出发点而来的，通过实现经典、使用率较高的组件来提高自己的认识。下面我们就一起来学习吧。
整体布局结构
|                    图1                    |                    图2                    |
| :--------------------------------------: | :--------------------------------------: |
| <img src="http://img.blog.csdn.net/20140913165858954?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYmJveWZlaXl1/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center" width="300" /> | <img src="http://img.blog.csdn.net/20140913165828046?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYmJveWZlaXl1/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center" width="300" /> |

该组件整体以竖直方向的LinearLayout为根视图，分别是Header、ContentView、Foooter, 从上到下依次排列下来，其中ContentView的宽高都为match_parent,footer和header的宽、高分别为match_parent、wrap_content，原始效果如图1；在Header、Foooter初始时都会通过设置padding隐藏掉，如图2中的Header设置paddingTop为负的Header的高度值，同理Footer也通过设置paddingBottom为Footer的负的Footer高度来达到隐藏的效果，所以只有 ContentView区域显示出来。当用户下拉到顶端，并且继续下拉时触发下拉刷新操作；当用户上拉到底部， 并且继续上拉时触发加载更多的操作。

原理都虽然简单，但是实现起来却也是会有很多小麻烦。这里没有采用通过设置onTouchListener的方法，因此使用这个方式在下拉的时候依然会出现ListView的最顶部的"HOLD"视图，不太爽。这种实现方法也蛮简单的，具体看郭神的博客 [Android下拉刷新完全解析](http://blog.csdn.net/guolin_blog/article/details/9255575)，教你如何一分钟实现下拉刷新功能。

## 下拉刷新基本原理

基本原理就是在用户滑动屏幕上的组件时，在onInterceptTouchEvent方法中判断是否到了ContentView （这里我们以ListView为例来说明）的最顶端，如果到了最顶端且用户还继续向下滑，那么会拦截触摸事件避免它分发到ListView，即在onInterceptTouchEvent中返回true （ 不太清楚的可以参考资料如下 : [Android Touch事件分发过程](http://blog.csdn.net/bboyfeiyu/article/details/38958829)。 ），这样就将触摸事件分发到了onTouchEvent函数中，我们对于用户触摸事件的处理逻辑主要都在这个函数中。如果该函数返回false，那么触摸事件则会分发给其Child View，这里的这个Child View就是ListView了，当返回false时用户滑动屏幕时就会滚动ListView。
```java
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
            mYDown = (int) ev.getRawY();  
            break;  
  
        case MotionEvent.ACTION_MOVE:  
            // int yDistance = (int) ev.getRawY() - mYDown;  
            mYDistance = (int) ev.getRawY() - mYDown;  
            showStatus(mCurrentStatus);  
            Log.d(VIEW_LOG_TAG, "%%% isBottom : " + isBottom() + ", isTop : " + isTop()  
                    + ", mYDistance : " + mYDistance);  
            // 如果拉到了顶部, 并且是下拉,则拦截触摸事件,从而转到onTouchEvent来处理下拉刷新事件  
            if ((isTop() && mYDistance > 0)  
                    || (mYDistance > 0 && mCurrentStatus == STATUS_REFRESHING)) {  
                return true;  
            }  
            break;  
  
    }  
  
    // Do not intercept touch event, let the child handle it  
    return false;  
}  
```
首先我们在ACTION_DOWN事件中记录用户按下的触摸点的Y轴坐标mYDown，然后在ACTION_MOVE中再次获取Y轴的坐标，计算出两者之间的差值。如果滑动的差值大于mTouchSlop则继续进行处理，mTouchSlop为判断一个触摸滑动事件是否有效的的最小阀值，如果小于这个阀值我们认为这个触摸滑动事件无效，例如手抖了一下，距离很短，因此我们忽略类似的事件。

在有效的滑动距离之内，我们判断当前组件的状态，如果不是正在刷新的状态，那么我们根据当前ListView的paddingTop的高度来设置不同的值，paddingTop如果高度大于ListView高度的70%，那么我们将当前状态设置为“释放可刷新”状态，即STATUS_RELEASE_TO_REFRESH状态；反之，我们设置状态为“继续下拉”状态，即“STATUS_PULL_TO_REFRESH”。通过这个paddingTop高度我们来规定当用户下拉距离到一定的距离后才出发刷新操作，否则视为无效下拉。然而不管这个时候是什么状态，我们都会修改Header的padding Top属性，从而达到拉伸header的效果。

当状态为“释放可刷新”时，我们抬起手指，会出发ACTION_UP事件，此时我们在该事件中进行下拉刷新操作。onTouchEvent代码如下 :
```java
/* 
 * 在这里处理触摸事件以达到下拉刷新或者上拉自动加载的问题 
 * @see android.view.View#onTouchEvent(android.view.MotionEvent) 
 */  
@Override  
public boolean onTouchEvent(MotionEvent event) {  
  
    Log.d(VIEW_LOG_TAG, "@@@ onTouchEvent : action = " + event.getAction());  
    switch (event.getAction()) {  
        case MotionEvent.ACTION_DOWN:  
            mYDown = (int) event.getRawY();  
            Log.d(VIEW_LOG_TAG, "#### ACTION_DOWN");  
            break;  
  
        case MotionEvent.ACTION_MOVE:  
            Log.d(VIEW_LOG_TAG, "#### ACTION_MOVE");  
            int currentY = (int) event.getRawY();  
            mYDistance = currentY - mYDown;  
            // 高度大于header view的高度才可以刷新  
            if (mYDistance >= mTouchSlop) {  
                if (mCurrentStatus != STATUS_REFRESHING) {  
                    //  
                    if (mHeaderView.getPaddingTop() > mHeaderViewHeight * 0.7f) {  
                        mCurrentStatus = STATUS_RELEASE_TO_REFRESH;  
                        mTipsTextView.setText(R.string.pull_to_refresh_release_label);  
                    } else {  
                        mCurrentStatus = STATUS_PULL_TO_REFRESH;  
                        mTipsTextView.setText(R.string.pull_to_refresh_pull_label);  
                    }  
                }  
  
                rotateHeaderArrow();  
                int scaleHeight = (int) (mYDistance * 0.8f);// 去了滑动距离的80%，减小灵敏度而已  
                // Y轴的滑动距离小于屏幕高度4分之一时才会拉伸header,反之保持不变  
                if (scaleHeight <= mScrHeight / 4) {  
                    adjustHeaderPadding(scaleHeight);  
                }  
            }  
  
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
```
抬起手指时出发的刷新操作，代码如下：
```java
/** 
 * 执行刷新操作 
 */  
private final void doRefresh() {  
    if (mCurrentStatus == STATUS_RELEASE_TO_REFRESH) {  
        // 设置状态为正在刷新状态  
        mCurrentStatus = STATUS_REFRESHING;  
        mArrowImageView.clearAnimation();  
        // 隐藏header中的箭头图标  
        mArrowImageView.setVisibility(View.GONE);  
        // 设置header中的进度条可见  
        mHeaderProgressBar.setVisibility(View.VISIBLE);  
        // 设置一些文本  
        mTimeTextView.setText(R.string.pull_to_refresh_update_time_label);  
        SimpleDateFormat sdf = new SimpleDateFormat();  
        mTimeTextView.append(sdf.format(new Date()));  
        //  
        mTipsTextView.setText(R.string.pull_to_refresh_refreshing_label);  
  
        // 执行回调  
        if (mPullRefreshListener != null) {  
            mPullRefreshListener.onRefresh();  
        }  
        // 使headview 正常显示, 直到调用了refreshComplete后再隐藏  
        new HeaderViewHideTask().execute(0);  
  
    } else {  
        // 隐藏header view  
        adjustHeaderPadding(-mHeaderViewHeight);  
    }  
} 
```
在刷新状态时，header正常显示，即此时的padding top需要设置为0，我们使用一个异步任务来逐步修改padding top的值，使得header从拉伸效果逐步、平滑的恢复原始的大小。用户调用refreshComplete()函数后，即刷新完成后，再逐步调整listview的padding top将其隐藏。至此，整个下拉刷新过程结束。

## 滑动到底部自动加载

滑动到底部自动加载相对来说要简单得多，我们也是以ContentView是ListView的情况来说明。原理就是监听ListView （ 即 ContentView ）的的滚动事件，因此如果ContentView的类型不支持滚动事件，则不能实现该功能。listview符合要求，因此其能实现自动加载。我们在onScroll函数中判断listview是否到了最后一项，如果到了最后一项，那么显示出footer，并且开始加载。当用户调用loadMoreComplete函数时代表加载结束。此时隐藏footer，整个过程结束。
​    
```java
/* 
 * 滚动事件，实现滑动到底部时上拉加载更多 
 * @see android.widget.AbsListView.OnScrollListener#onScroll(android.widget. 
 * AbsListView, int, int, int) 
 */  
@Override  
public void onScroll(AbsListView view, int firstVisibleItem, int visibleItemCount,  
        int totalItemCount) {  
  
    Log.d(VIEW_LOG_TAG, "&&& mYDistance = " + mYDistance);  
    if (mFooterView == null || mYDistance >= 0 || mCurrentStatus == STATUS_LOADING  
            || mCurrentStatus == STATUS_REFRESHING) {  
        return;  
    }  
  
    loadmore();  
}  
  
/** 
 * 下拉到底部时加载更多 
 */  
private void loadmore() {  
    if (isShowFooterView() && mLoadMoreListener != null) {  
        mFooterTextView.setText(R.string.pull_to_refresh_refreshing_label);  
        mFooterProgressBar.setVisibility(View.VISIBLE);  
        adjustFooterPadding(0);  
        mCurrentStatus = STATUS_LOADING;  
        mLoadMoreListener.onLoadMore();  
    }  
} 
```
其中loadmore函数中调用的isShowFooterView函数就是用来判断是否到了最底部的，代码如下 ： 
```java
/* 
    * 下拉到listview 最后一项时则返回true, 将出发自动加载 
    * @see com.uit.pullrefresh.base.PullRefreshBase#isShowFooterView() 
    */  
   @Override  
   protected boolean isShowFooterView() {  
       if (mContentView == null || mContentView.getAdapter() == null) {  
           return false;  
       }  
  
       return mContentView.getLastVisiblePosition() == mContentView.getAdapter().getCount() - 1;  
   }  
```
OK，至此整个核心的过程介绍完毕了。

## 效果图

ListView

![](http://img.blog.csdn.net/20140913182119437)

TextView下拉刷新

![](http://img.blog.csdn.net/20140913182350710)

## github地址
[猛击这里!](https://github.com/hehonghui/android_my_pull_refresh_view)