## RecycleView的用法

> RecycleView 一般认为是替换 ListView。但从命名上来看，RecycleView 只负责 view 的回收与复用。至于其它的需要自己实现，实现了各个逻辑的分离。

### 参考文献

1. [如何判断滑动到顶部或者底部](http://blog.csdn.net/u011374875/article/details/51744332)
2. [判断滑动到顶部或者底部：第一个方法不好，第二个方法可以试一下](http://blog.csdn.net/u010940300/article/details/49252395#reply)

### 一、RecycleView.Adapter<ViewHolder> 适配器

~~~java
import android.content.Context;
import android.support.v7.widget.RecyclerView;
import android.view.LayoutInflater;
import android.view.View;
import android.view.ViewGroup;
import android.widget.TextView;

import com.shmj.mouzhai.recyclerviewdemo.R;

import java.util.List;

/**
 * 自定义 RecyclerView 适配器
 */

public class MyAdapter extends RecyclerView.Adapter<MyAdapter.MyViewHolder> {

    private LayoutInflater mInflater;
    private List<String> mDatas;
    private int layoutId;
    private OnItemClickListener mOnItemClickListener;

    public MyAdapter(Context context, List<String> data, int layoutId) {
        this.mDatas = data;
        this.layoutId = layoutId;
        mInflater = LayoutInflater.from(context);
    }

  /**
  * RecycleView 自身并没有 item 的点击事件，所以需要自己建立监听器 listener
  */
    public void setOnItemClickListener(OnItemClickListener listener) {
        mOnItemClickListener = listener;
    }

  /**
  * 顾名思义：此处用于创建 ViewHolder。传入 inflate 出的 view 用于找到 ViewHolder
  * 中的 view
  */
    @Override
    public MyViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
        View view = mInflater.inflate(layoutId, parent, false);
        return new MyViewHolder(view);
    }

  /**
  * 顾名思义：此处用于绑定 ViewHolder 中的 itemView 和 数据 data
  */
    @Override
    public void onBindViewHolder(final MyViewHolder holder, final int position) {
        holder.textView.setText(mDatas.get(position));
        setUpClickEvent(holder);
    }

  /**
  * 创建的点击事情监听器和自定义的 listener 关联
  */
    void setUpClickEvent(final MyViewHolder holder) {
        //设置点击事件
        if (mOnItemClickListener != null) {
            holder.itemView.setOnClickListener(new View.OnClickListener() {
                @Override
                public void onClick(View view) {
      // 注意通知刷新的方式不是 notifySetDataChanged，所以并没有刷新 adapter，因此增加的 item，index
      // 其实并没有改变，要想使其改变，需要用
      // int pos = mHolder.getLayoutPosition();
                    mOnItemClickListener.onItemClick(holder.itemView, holder.getLayoutPosition());
                }
            });

            holder.itemView.setOnLongClickListener(new View.OnLongClickListener() {
                @Override
                public boolean onLongClick(View view) {
                    mOnItemClickListener.onLongItemClick(holder.itemView, 		     holder.getLayoutPosition());
                    return false;
                }
            });
        }
    }

    @Override
    public int getItemCount() {
        return mDatas.size();
    }

    /**
     * 添加 Item
     */
    public void addItem(int position) {
        mDatas.add(position, "Insert One");
      // 注意通知刷新的方式不是 notifySetDataChanged，所以并没有刷新 adapter
        notifyItemInserted(position);
    }

    /**
     * 删除 Item
     */
    public void deleteItem(int position) {
        mDatas.remove(position);
      // 注意通知刷新的方式不是 notifySetDataChanged
        notifyItemRemoved(position);
    }

    /**
     * item 的点击回调接口
     */
    public interface OnItemClickListener {
      
        void onItemClick(View view, int position);

        void onLongItemClick(View view, int position);
    }

    class MyViewHolder extends RecyclerView.ViewHolder {

        TextView textView;

        MyViewHolder(View itemView) {
            super(itemView);

            textView = (TextView) itemView.findViewById(R.id.tv_content);
        }
    }
}
~~~



### 二、LayoutManager 样式

RecycleView 的样式是通过 LayoutManager 设置的。Listview 和 GridView 的转变，只需要一行代码。

~~~java
//设置为 ListView 的样式
        LinearLayoutManager linearLayoutManager = new LinearLayoutManager(this,
                LinearLayoutManager.VERTICAL, false);
	//listView 样式
        mRecycleView.setLayoutManager(linearLayoutManager);
	//gridview 样式
        mRecycleView.setLayoutManager(new GridLayoutManager(this, 3));
	//能水平滚动的 GridView 样式
        mRecycleView.setLayoutManager(new StaggeredGridLayoutManager(5,
                        StaggeredGridLayoutManager.HORIZONTAL));
~~~



### 三、ItemDecoration 分割线

~~~java
package iqiyi.com.recycleviewdemo;

import android.content.Context;
import android.content.res.TypedArray;
import android.graphics.Canvas;
import android.graphics.Rect;
import android.graphics.drawable.Drawable;
import android.support.v7.widget.LinearLayoutManager;
import android.support.v7.widget.RecyclerView;
import android.view.View;

/**
 * Created by zhenzhen on 2017/4/10.
 */

public class DividerItemDecoration extends RecyclerView.ItemDecoration {

    private static final int[] ATTRS = new int[]{android.R.attr.listDivider};

    private static final int HORIZONTAL_LIST = LinearLayoutManager.HORIZONTAL;
    private static final int VERTICAL_LIST = LinearLayoutManager.VERTICAL;

    private Drawable mDividerDrawable;
    private int mOrientation;

    public DividerItemDecoration(Context context, int orientation) {
        final TypedArray array = context.obtainStyledAttributes(ATTRS);
        mDividerDrawable = array.getDrawable(0);
        array.recycle();
        setOrientation(orientation);
    }


    private void setOrientation(int orientation){

        if(orientation != HORIZONTAL_LIST && orientation != VERTICAL_LIST){
            throw new IllegalArgumentException("invalid orientation");
        }

        mOrientation = orientation;
    }

    @Override
    public void onDraw(Canvas c, RecyclerView parent, RecyclerView.State state) {
      /**
      *核心思想其实是：找到 每个item 下的 drawable 的位置 然后绘制
      */
        if(mOrientation == VERTICAL_LIST){
            drawVertical(c, parent);
        }else {
            drawHorizontal(c, parent);
        }
    }

    private void drawVertical(Canvas canvas, RecyclerView parent){

        /**
         * item 在Recycleview内部  相对于父控件而言
         */
        final int left = parent.getPaddingLeft();
        final int rigth = parent.getWidth() - parent.getPaddingRight();

        final int childCount = parent.getChildCount();
        for (int i = 0; i < childCount; i++) {
            final View child = parent.getChildAt(i);
            final RecyclerView.LayoutParams params = (RecyclerView.LayoutParams) child.getLayoutParams();
            final int top = child.getBottom() + params.bottomMargin;
            final int bottom = top + mDividerDrawable.getIntrinsicHeight();
            mDividerDrawable.setBounds(left, top, rigth, bottom);
            mDividerDrawable.draw(canvas);
        }
    }

    private void drawHorizontal(Canvas canvas, RecyclerView parent){

        final int top = parent.getPaddingTop();
        final int bottom = parent.getHeight() - parent.getPaddingBottom();

        final int childCount = parent.getChildCount();
        for (int i = 0; i < childCount; i++) {
            final View child = parent.getChildAt(i);
            final RecyclerView.LayoutParams params = (RecyclerView.LayoutParams) child.getLayoutParams();
          // padding 是属于自身的 但是 margin 只是自身 view 相对于 父控件 view 的位置变化 不属于自身
            final int left = child.getRight() + params.rightMargin;
            final int right = left + mDividerDrawable.getIntrinsicWidth();

            mDividerDrawable.setBounds(left, top, right, bottom);
            mDividerDrawable.draw(canvas);
        }
    }

    @Override
    public void getItemOffsets(Rect outRect, View view, RecyclerView parent, RecyclerView.State state) {
        if(mOrientation == VERTICAL_LIST){
            outRect.set(0, 0, 0, mDividerDrawable.getIntrinsicHeight());
        }else {
            outRect.set(0, 0, mDividerDrawable.getIntrinsicWidth(), 0);
        }
    }
}
~~~



### 四、ItemAnimator

~~~java
//默认的动画效果
        mRecycleView.setItemAnimator(new DefaultItemAnimator());
~~~





