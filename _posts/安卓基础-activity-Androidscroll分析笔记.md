---
title: 安卓基础-androidscroll分析笔记
date: 2018-01-13 15:42:18
tag : "activity"
category : "安卓基础"
description : "本文主要是对androidscroll分析笔记进行学习和反思"
---

# android坐标系

在android中，将屏幕最左上角的顶点作为android坐标系的原点，往右是x轴正方向，往下是y轴正方向。

# 视图坐标系

视图坐标系原点以父视图左上角为坐标原点。

# 触控事件 - MotionEvent

```
	public static final int ACTION_DOWN = 0;//单击触摸按下动作
	public static final int ACTION_UP = 1;//单击触摸离开动作
	public static final int ACTION_MOVE = 2;//触摸点移动动作
	public static final int ACTION_CANCEL = 3;//触摸动作取消
	public static final int ACTION_OUTSIDE = 4;//触摸动作超出边界
	public static final int ACTION_POINTER_DOWN = 5;//多点触摸按下动作
	public static final int ACTION_POINTER_UP = 6;//多点离开动作
```

## ACTION_CANCEL

当你的手指（或者其它）移动屏幕的时候会触发这个事件，比如当你的手指在屏幕上拖动一个listView或者一个ScrollView而不是去按上面的按钮时会触发这个事件。

在设计设置页面的滑动开关时，如果不监听ACTION_CANCEL，在滑动到中间时，如果你手指上下移动，就是移动到开关控件之外，则此时会触发ACTION_CANCEL，而不是ACTION_UP，造成开关的按钮停顿在中间位置。

意思就是，当用户保持按下操作，并从你的控件转移到外层控件时，会触发ACTION_CANCEL，建议进行处理～

当前的手势被中断，不会再接收到关于它的记录。
推荐将这个事件作为 ACTION_UP 来看待，但是要区别于普通的 ACTION_UP

话说回来，平常还真碰不到这个事件，习惯上就直接当 ACTION_UP 处理了就

例如：上层 View 是一个 RecyclerView，它收到了一个 ACTION_DOWN 事件，由于这个可能是点击事件，所以它先传递给对应 ItemView，询问 ItemView 是否需要这个事件，然而接下来又传递过来了一个 ACTION_MOVE 事件，且移动的方向和 RecyclerView 的可滑动方向一致，所以 RecyclerView 判断这个事件是滚动事件，于是要收回事件处理权，这时候对应的 ItemView 会收到一个 ACTION_CANCEL ，并且不会再收到后续事件。

## ACTION_OUTSIDE

一个触摸事件已经发生了UI元素的正常范围之外。因此不再提供完整的手势，只提供 运动/触摸 的初始位置。dialog,popupwindow中比较常见

## ACTION_POINTER_DOWN

这个代表用户的第二根手指（之后动手的一根）触摸了屏幕，可以getactionindex获取某一根手指的数字来判断

## ACTION_POINTER_UP

同第二根手指离开了屏幕

# 获取坐标的方法

## View提供的方法（以父布局为坐标系）
getTop():自身到其父布局顶点的距离
getLeft():
getRight():
getBottom():

## MotionEvent提供的方法
getX():获取点击事件距离空间左边的距离
getY():
getRawX():获取点击事件距离整个屏幕左边的距离
getRawY():

# 实现滑动效果的7种方法

1. layout方法
在ACTION_MOVE中计算偏移量，在action_down中记录触摸点的坐标，并且在move中进行计算偏移量，然后调用view的layout方法来进行调整。

2. offsetLeftAndRight和offsetTopAndBottom
使用方法和layout一样，差别是一个需要x的偏移一个需要y的偏移

3. LayoutParams

layoutParams保留了一个view的参数，可以改变view的layoutParams然后setLayoutParams进行更改。

4. scrollto和scrollby

scrollby(offsetx，offsety)，该方法会造成所有子view移动。因此需要对view的父view使用这个方法，

5. Scroller

scroller可以通过重写view的computeScroll方法，通过获取当前滚动值，来进行不断的瞬间移动，实现整体上的平移效果
ps：computeScroll方法是不会自动调用的，只能通过invalidate() -> draw() -> computeScroll()来间接调用该方法。
之后使用startScroll即可。

6. 属性动画

7. ViewDragHelper

viewdraghelper是谷歌在support库中提供的drawerlayout和slidingpanelayout两个布局中使用的，用法较为复杂。

1. 初始化viewdraghelper

```
	mViewDragHelper = ViewDragHelper.create(this, callback);
```

2. 拦截事件

将事件传递给viewdraghelper处理
```
	@Override
	public boolean onInterceptTouchEvent(MotionEvent ev){
		return mViewDragHelper.shouldInterceptTouchEvent(ev);
	}
	@Override
	public boolean onTouchEvent(MotionEvent ev){
		mViewDragHelper.processTouchEvent(event);
		return true;
	}
```

3. 处理computeScroll()

与scroller相似的，需要处理一个computescroll()方法，因为viewdraghelper内部也是使用scroller实现平滑移动的。

```
	@Override
	public void computeScroll(){
		if(mViewdragHelper.continueSettling(true)){
			ViewCompat.postInvalidateOnAnimation(this);
		}
	}
```

4. 处理回调

```
	private ViewDragHelper.Callback callback = new ViewDragHelper.Callback(){
		@Override
		public boolean tryCaptureView(View child, int pointerId){
			return false;
		}
	}
```
使用上述回调，对child进行判断，如果是需要拖动view，就可以返回true

```
	@Override
	public int clampViewPositionVertical(View child, int top, int dy){
		return top;
	}
	@Override
	public int clampViewPositionHorizontal(View child, int left, int dx){
		return left;
	}
```

使用如上方法，来对滑动效果进行设置，返回的top和left为垂直和水平方向上面的距离。dy表示比较前一次的增量

5. 拖动结束之后，子View回到原来的位置

该效果可以通过监听action_up事件，并通过调用Scroller类来实现。
在viewdraghelper中可以重写onViewRelased()方法来实现。

```
	@Override
	public void onViewReleased(View releasedChild, float xvel, float yvel){
		super.onViewReleased(releasedChild, xvel, yvel);
		if(mMianView.getLeft()<500){
			mViewDragHelper.smoothSlideViewTo(mMainView, 0, 0);
			ViewCompat.postInvalidateOnAnimation(DragViewGroup.this);
		}else{
			mViewDragHelper.smoothSlideViewTo(mMainView, 300, 0);
			ViewCompat.postInvalidateOnAnimation(DragViewGroup.this);
		}
	}
```

这样就可以做到滑动距离小于500时回到原来的位置。

除了以上内容，还有大量的监听事件可以用来处理各种事件。
onViewCaptured():在用户触摸到view后回调
onViewDragStateChanged():在拖拽状态改变时回调
onViewPositionChanged():这个事件在位置改变时回调