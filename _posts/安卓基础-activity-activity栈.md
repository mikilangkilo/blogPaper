---
title: "安卓基础-Activity栈学习"
date: 2018-01-24 17:34:25
tag : "activity"
category : "安卓基础"
description : "本文主要是对Activity与Activity调用栈进行学习和反思"
---

# Android任务栈简介

andriod使用栈结构来管理activity

# AndroidMainifest启动模式

1. standard

每次启动都会创建新的实例，覆盖在原来的activity上面

2. singleTop

每次启动时判断栈顶是否是要启动的activity，如果是则不创建新的而直接引用这个activity。不是的话则创建一个并启动。

3. singleTask

每次启动时判断整个栈是否有要启动的activity，如果有就将其以上的activity销毁（同一个app启动这个activity是销毁，不同app启动这个activity则会创建一个新的任务栈），如果activity在后台的一个栈中，后台这个任务栈将同时切换到前台。

这种启动模式可以用来设置主activity，这样主activity启动别的activity，退出回到主activity时可以顺便销毁别的activity。

4. singleInstance

这种启动模式常用于需要与程序分离的界面，不同应用共同享用一个activity

ps:不同栈是无法使用startActivityForResult()方法来获得数据的，只可以通过intent绑定来传。

# Intent Flag启动模式

介绍一些常用的Flag

1. Intent.FLAG_ACTIVITY_NEW_TASK

使用一个新的task来启动activity，启动的每个activity都将在一个新的task中。

该flag通常使用在从service中启动activity的场景，由于在service中并不存在activity栈，所以使用该flag来创建一个新的activity栈，并创建新的activty实例。

2. FLAG_ACTIVITY_SINGLE_TOP

与singletop等同

3. FLAG_ACTIVITY_CLEAR_TOP

使用singletask模式来启动一个activity

4. FLAG_ACTIVITY_NO_HISTORY

使用这种模式启动activity，当该activity启动其他activity后，该activity就消失了，不会保存在栈中。

# 清空任务栈

可以在mainifest中activity标签中使用以下几种属性来清理任务栈

1. clearTaskOnLaunch

每次返回该activity时，都将该activity之上的所有activity清除，通过这个属性，可以让这个task每次在初始化的时候，都只有这一个activity

2. finishOnTaskLaunch

当离开这个activity所处的task，用户在返回时，该activity就会被finish掉

3. alwaysRetainTaskState

如果将这个属性设为true，那么该activity所在的task将不受任何清理命令，一直保持当前task状态