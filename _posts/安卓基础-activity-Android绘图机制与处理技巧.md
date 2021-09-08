---
title: 安卓基础-Android绘图机制与处理技巧
date: 2018-01-16 21:53:31
tag : "activity"
category : "安卓基础"
description : "本文主要是对Android绘图机制与处理技巧进行学习和反思"
---

本章用于做群英传第六章《Android绘图机制与处理技巧》的学习笔记。主要是整理自己不熟悉的知识点。

# 屏幕的尺寸信息

1. 屏幕大小

屏幕大小指对角线的长度，一般使用寸来度量，寸指英寸，一英寸为2.54cm

2. 分辨率

分辨率指手机屏幕的像素点个数。1920*1080指宽有1920个像素点，高有1080个像素点

3. ppi

pixel per inch，每英寸像素，又称为dpi，为对角线的像素点除以屏幕大小得到。

# 系统屏幕密度

根据dpi大小来进行设置，系统定义了几个标准的dpi值。

120: ldpi
160: mdpi
240: hdpi
320: xhdpi
480: xxhdpi

# 独立像素密度 dp

在mdpi，即dpi为160时， 1dp = 1px。之后顺推即可，xxhdpi为160的3倍，即1dp = 3px

dp涉及到像素工具类的使用。

```
public class DisplayUtil{

	public static int px2dp(Context context, float px){
		final float scale = context.getResource().getDisplayMetrics().density;
		return (int)(px/scale + 0.5f);
	}

	public static int dp2px(Context context, float dp){
		final float scale = context.getResource().getDisplayMetrics().density;
		return (int)(dp*scale + 0.5f);
	}

	public static int px2sp(Context context, float px){
		final float fontScale = context.getResource().getDisplayMetrics().scaledDensity;
		return (int)(px/fontScale + 0.5f);
	}

	public static int sp2dp(Context context, float sp){
		final float fontScale = context.getResource().getDisplayMetrics().scaledDensity;
		return (int)(sp*fontScale + 0.5f);
	}

	//以上为使用公式进行换算的
	//还可以使用TypedValue进行换算

	public static int dp2px(int dp){
		return (int)TypedValue.applyDimension(
			TypedValue.COMPLEX_UNIT_DIP,
			dp,
			getResources().getDisplayMetrics());
	}

	public static int sp2px(int sp){
		return (int)TypeValue.applyDimension(
			TypedValue.COMPLEX_UNIT_SP,
			sp,
			getResources().getDisplayMetrics());
	}

}
```

# 2D绘图基础

2d绘图即使用系统提供的Canvas对象来提供绘图方法，该章主要是复习一些常用的api。

1. paint

setAntiAlias():设置抗锯齿效果
setColor():设置画笔的颜色
setARGB():设置画笔的啊a,r,g,b值
setAlpha():设置画笔透明度
setTextSize():设置字体的尺寸
setStyle():设置画笔的风格（空心或者实心）
setStrokeWidth():设置空心边框的宽度

2. canvas

canvas.drawPoint(x, y, paint): 绘制点

canvas.drawLine(startX, startY, endX, endY, paint): 绘制直线

```
	float[] pts = {
		startX1, startY1, endX1, endY1,
		... ...
		startXn, startYn, endXn, endYn
	};
	canvas.drawLines(pts, paint);
	//画多条直线
```

canvas.drawRect(left, top, right, bottom, paint): 绘制矩形

canvas.drawRoundRect(left, top, right, bottom, radiusX, radiuxY, paint):绘制圆角矩形

canvas.drawCircle(circleX, circleY, radius, paint):绘制圆

canvas.drawArc(left, top, right, bottom, startAngle, sweepAngle, useCenter, paint):绘制弧形，扇形，区别在于useCenter

canvas.drawOval(left, top, right, bottom, paint):画椭圆

canvas.drawText(text, startX, startY, paint):绘制文本

canvas.drawPosText(text, new float[]{X1, Y1, X2, Y2... ... Xn, Yn}, paint):在制定位置绘制文本

canvas.drawPath(path, paint):绘制路径

# Android xml绘图

1. Bitmap

在xml中使用bitmap
```
<?xml version="1.0" encoding="utf-8"?>
<bitmap xmlns:Android="http://schemas.Android.com/apk/res/Android"
	Android:src="@drawable/ic_launcher"/>
```

2. Shape

```
<shape xmlns:Android:"http://schemas.Android.com/apk/res/Android"
	Android:shape=["rectangle" | "oval" | "line" | "ring"]>
	//默认为rectangle
	<corners //shape = "rectangle" 有用
		// 半径，会被后面的单个半径属性覆盖，默认为1dp
		Android:radius="integer"
		Android:topLeftRadius="integer"
		Android:topRightRadius="integer"
		Android:bottomLeftRadius="integer"
		Android:bottomRightRadius="integer"/>
	<gradient //渐变
		Android:angle="integer"
		Android:centerX="integer"
		Android:centerY="integer"
		Android:centerColor="integer"
		Android:endColor="color"
		Android:gradientRadius="integer"
		Android:startColor="color"
		Android:type=["linear"| "radius" | "sweep"]
		Android:useLevel=["true" | "false"]/>
	<padding
		Android:left="integer"
		Android:top="integer"
		Android:right="integer"
		Android:bottom="integer"/>
	<size // 指定大小，一般用在imageview配合scaletype属性使用
		Android:width="integer"
		Android:height="integer"/>
	<solid // 填充颜色
		Android:color="color"/>
	<stroke //指定边框
		Android:width="integer"
		Android:color="color"
		Android:dashWidth="integer" //虚线宽度
		Android:dashGap="integer" // 虚线间隔宽度
		/>
</shape>
```

