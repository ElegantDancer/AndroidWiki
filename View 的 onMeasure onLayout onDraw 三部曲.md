## View 的 onMeasure onLayout onDraw 三部曲

[TOC]

## 前言

## 0.1 ViewRoot 和 decorView

view 的绘制流程是从 ViewRoot 的 performTraversals 方法开始的。经过 measure、layout、draw 三个过程才能最终将一个 View 绘制出来，其中 measure 用来绘制 View 的宽和高，layout 用来确定 VIew 在父容器中的放置位置，而 draw 负责将 View 绘制在屏幕上。大致流程如下：

![](https://ww4.sinaimg.cn/large/006tKfTcgy1fekz5kxpzfj30cx09jwf4.jpg)

performMeasure、performLayout、performDraw 完成顶级 View 的 measure、layout、draw 过程。

decorView 作为顶级 view ，一般情况下包含一个竖直方向的 LinearLayout，包含两个部分，上面是标题栏，下面是内容栏。通过 setContentView 将我们的布局文件加入 `id = content`内容栏。通过 `ViewGroup content = findViewById(R.android.id.content);`可以得到 content。而通过 `content.getChildAt(0)`可以得到我们自己的 View。

![](https://ww2.sinaimg.cn/large/006tKfTcgy1fekzemfr5uj307708wglp.jpg)

## 0.2 MeasureSpec

Measure 将 SpecMode 和 SpecSize 打包成一个 32 位的 int 值。高 2 位代表 SpecMode（测量模式），低 30 位代表 SpecSize（某种测量模式下的规格大小）。

### 0.2.1 SpecMode

- UNSPECFIED

  父容器不对 View 做任何限制，要多大给多大，一般用于系统内部，表示一种测量的状态。

- EXACTLY

  父容器已经检测出 View 所需要的精确大小，这个时候 View 的最终大小就是 SpecSize 所指定的值。对应于 LayoutParams 中的 match_parent 和具体的数值两种模式。

- AT_MOST

  父容器制定了一个可用大小即 SpecSize，View 的大小不能超过这个值。对应于 LayoutParams 中的 wrap_content。

### 0.2.2 MeasureSpec 和 MarginLayoutParams 的对应关系

总结：父容器的 MeasureSpec 和 子 View  的 MarginLayoutParams 共同决定了 子View 的 MeasureSpec。

下面的代码展示 DecorView 的 MeasureSpec 的创建过程，其中 desiredWindowWidth 和 desiredWindowHeight 是屏幕的尺寸：

~~~java
childWidthMeasureSpec = getRootMeasureSpec(desiredWindowWidth, lp.width);
childHeightMeasureSpec = getRootMeasureSpec(desiredWiondwHeight, lp.height);
performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
~~~

接下来看一下 getRootMeasureSpec 方法的实现:

~~~java
private static int getRootMeasureSpec(int wiindowSize, int rootDimension){
    int measureSpec;
    switch (rootDimension) {
        case ViewGroup.LayoutParams.MATCH_PARENT:
            // window can't resize. Force root view to be windowSize
            measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.EXACTLY);
            break;
        case ViewGroup.LayoutParams.WRAP_CONTENT:
            // window can resize. Set max size for root view
            measureSpec = MeasureSpec.makeMeasureSpec(wiindowSize, MeasureSpec.AT_MOST);
            break;
        default:
            // Window wants to be an exact size. Force root view to be that size
            measureSpec = MeasureSpec.makeMeasureSpec(rootDimension, MeasureSpec.EXACTLY);
            break;
    }
    return measureSpec;
}
~~~

根据上面的代码来看，具体其遵守如下规则，根据它的 LayoutParams 中的宽 高的参数来划分

- LayoutParams.MATCH_PARENT : 精确模式，大小就是窗口的大小
- LayoutParams.WRAP_CONTENT: 最大模式，大小不定，但是不能超过窗口的大小
- 固定大小：精确模式，大小为 LayoutParams 中指定的大小

普通 ViewGroup 的 measureChildWithMargins 方法：

~~~java
protected void measureChildWithMargins(View child, int parentWidthMeasureSpec, 
        int widthUsed, int parentHeightMeasureSpec, int heightUsed){
            final MarginLayoutParams lp = (MarginLayoutParams)child.getLayoutParams();
            /**
             * 获取 子View 的 MeasureSpec 的过程中。需要考虑 子 view 设置的一些 margin 属性。
             * 父 ViewGroup - padding - 子view 的 margin 才真正是 子 view 能利用的空间大小
             */
            final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
            mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin + widthUsed, lp.width);
            final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
            mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin + heightUsed, lp.height);

            child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
        }
