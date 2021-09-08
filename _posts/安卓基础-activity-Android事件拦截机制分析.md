---
title: 安卓基础-Android事件拦截机制分析
date: 2018-01-03 19:04:36
tag : "activity"
category : "安卓基础"
description : "本文主要是对Android渠道包打包取舍以及打包流程分析"
---
由于Android是树状结构，嵌套会导致事件发生区域重叠，针对重叠区域的处理，就叫事件拦截机制。

我们现在假设有一个嵌套结构

第一层： a-ViewGroup
第二层： b-ViewGroup
第三层： c-View

重写viewgroup的三个方法：
```
@Override
public boolean dispatchTouchEvent(MotionEvent ev){
	LogUtil.i(name+"dispatch");
	return super.dispatchTouchEvent(ev);
}
```

```
@Override
public boolean onInterceptTouchEvent(MotionEvent ev){
	LogUtil.i(name+"intecept");
	return super.onInterceptTouchEvent(ev);
}
```

```
@Override
public boolean onTouchEvent(MotionEvent ev){
	LogUtil.i(name+"onTouch");
	return super.onTouchEvent(ev);
}
```

由于view不需要重写onInterceptTouchEvent,只需要重写另外的两个事件即可。

# 点击c --正常情况

Log显示为
```
a dispatch
a intercept
b dispatch
b intercept
c dispatch
c ontouch
b ontouch
a ontouch
```

事件的传递顺序为a->b->c， 事件传递的时候是先执行dispatchTouchEvent()方法，之后在执行onInterceptTouchEvent()方法。
事件处理的顺序为c->b->a

事件传递的返回值很容易理解：True,拦截，不继续;False,不拦截，继续流程。
事件处理的返回值也类似：True,处理了，不用审核了;False,给上级处理。

初始情况下，返回值都是false。

# 让a的onInterceptTouchEvent()返回true

```
a dispatch
a intercept
a ontouch
```

很明显是a处截断了。

# 让b的onInterceptTouchEvent()返回true

```
a dispatch
a intercept
b dispatch
b intercept
b ontouch
a ontouch
```

很明显是b处截断了。

# 让c的onInterceptTouchEvent()返回true

```
a dispatch
a intercept
b dispatch
b intercept
c dispatch
c ontouch
```

解释一下ontouchevent为true的意义。
假如ontouchevent为false，代表这个ontouch事件没有结束，因此还会继续找别的事件来进行处理直到返回的ontouchevent为true。有一些view自己写了默认的ontouch事件，因此假如想继承一个有功能的view而又不想它产生ontouch效果，可以返回true来截断默认的ontouch事件。
一般情况下false可以起到添加额外效果的功能。