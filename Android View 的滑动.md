## Android View 的滑动

### 1.  View 的滑动是如何产生的

滑动一个 view，本质上是移动一个 view。原理与动画效果的实现非常相似，都是通过不断地改变 view 的坐标（scrllX、scrollY 或 translateX、translateY）的坐标来实现这一效果。

实现方法：监听用户手指 ACTION_DOWN 的位置，然后在 ACTION_MOVE 中得到 offsetX 或者 offsetY，利用这个量来不停地改变 view 的位置。

### 2. View 的坐标系 ScrollX/Y 、TranslateX/Y

#### 2.1. view 的坐标系

![](https://ww2.sinaimg.cn/large/006tKfTcgy1feikd3oeb0j307708laa7.jpg)

getLeft、getTop、getBottom、getRight、getX 和 getY 都是相对于父控件而言。getRawX 和 getRawY 是相对于屏幕而言



#### 2.2. ScrollX 和 ScrollY

scrollX 指的是 view 的左边缘相对于 **view 内容**的左边缘的距离。scrollY 指的是 view 的上边缘相对于 view 内容的上边缘的距离。

scrollX 和 scrollY 改变的是 **view 内容**的位置，而不是 view 的位置。



#### 2.3. TranslateX 和 TranslateY

android 3.0 以后添加的属性。

~~~java
x = getleft() + translateX
~~~

getLeft() 的值是不变的，变化的是 x 和 translateX。改变的是 view。



### 3. 具体滑动方法

#### 3.1 scrollTo 与 scrollBy

scrollTo ( x, y ) 是移动到一个具体的点 （ x, y ）。scrollBy ( offsetX, offsetY ) 是移动一个偏移量。

~~~java
int offsetX = x - lastX;
int offsetY = y - lastY;
// 此时移动的是 view 里面的内容，并没有移动 view
scrollBy(offsetX, offsetY);
//换成下面这句才行
// 所有在 parent 里面的所有子 view 都会移动
// 注意方向问题
((View)getParent()).scrollBy(-offsetX, -offsetY);
~~~

所以在做 嵌套滑动 的时候，不能用 scrollTo 和 scrollBy。因为里面的 view 都会整体移动而流出来空隙。

#### 3.2 Scroller

scrollTo 和 scrollBy 的滑动是瞬间完成的，Scroller 可以完成一个平滑的滑动。

在实现 子 view 的滑动中，虽然 scrollBy 方法是让子 view 瞬间从一个点到另一个点，但是由于在 ACTION_MOVE 事件中不断获取手指移动的微小的偏移量，这样就将一段距离划分成了 N 个非常小的偏移量。利用人眼的视觉暂留特性，整体上实现了 view 的平滑移动。Scroller 的原理与此类似。

使用步骤如下：

- 初始化 Scroller

  通过构造方法来创建一个 Scroller 对象：

  ~~~java
  mScroller = new Scroller(context);
  ~~~

- 重写 computerScroll() 方法，实现模拟滑动

  computerScroll() 是 Scroller 类的核心，系统绘制 view 的时候会在 draw() 方法中调用该方法。内部实际上使用的是 scrollTo 方法，通过不停地瞬间移动一个小的距离来实现整体上的平滑移动效果。

  ~~~java
  @Override
  public void computeScroll(){
    super.computeScroll();
    //判断 Scroller 是否执行完毕, 返回 TRUE 证明还没有执行完毕
    if(mScroller.computeScrollOffset()){
      ((View)getParent()).scrollTo(mScroller.getCurrX,
      							mScroller.getCurrY);
      // 调用 invalidate() 不断地通知重绘 从而调用 computeScroll
      invalidate();
    }
  }
  ~~~

  Scroller 类提供了 computerScrollOffset() 方法来判断是否完成了整个滑动，同时提供了 getCurrX()、getCurrY() 方法来获得当前的滑动坐标。由于 computeScroll() 方法不会主动调用，所以只能通过 invalidate() —> draw() —> computeScroll() 来间接调用。

- startScroll 开始调用

  Scroller 类的 startScroll() 方法来开启平滑移动过程。方法参数如下：

  ~~~java
  public void startScroll(int startX, int startY, int dx, int dy, int duration)
  public void startScroll(int startX, int startY, int dx, int dy);
  ~~~

  下面是调用的具体代码：

  ~~~java
  View viewGroup = (View)getParent;
  mScroller.startScroll(startX, startY, dx, dy);
  ~~~



### 3.3 LayoutParams

Layoutparams 保存了一个 View 的布局参数（对应于 xml 中的一些 marginTop 之类的，这些属性属于 ViewGroup 不属于 view 本身， 但是通过 view 的 个体LayoutParams() 可以得到）。通过动态地改变一个布局的位置参数，从而达到改变 view 位置的效果。

~~~java
// 根据 父控件的不同选择不同的 LayoutParams
LinearLayout.LayoutParams layoutParams = (LinearLayout.LayoutParams)getLayoutParams();
// 如果嫌麻烦 或者 事先不知道父布局，可以使用 ViewGroup.MarginLayoutParams
layoutParams.leftMargin = getLeft() + offsetX;
layoutParams.topMargin = getTop() + offsetY;
setLayoutParams(layoutParams);
~~~

### 3.4 Layout 方法

view 进行绘制时，会调用 onLayout() 方法来设置显示的位置。因此，可以通过修改 view 的 left、top、right、bottom 属性来控制 view 的坐标。**这种方法只能在自定义 View 中使用。**代码如下：

~~~java
  // 视图坐标方式
    @Override
    public boolean onTouchEvent(MotionEvent event) {
        int x = (int) event.getX();
        int y = (int) event.getY();
        switch (event.getAction()) {
            case MotionEvent.ACTION_DOWN:
                // 记录触摸点坐标
                lastX = x;
                lastY = y;
                break;
            case MotionEvent.ACTION_MOVE:
                // 计算偏移量
                int offsetX = x - lastX;
                int offsetY = y - lastY;
                // 在当前left、top、right、bottom的基础上加上偏移量
                layout(getLeft() + offsetX,
                        getTop() + offsetY,
                        getRight() + offsetX,
                        getBottom() + offsetY);
//                        offsetLeftAndRight(offsetX);
//                        offsetTopAndBottom(offsetY);
                break;
        }
        return true;
    }
~~~



### 3.4 ViewDragHelper (TODO)

google 为 support 库中为我们提供了 DraweLayout 和 SlidingPaneLayout 两个布局来帮助开发者实现侧边栏滑动的效果。里面的实现为 ViewDragHelper。**ViewDragHelper 的子类最好为 ViewGroup **。

##### 3.4.1 ViewDragHelper 使用步骤

- 初始化 ViewDragHelper

  ViewDragHelper 一般在 ViewGroup 内部，通过静态工厂方法进行初始化，代码如下：

  ~~~java
  // this 一般是一个 ViewGroup，即 parentView。第二个参数是一个 // Callback 回调，这个回调就是 ViewDragHelper 的逻辑核心
  mViewDragHelper = ViewDragHelper.create(this, callback);
  ~~~

- 拦截事件

  重写事件拦截方法，将事件传递给 ViewDragHelper 进行处理。

  ~~~java
  @Override
  public boolean onInterruptTouchEvent(MotionEvent ev){
    return mViewDragHelper.shouldInterceptTouchEvent(ev);
  }

  @Override
  public boolean onTouchEvent(MotionEvent event){
    //将触摸事件传递给 ViewDragHelper， 此操作必不可少
    mViewDragHelper.processTouchEvent(event);
    return true;
  }
  ~~~

- 处理 computeScroll()

  ViewDragHelper 内部是通过 Scroller 来实现平滑移动的。

  ~~~java
  @Override
  public void computeScroll(){
    if(mViewDragHelper.continueSetting(true)){
      ViewCompat.postInvalidateOnAnimation(this);
    }
  }
  ~~~

- 处理回调 Callback

  ~~~java
  private ViewDragHelper.Callback callback = new ViewDragHelper.Callback(){
    @Override
    public boolean tryCaputeView(View child, int pointerId){
      /**
      * 通过 tryCaputeView，可以指定哪一个子 view 可以被移动。例如在这
      * 个实例中自定义了一个 ViewGroup, 里面自定义了两个子 View  		*(MenuView 和 MainView)，指定 mMainView 可以被拖动
      */
      return mMainView == child;
    }
    
        // 当拖拽状态改变，比如idle，dragging
    @Override
    public void onViewDragStateChanged(int state) {
      super.onViewDragStateChanged(state);
    }

        // 当位置改变的时候调用,常用与滑动时更改scale等
    @Override
    public void onViewPositionChanged(View changedView, int left, int top, int dx, int dy) {
      super.onViewPositionChanged(changedView, left, top, dx, dy);
    }
    
    @Override
    public int clampViewPositionVertical(View child, int top, int dy){
      // 如果返回 top 则表明可以在数值方向上滑动。如果不断此方向滑动，返回 0
      // dy 表明相对于前一次的增量
      return top;
    }
    
    @Override
    public int clampViewPositionHorizontal(View child, int left, int dx){
      // 返回 left 表明可以在水平位置上滑动。如果此方向不滑动，return 0
      // dx 表明相对于前一次的增量
      return left;
    }
    
    @Override
    public void onViewRelease(View releasedChild, float xvel, float yvel){
      // 拖动结束后释放 View，内部方法通过 Scroll 实现，所以要重写 computScroll() 方法
      super.onViewRelease(releaseChild, xvel, yvel);
      mViewDragHelper.smoothSlideViewTo(mMainView, 0, 0);
      ViewCompant.postInvalidateOnAnimation(this);
    }
  }
  ~~~



##### 3.4.2 为什么 clickable = true 的对象不可以呢？

如果子 View 不消耗事件，那么整个手势（DOWN-MOVE*-UP）都是直接进入 onTouchEvent，在 onTouchEvent的 DOWN 的时候就确定了 captureView。 如果消耗事件，那么就会先走 onInterceptTouchEvent 方法，判断是否可以捕获，而在判断的过程中会去判断另外两个回调的方法： getViewHorizontalDragRange 和getViewVerticalDragRange，只有这两个方法返回大于0的值才能正常的捕获。

所以，如果你用 Button 测试，或者给 TextView 添加了 clickable = true ，都记得重写下面这两个方法：

~~~java
@Override
public int getViewHorizontalDragRange(View child)
{
     return getMeasuredWidth()-child.getMeasuredWidth();
}

@Override
public int getViewVerticalDragRange(View child)
{
     return getMeasuredHeight()-child.getMeasuredHeight();
}
~~~



### 3.5 属性动画（ObjectAnimator）

~~~java
ObjectAnimator.ofFloat(mView, "translationX", 0, 100).setDuration(100).start();
~~~

其实改变的是 translationX 属性。translationX 改变的是 view。



### 4. 同样是滑动 改变的是什么属性

Layout() 方法 和 LayoutParams 的方式真正改变了 View 的位置（能有引发 view 的getLeft getTop getRight getBottom 的变化）。其余都不行。

**x 的值 和 getLeft 不是一个概念** 所以在自定义 View 的时候，根据使用的方法不同，保持统一性