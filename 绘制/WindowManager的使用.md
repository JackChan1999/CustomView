## 1. WindowManager,Window的介绍

### 1.1 Window

Window表示一个窗口。对于Android里的Window，我们可以类比Windows系统中的Window，在Windows中，每打开一个软件，都会弹出一个窗口，这个窗口右上角有最小化，最大化，关闭按钮，做了某些操作时，也可能会弹出一个窗口，下面可能会有确定，取消之类的按钮，这些都是Windows系统中的窗口。如图所示：

![img](http://www.itcast.cn/files/image/201607/20160726102657454.png)

在Android里，也有Window的概念，但是Android里的Window没有边框， 也没有最大最小关闭按钮的。如图所示：

![img](http://www.itcast.cn/files/image/201607/20160726102757112.png)

Android中所有的界面都是显示在一个个Window中的，包括Activity，Dialog，Toast，甚至状态栏，最近应用列表，都是在Window中显示的。只是我们看不到这些Window的边框，只能看到里面的内容。其实 Window并不能真正的显示内容，它只是一个虚拟的"框"，真正能显示内容的是View。Window是View的直接管理者，触摸事件也是先由Window接收，然后传递给View的。

Window是一个抽象类，在Android手机中，Window的实现类是PhoneWindow。

### 1.2 WindowManager

WindowManager是Window的管理者，对应着系统底层的一个服务：WindowManagerService。

我们无法直接访问Window，要操作Window，必须通过WindowManager。WindowManager有三个常用方法：addView()、removeView()、updateViewLayout()，我们可以通过WindowManager往屏幕上添加/删除一个Window，或者通过它修改一个Window的布局参数。

WindowManager是一个接口，在Android中，WindowManager的实现类WindowManagerImpl。

## 2. WindowManager使用详解

### 2.1 往屏幕上添加一个Window

调用WindowManager的addView方法即可。
```java
private WindowManager mWindowManager;
private View mView;
private WindowManager.LayoutParams mParams;
private void addWindow() {
    mWindowManager = (WindowManager) getSystemService(Context.WINDOW_SERVICE);
    mView = new TextView(getApplicationContext());
    TextView tv = (TextView) mView;
    tv.setText("我是Window中的View");
    tv.setTextColor(Color.RED);
    mParams = new WindowManager.LayoutParams();
    // 设置宽高
    mParams.width = WindowManager.LayoutParams.WRAP_CONTENT;
    mParams.height = WindowManager.LayoutParams.WRAP_CONTENT;
    // 设置Window的背景支持半透明
    mParams.format = PixelFormat.TRANSLUCENT;
    mParams.flags = WindowManager.LayoutParams.FLAG_KEEP_SCREEN_ON
            | WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE
            | WindowManager.LayoutParams.FLAG_NOT_TOUCHABLE;
    mParams.type = WindowManager.LayoutParams.TYPE_TOAST;
    // 设置Window的对齐方式，其实就是设置Window的坐标原点位置
    mParams.gravity = Gravity.LEFT | Gravity.TOP;
    // 设置Window的 x，y 坐标(相对于坐标原点)
    mParams.x = 100;
    mParams.y = 250;
    // 设置Window标题，显示在在 HierarchyView 透视图中的 Windows面板里的名称
    mParams.setTitle("AddWindow");
    // 往屏幕上添加一个Window，并且把第一个参数View放在Window中
    mWindowManager.addView(mView,mParams);
}
```
显示效果如下：

![img](http://www.itcast.cn/files/image/201607/20160726102946584.png)

### 2.2 代码详解

上面的代码表示：调用 WindowManager的 addView(View view，WindowManager.LayoutParams lp) 方法，往屏幕上添加一个Window，这个Window中显示的内容为第一个参数设置的View，Window的显示位置以及其他属性由第二个参数 WindowManager.LayoutParams指定。

这个方法很简单，但是 WindowManager.LayoutParams 中有两个字段比较重要，这里详细说一下。

#### flags

用来控制Window的显示特性，有很多可取的值，不同的的值表示不同的显示特性， 如果希望Window具有多个值的特性，可以使用 “|” 将这些值进行按位或运算。这里介绍几个比较常用的取值：

| 标记                   | 功能说明                                     |
| :------------------- | :--------------------------------------- |
| FLAG_NOT_TOUCHABLE   | Window不接收触摸事件                            |
| FLAG_NOT_FOCUSABLE   | Window不获取焦点，即不能接收按键事件，按键事件传递给下层具有焦点的Window |
| FLAG_NOT_TOUCH_MODAL | 表示系统会将当前Window区域外的任何事件传递给底层的Window，当前Window区域内的事件自己处理，一般来说，都需要开启此标记，否则其他Window无法获取事件. 当设置了FLAG_NOT_FOCUSABLE后，此标记也会自动设置 |
| FLAG_KEEP_SCREEN_ON  | Window显示期间，保持屏幕高亮                        |

#### type

用来表示Window的类型，Window有三种大的类型，分别是应用Window，子Window和系统Window。Activity的Window就是一种应用Window，Dialog的Window是一种子Window，子Window不能单独存在，必须附属在特定的父Window中，这也就是为什么Dialog的Context必须是Activity。系统Window大都是(不是全部)需要声明权限才能创建，独立应用Window之外，比如Toast，状态栏等等。

Window是分层的，层级大的会覆盖在层级小的之上，三大类Window中，应用Window层级范围是1-99，子Window是1000-1999，系统Window是2000-2999，这就是为什么Dialog显示在Activity之上，而Toast又可以显示在Dialog之上。如图：

![img](http://www.itcast.cn/files/image/201607/20160726103201714.png)

### 2.3. 查看屏幕上的Window

我们再往屏幕上加一个PopupWindow和一个Dialog，当前界面如下：

![img](http://www.itcast.cn/files/image/201607/20160726103710315.jpg)

在AS中，点击菜单 Window - Open Perspective - Others，选择 HierarchyView，打开，选择Windows面板，可以看到当前屏幕中所有的Window：

![img](http://www.itcast.cn/files/image/201607/20160726103754707.jpg)

我们添加的Window在其中显示的标题为AddWindow，另外，我们可以看到还有别的几个Window，比如 PopupWindow，MainActivity，加粗的那一个其实是MainActivity中弹出的Dialog，还能看到 StatusBar(状态栏)，RecentsPanel(最近应用列表)等等，这也证明了我们前面说的，Android中所有的界面都是显示在Window中的。

## 3. 桌面悬浮窗实现思路

### 3.1 在桌面上显示Window

如果我们在Activity中使用WindowManager添加Window，当Activity退出时，添加的Window也会被回收掉。所以要想在桌面上显示悬浮窗，可以在Service中使用WindowManager添加Window，这样只要服务不停止，就可以一直显示。当服务启动时，在其onCreate方法中，使用WindowManager的 addView方法添加一个系统Window，当服务销毁时，可以在其 onDestroy中使用WindowManager的removeView 方法移除Window。大体是这样的思路，代码就不再给出了。

### 3.2 让这个Window随手指移动

要想让这个Window能接收事件，需要给他设置相应的flags(只要不包含FLAG_NOT_TOUCHABLE即可)，另外其type也不能是 TYPE_TOAST。可以使用：TYPE_PRIORITY_PHONE，表示比来去电界面的Window级别还要高一些(来去电界面的Window是系统Window)。

```java
mParams.type = WindowManager.LayoutParams. TYPE_PRIORITY_PHONE;
mParams.flags = WindowManager.LayoutParams.FLAG_KEEP_SCREEN_ON
            | WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE;
```
注意添加权限：

```xml
<uses-permission android：name="android.permission.SYSTEM_ALERT_WINDOW"/>
```
然后给Window里的View设置onTouchListener，重写onTouch方法：
```java
private int mStartX;
private int mStartY;
@Override
public boolean onTouch(View v,MotionEvent event) {
    switch (event.getAction()) {
    case MotionEvent.ACTION_DOWN：
        // 记录坐标起始点，getRawX，getRawY返回值为float，
        // 需转化为int，变成像素数后再使用
        mStartX = (int) event.getRawX();
        mStartY = (int) event.getRawY();
        break;
    case MotionEvent.ACTION_MOVE：
        int newX = (int) event.getRawX();
        int newY = (int) event.getRawY();
        // 获取手指移动的距离
        int dx = newX - mStartX;
        int dy = newY - mStartY;
        // 修改Window的x，y坐标
        mParams.x += dx;
        mParams.y += dy;
        // 修改Window的布局参数
        // 这里不能修改Window里的View的布局参数，因为View是在Window中显示的，
        // 修改View的布局参数并不能移动外面的Window
        mWindowManager.updateViewLayout(mView,mParams);
        // 重新记录新的坐标起始点
        mStartX = (int) event.getRawX();
        mStartY = (int) event.getRawY();
        break;
    default：
        break;
    }
    return true;
}
```
这样就实现了Window随着手指拖动而移动了。