~~~

> 子元素的创建不仅仅和父容器的 MeasureSpec 和 子元素本身的 LayoutParams（MarginLayoutParams是子类）有关，还和 父容器的 padding 有关系。

下面看一下 `getChildMeasureSpec`的代码：

~~~java
public static int getChildMeasureSpec(int spec, int paddding, int childDimension){
    int specMode = MeasureSpec.getMode(spec);
    int specSize = MeasureSpec.getSize(spec);

  	/**
    * 父容器 能分配给 子 view 的 maxSize
    * padding = 父容器的 padding + 子 view 的 margin
    */
    int size = Math.max(0, specSize - paddding);

    int resultSize = 0;
    int resultMode = 0;

    switch (specMode) {
        // parent has imposed exact size on us
        case MeasureSpec.EXACTLY:
            if(childDimension >= 0){
                // child want to be a specific size .So be it
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            }else if(childDimension == LayoutParams.MATCH_PARENT){
                // child want to be our size. So be it
                resultSize = size;
                resultMode = MeasureSpec.EXACTLY;
            }else if(childDimension == LayoutParams.WRAP_CONTENT){
                // child wants to determine its own size . It can't be bigger than us
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;

        // parent has imposed a maximum size on us
        case MeasureSpec.AT_MOST:
            if(childDimension >= 0){
                // child wants to be a specific size. So be it
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if(childDimension == LayoutParams.MATCH_PARENT){
                // child wants to be our size. but our size is not fixed.
                // constrain child to not be bigger than us
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }else if(childDimension == LayoutParams.WRAP_CONTENT){
                // child wants to determine its own size. It can't be
                // bigger than us
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;

        // parent asked to see how big we want to be
        case MeasureSpec.UNSPECIFIED:
            if(childDimension >= 0){
                // child want to be a specific size .. let him have it
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            }else if(childDimension == LayoutParams.MATCH_PARENT){
                // child wants to be our size ... find out how big it should be
                resultSize = 0;
                resultMode = MeasureSpec.UNSPECIFIED;
            }else if(childDimension == LayoutParams.WRAP_CONTENT){
                //child wants to determine its own size...find out how big
                // it should be
                resultSize = 0;
                resultMode = MeasureSpec.UNSPECIFIED;
            }
            break;
        default:
            break;
    }

    return MeasureSpec.makeMeasureSpec(resultSize, resultMode);

}
~~~

上面代码总结起来 父容器 和 子 view 的 MeasureSpec mode 的对应关系如下：

![](https://ww4.sinaimg.cn/large/006tKfTcgy1fel64bj76wj30hf05r74s.jpg)

### 1. Measure

>  Note：view 的测量根据 View 树，从上往下进行测量。任何一个 view 的测量，不仅仅依赖于自己(`ViewGroup.MarginLayoutParams`）还和 父控件的 MeasureSpec 相关

view 的 measure 过程由其 measure 方法来完成， measure 方法是一个 final 类型的方法，这个方法会调用 View 的 onMeasure 方法。onMeasure 的具体实现如下：

~~~java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    setMeasuredDimension(getDefaultSize(getSuggestedMinimunWidth(),
    widthMeasureSpec), getDefaultSize(getSuggestedMinimunHeight(),
    heightMeasureSpec));
}
~~~

getDefaultSize 的代码如下：

~~~java
public static int getDefaultSize(int size, int measureSpec){
    int result = size;
    int specMode = MeasureSpec.getMode(measureSpec);
    int specSize = MeasureSpec.getSize(measureSpec);

    switch (specMode) {
        case MeasureSpec.UNPECIFIED:
            result = size;
            break;
        case MeasureSpec.AT_MOST:
        case MeasureSpec.EXACTLY:
        // 这就是为什么 自定义 view 的时候 一定要重写 onMeasure 方法中 wrap_content
        // 因为默认情况下  wrap_content 和 match_parent 是一样的效果
            result = specSize;
            break;
        default:
            break;
    }
}
~~~

getSuggestedMinimunWidth 的代码如下：

~~~java
protected int getSuggestedMinimunWidth(){
    return (mBackground == null) ? mMinWidth : max(mMinWidth, mBackground.getMinmunWidth())
}

public int getMinmunWidth(){
    final int intrinsicWidth = getIntrinsicWidth();
    return intrinsicWidth > 0 ? intrinsicWidth : 0;
}
~~~



##### 1.1 getMeasureWidth 和 getWidth 的关系

​	getMeasureWidth 成为测量大小，在 onMeasure 时期就可以确定了。 getWidth 是最终大小，一般在 onLayout 时期就可以得到。两者大部分情况下是相等的。

#####  1.2 ViewGroup 的 onMeasure 过程

​	ViewGroup 并不直接实现自己的 measure 过程， 而是通过遍历子 View 从而得到自己的宽和高。ViewGroup 字段自己最终的宽和高的时候， 同时需要加入 子 view 的 margin 属性 和 自身的 padding 属性。最终是通过 `setMeasuredDimension(width, height) ` 设置自己的宽高

##### 1.3 View 的Measure 过程

​	View 根据 父 ViewGroup 的 mode 属性 和 自己的 `ViewGroup.LayoutParams.width` 和 `ViewGroup.LayoutParams.height` 得到自己的宽和高， 最终依旧是 通过 `setMeasuredDimension(width, height)` 得到自己本身的宽和高

### 2. Layout

### 3. Draw

----



## 自定义view

#### 1. 继承 view 重写 onDraw 方法

​	这种方法主要用于实现一些不规则的效果，即这种效果不方便通过布局的组合方式来达到，往往需要静态或者动态地显示一些不规则的圆形。很显然这种方式需要通过绘制的方法来重写，即 onDraw 方法。

​	这个 view 没有任何子元素。 所以处理的时候需要将其当做一个没有子元素的 ViewGroup（三部曲其实都是嵌套过程）， 由于 ViewGroup 需要自己处理 wrap_content 的情况 以及  padding 和 子 view 的 margin（当前view 没有子元素，所以不考虑）。所以**继承 view 需要自己单独处理 wrap_content 的情况（设置一个默认值） 和 padding**。 经典代码如下：

```java
package iqiyi.com.selfviewbetter;

import android.content.Context;
import android.content.res.TypedArray;
import android.graphics.Canvas;
import android.graphics.Color;
import android.graphics.Paint;
import android.support.annotation.Nullable;
import android.util.AttributeSet;
import android.util.Log;
import android.view.View;
import android.view.ViewGroup;

/**
 * Created by zhenzhen on 2017/3/16.
 */

public class CircleView extends View {

    private static String TAG = "CircleView----->";
    private static final int DEFAULT_SIZE = 300;

    private Paint mPaint;
    private int mWidth;
    private int mHeight;
    private int widthSize;
    private int heightSize;

    private int color;

    public CircleView(Context context) {
        this(context, null);
    }

    public CircleView(Context context, @Nullable AttributeSet attrs) {
        this(context, attrs, 0);
    }

    public CircleView(Context context, @Nullable AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
		// 获取自定义属性
        TypedArray array = context.obtainStyledAttributes(attrs, R.styleable.CircleView);

        color = array.getColor(R.styleable.CircleView_my_color, Color.GREEN);

        array.recycle();
        initView();
    }

    private void initView() {

        mPaint = new Paint(Paint.ANTI_ALIAS_FLAG);
        mPaint.setColor(color);
        mPaint.setStyle(Paint.Style.FILL);
    }

    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);
		//对 wrap_content 的处理
        widthSize = MeasureSpec.getSize(widthMeasureSpec);
        int widthMode = MeasureSpec.getMode(widthMeasureSpec);
        heightSize = MeasureSpec.getSize(heightMeasureSpec);
        int heightMode = MeasureSpec.getMode(heightMeasureSpec);
        //针对 wrap_content 的情况，指定一个 默认的值（如果不指定 会和 match_parent的情况一样）。
        if(widthMode == MeasureSpec.AT_MOST && heightMode == MeasureSpec.AT_MOST){
            widthSize = DEFAULT_SIZE;
            heightSize = DEFAULT_SIZE;
        }else if(widthMode == MeasureSpec.AT_MOST){
            widthSize = DEFAULT_SIZE;
        }else if(heightMode == MeasureSpec.AT_MOST){
            heightSize = DEFAULT_SIZE;
        }
		// 设置最终的宽和高
        setMeasuredDimension(widthSize, heightSize);

    }

    @Override
    protected void onLayout(boolean changed, int left, int top, int right, int bottom) {

        mWidth = right - left;
        mHeight = bottom - top;
//
//        ViewGroup.LayoutParams params = getLayoutParams();
//        params.width = mWidth;
//        params.height = mHeight;
//        setLayoutParams(params);
    }

    @Override
    protected void onDraw(Canvas canvas) {
		// 处理 padding 属性
        int paddingLeft = getPaddingLeft();
        int paddingTop = getPaddingTop();
        int paddingBottom = getPaddingBottom();
        int paddingRight = getPaddingRight();

      // 多想想 最终影响  width 和 height 的因素
        int width = getWidth() - paddingLeft - paddingRight;
        int heigth = getHeight() - paddingTop - paddingBottom;

        canvas.drawCircle(paddingLeft + width / 2, paddingTop + heigth / 2, Math.min( width, heigth) / 2, mPaint);
        invalidate();

    }
}
	
```



#### 2. 继承 ViewGroup 派生特殊的 Layout

​	这种方法主要用于实现自定义的布局，即除了 LinearLayout、RelativeLayout、FrameLayout 这几种布局之外，需要重新定义一种新布局，使得几个 view 组合在一起。

​	需要合适地处理 ViewGroup 的测量、布局这两个过程， 并同时处理子元素的测量和布局过程。



#### 3. 继承特定的 View

​	这种主要用于拓展某种已有的 View 功能，比如 TextView， 这种方式比较容易实现。**不需要自己处理 wrap_content 和 padding 等**。



#### 4. 继承特定的 ViewGroup

​	当某种效果看起来像几种 View 组合在一起的时候，可以采用这种方式。**不需要自己处理 ViewGroup 的测量和布局这两个过程。**

​	同方法 2 的区别在于：一般方法 2 能实现的，方法 4 同样能实现。方法 2 更接近于 View 的底层。



#### 5. 自定义 View 须知

##### 5.1 让 View 支持 wrap_content


​	直接继承 view 或者 viewGroup 的时候，如果不对 wrap_content 特别处理，无法达到预期效果。

##### 5.2 如果有必要， 让 view 支持 padding

​	直接继承 View 的控件，如果不在 draw 方法中处理 padding，那么 padding 属性无法起作用。

​	直接继承 ViewGroup 的控件，需要在 onMeasure 和 onLayout 中考虑 padding 和 子元素的 margin 对其造成的影响，不然将导致 padding 和子元素的 margin 失效。

##### 5.3 尽量不要在 view 中使用 handler，没必要

​	View 本身提供了 post 系列的方法，完全可以替代 Handler 的作用

##### 5.4 如果 View 中有线程或者动画，需要及时停止，参考 View#onDetachedFromWindow

​	当包含此 view 的Activity 退出 或者 当前 View 被 remove时，View的 onDetachFromWindow 方法就会被调用，此方法对应 onAttachToWindow，当包含此 View 的 Activity 启动时，View 的 onAttachToWindow 方法就会被调用。

​	用这个方法的意义在于：**避免内存泄漏**。



