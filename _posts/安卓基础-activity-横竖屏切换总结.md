---
title: 安卓基础-Android横竖屏切换总结
date: 2018-02-23 00:08:15
tag : "activity"
category : "安卓基础"
description : "本文是学习横竖屏切换时的笔记"
---

昨天看网上一个关于横竖屏切换的总结，里面说不设置activity的configChanges时，在竖屏切换横屏时会进行两次生命周期的加载，看到这个就很疑惑，为什么两次呢？结果试了一下，大相径庭。

设备 samsung-galaxy-mega2

+ 不设置configChanges时，从横屏切换到竖屏时的生命周期

onPause -> onSaveInstanceState -> onStop -> onDestroy -> onCreate -> onStart -> onRestoreInstanceState -> onResume

很明显，是一次activity的销毁和重建，onPause之后便进行了onSaveInstanceState 而onResume之前也进行了onRestoreInstanceState。很正常

+ 不设置configChanges时，从竖屏切换到横屏时的生命周期

onPause -> onSaveInstanceState -> onStop -> onDestroy -> onCreate -> onStart -> onRestoreInstanceState -> onResume

和1状态一毛一样。并没有走两次。

+ 设置configChanges = "orientation"时，从横屏切换到竖屏时的生命周期

onPause -> onSaveInstanceState -> onStop -> onDestroy -> onCreate -> onStart -> onRestoreInstanceState -> onResume

也是一毛一样

+ 设置configChanges = "orientation"时，从竖屏切换到横屏的生命周期

onPause -> onSaveInstanceState -> onStop -> onDestroy -> onCreate -> onStart -> onRestoreInstanceState -> onResume

也是一毛一样

这就代表着configChanges = "orientation"没有任何作用

+ 设置configChanges = "orientation|keyboardHidden"时，从横屏切换到竖屏 && 竖屏切换到横屏的生命周期

onPause -> onSaveInstanceState -> onStop -> onDestroy -> onCreate -> onStart -> onRestoreInstanceState -> onResume

依旧一毛一样

+ 设置configChanges = "orientation|keyboardHidden|screensize"时，从横屏切换到竖屏 && 竖屏切换到横屏的生命周期

只走了onConfigurationChanged这一个方法。

+ 只设置configChanges = "screensize" || 只设置configChanges = "orientation",从横屏切换到竖屏 || 竖屏切换到横屏的生命周期

onPause -> onSaveInstanceState -> onStop -> onDestroy -> onCreate -> onStart -> onRestoreInstanceState -> onResume

代表只设置一个是无效的

+ 只设置configChanges = "keyboardHidden|screensize",从横屏切换到竖屏 && 竖屏切换到横屏的生命周期

onPause -> onSaveInstanceState -> onStop -> onDestroy -> onCreate -> onStart -> onRestoreInstanceState -> onResume

仍旧无效

+ 只设置configChanges = "orientation|screensize",从横屏切换到竖屏 || 竖屏切换到横屏的生命周期

只走了onConfigurationChanged这一个方法。


###### 总结

若activity的configChanges没有设置，或者设置却没有同时设置 orientation 和 screensize时，会导致activity销毁重建。

而若是设置了configChanges = "orientation|screensize",则不会销毁，只会走onConfigurationChanged。

和网上的内容大相径庭。若是偏信网上的内容，这部分的知识点就是错误的理解。

###### 心得

实践出真知！！！！！不能过于相信网上的内容，要自己动手实践一下。
