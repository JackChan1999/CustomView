Touch 事件在进行分发的时候，由父 View 向它的子 View 传递，一旦某个子 View 开始接收进行处理，那么接下来所有事件都将由这个 View 来进行处理，它的 ViewGroup 将不会再接收到这些事件，直到下一次手指按下。而嵌套滚动机制（NestedScrolling）就是为了弥补这一机制的不足，为了让子 View 能和父 View 同时处理一个 Touch 事件

## NestedScrollingParent

```java
public interface NestedScrollingParent {
    
    public boolean onStartNestedScroll(View child, View target, int nestedScrollAxes);

    public void onNestedScrollAccepted(View child, View target, int nestedScrollAxes);

    public void onStopNestedScroll(View target);

    public void onNestedScroll(View target, int dxConsumed, int dyConsumed,
            int dxUnconsumed, int dyUnconsumed);

    public void onNestedPreScroll(View target, int dx, int dy, int[] consumed);

    public boolean onNestedFling(View target, float velocityX, float velocityY, boolean consumed);

    public boolean onNestedPreFling(View target, float velocityX, float velocityY);

    public int getNestedScrollAxes();
}
```

## NestedScrollingChild

NestedScrollingParent接口，对应处理的就是NestedScrollingChild传递出的数据，让控件通知CoordinatorLayout自己的状态

## NestedScrollingChildHelper

| setNestedScrollingEnabled() |      |
| --------------------------- | ---- |
| isNestedScrollingEnabled()  |      |
| startNestedScroll()         |      |
| stopNestedScroll()          |      |
| hasNestedScrollingParent()  |      |
| dispatchNestedScroll()      |      |
| dispatchNestedPreScroll()   |      |
| dispatchNestedFling()       |      |
| dispatchNestedPreFling()    |      |

```
public class TouchBehavior extends CoordinatorLayout.Behavior implements NestedScrollingChild {

    private NestedScrollingChildHelper childHelper;
    private float                      ox;
    private float                      oy;
    private int[] consumed = new int[2];
    private int[] offsetInWindow = new int[2];

    public TouchBehavior(Context context, AttributeSet attrs) {
        super(context, attrs);
    }
    @Override
    public boolean onTouchEvent(CoordinatorLayout parent, View child, MotionEvent ev) {
        if (childHelper == null) {
            childHelper = new NestedScrollingChildHelper(child);
            setNestedScrollingEnabled(true);
        }
        switch (ev.getAction()) {
            case MotionEvent.ACTION_DOWN:
                ox = ev.getX();
                oy = ev.getY();
                if (oy < child.getTop() || oy > child.getBottom() || ox < child.getLeft() || ox > child.getRight()) {
                    return true;
                }
                //开始滑动
                startNestedScroll(ViewCompat.SCROLL_AXIS_VERTICAL);
                break;
            case MotionEvent.ACTION_MOVE:

                float clampedX = ev.getX();

                float clampedY = ev.getY();

                int dx = (int) (clampedX - ox);
                int dy = (int) (clampedY - oy);

                if (Math.abs(dy) > Math.abs(dx)) {
                    //分发触屏事件给parent处理
                    dispatchNestedPreScroll(dx, -dy, consumed, offsetInWindow);
                }
                break;
            case MotionEvent.ACTION_UP:
                stopNestedScroll();
                break;
        }
        return true;
    }

    @Override
    public void setNestedScrollingEnabled(boolean enabled) {
        childHelper.setNestedScrollingEnabled(enabled);
    }

    @Override
    public boolean isNestedScrollingEnabled() {
        return childHelper.isNestedScrollingEnabled();

    }

    @Override
    public boolean startNestedScroll(int axes) {
        return childHelper.startNestedScroll(axes);
    }

    @Override
    public void stopNestedScroll() {
        childHelper.stopNestedScroll();

    }

    @Override
    public boolean hasNestedScrollingParent() {
        return childHelper.hasNestedScrollingParent();
    }

    @Override
    public boolean dispatchNestedScroll(int dxConsumed, int dyConsumed, int dxUnconsumed, int dyUnconsumed, int[] offsetInWindow) {
        return childHelper.dispatchNestedScroll(dxConsumed, dyConsumed, dxUnconsumed, dyUnconsumed, offsetInWindow);
    }

    @Override
    public boolean dispatchNestedPreScroll(int dx, int dy, int[] consumed, int[] offsetInWindow) {
        return childHelper.dispatchNestedPreScroll(dx, dy, consumed, offsetInWindow);
    }

    @Override
    public boolean dispatchNestedFling(float velocityX, float velocityY, boolean consumed) {
        return childHelper.dispatchNestedFling(velocityX, velocityY, consumed);
    }

    @Override
    public boolean dispatchNestedPreFling(float velocityX, float velocityY) {
        return childHelper.dispatchNestedPreFling(velocityX, velocityY);
    }
}
```

## NestedScrollingParentHelper

## 对外传递触摸事件