3. Layer

layer中可以使用层级来进行叠加，主要是用item，item可以使用drawable，也可使用shape

4. Selector

Selector用于帮开发者实现静态绘图中的事件反馈，通过不同的事件设置不同的图像。

```
<?xml version="1.0" encoding="utf-8" ?>
<selector xmlns:Android="http://schemas.Android.com/apk/res/Android">
	//默认时的背景图片
	<item Android:drawable="@drawable/x1"/>
	//没有焦点时的图片
	<item Android:state_window_focused="false" Android:drawable="@drawable/x2"/>
	//非触摸模式下获得焦点并单击时的背景图片
	<item Android:state_focuse="true" Android:state_pressed="true" Android:drawable="@drawable/x3"/>
	//触摸模式下单击时的背景图片
	<item Android:state_focuse="false" Android:state_pressed="true" Android:drawable="@drawable/x4"/>
	//选中时的背景图片
	<item Android:state_selected="true" Android:drawable="@drawable/x5"/>
	//获得焦点时的背景图片
	<item Android:state_focused="true" Android:drawable="@drawable/x6"/>
</selector>
```

以上可以用于制作view的触摸反馈。

# Android 绘图技巧

之上的是基本绘图技巧，之下的是常用绘图技巧

1. Canvas

Canvas.save(): 将之前的所有绘制图像保存起来，之后的操作就好像在一个新的图层上面操作一样。

Canvas.restore(): 用于合并图层，可以用于将save之后绘制的所有图像与save之前的图像合并

Canvas.translate():调用translate(x, y)操作可以将原点(0, 0)移动到(x, y)之后的所有操作都将以(x, y)为原点执行

Canvas.rotate():调用rotate(degree)之后可以将canvas调转一定的角度。

2. Layer图层

Android通过调用saveLayer()方法，saveLayerAlpha()方法将一个图层入栈，使用restore()方法,restoreToCount()方法将一个图层出栈。出栈的时候，会把图像绘制到上层Canvas上。

# 画笔特效处理

1. PorterDuffXferMode

该模式控制的是两个图像间的混合显示模式。dst是先画的图形，src是后画的图形。

2. Shader

shader被称为着色器，渲染器。可以用来实现一系列的渐变、渲染效果。

其中shader包括：
BitmapShader   ---  位图shader
LinearGradient ---  线性shader
RadialGradient ---  光束shader
SweepGradient  ---  梯度shader
ComposeShader  ---  混合shader

shader的作用就是通过paint对画布进行指定的bitmap的填充，填充时有三种模式

CLAMP  --- 拉伸的是图片最后的那一个像素，不断重复
REPEAT --- 横向纵向不断重复
MIRROR --- 横向不断翻转重复，纵向不断翻转重复

3. PathEffect

patheffect是指用各种笔触效果来绘制一个路径。

CornerPathEffect:   将拐角处变的圆滑
DiscretePathEffect:	使用这个之后线段上会出现很多杂点
DashPathEffect:		使用这个可以绘制虚线，用一个数组来设置各个点之间的间隔，此后绘制虚线时就重复这样的间隔进行绘制，另一个参数phase可以用来绘制时数组的一个偏移量，可以通过设置值来实现路径的动态效果。
PathDashPathEffect: 与dashpatheffect类似，不过功能更强大，可以设置点的图形，例如方形点的虚线，圆形点的虚线
ComposePathEffect   可以通过composepatheffect来组合patheffect，这个方法的功能就是将任意的两种路径特性组合起来形成一个新的效果。

# SurfaceView

1. surfaceview 和view的区别

对view来说，Android系统通过发出VSYNC信号来进行屏幕的重绘，刷新的间隔为16ms，如果在16ms内view完成了所需要执行的所有操作，那么用户在视觉上就不会产生卡顿的感觉。但是执行的操作逻辑太多，在需要频繁刷新的界面上，就会阻塞主线程，因此Android提供了surfaceview。

view主要适用于主动更新的情况下，而surfaceview主要适用于被动更新，例如频繁刷新
view在主线程中对画面进行刷新，surfaceview则通常会通过一个子线程来进行页面的刷新
view在绘图时没有使用双缓冲机制，而surfaceview在底层实现机制中就已经实现了双缓冲机制

2. surfaceview的使用

surfaceview使用有一套模版可以根据模版进行操作

## 创建surfaceview：

创建自定义的surfaceview需要继承SurfaceView,并实现两个接口-- surfaceholder callback和 runnable

```
	public class SurfaceViewTemplate extends SurfaceView implements SurfaceHolder.Callback, Runnable
```

之后需要实现这两个接口

## 初始化SurfaceView

自定义surfaceview的构造方法中，需要对surfaceview进行初始化，在自定义的surfaceview中，通常需要定义三个成员变量

```
	//SurfaceHolder
	private SurfaceHolder mHolder;
	//用于绘图的Canvas
	private Canvas mCanvas;
	//子线程标识位
	private boolean mIsDrawing;
```

canvas用于绘图，标志位用于控制子线程。

## 使用SurfaceView

通过surfaceholder对象的lockCanvas()方法，就可以得到当前的canvas绘图对象，接下来就可以与在view中绘制一样的操作了。
每次调用这个方法获取到的canvas都是继续上次的对象。清屏可以使用drawColor()操作。

绘制的时候，在surfaceCreated()中开启子线程进行绘制，而子线程使用一个while(mIsDrawing)的循环来不停的进行绘制。
绘制的具体过程，使用lockCanvas()方法获得的Canvas对象进行绘制，并通过unlockCanvasAndPost(mCanvas)方法对画布内容进行提交。