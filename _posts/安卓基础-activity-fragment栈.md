---
title: 安卓基础-针对activity栈启动模式进行fragment栈的引申
date: 2018-05-03 23:30:29
tag : "activity"
category : "安卓基础"
description : "本文主要是对Activity与Activity调用栈进行学习和反思"
---

“one activity for the whole app, you can use fragments, just don't use the backstack with fragments”

--- jake·warton

安卓开发有其独特的魅力，相对于每个页面不断的切换，事实上前端开发从安卓的角度来看可以有不同的作为。

比如说jake·warton所说的只使用一个activity窗口，外加很多个碎片，来组成一个app。这句话很好理解，可是后面的这半句话是什么意思呢？主要是fragment的后台管理栈比较不容易操控。

所以我们需要了解一下如何操控这个栈。这次就从activity的启动模式来分析。

- standard

默认的启动模式，对fragment来讲，在basefragment中抽取activity的stackedfragments，每次新建一个fragment时，创建一个新的加入到栈顶即可。

- singleTop

在栈顶的话，启动还是会转到自己，而不在栈顶的话就会新建。这个属性对fragment来讲不怎么实用，没怎么遇到过这种情形，如果要用的话，就判断之前的栈顶是否和新传入的相同，相同的话，就仍然返回的自身，不同就创建一个。

- singleTask

在栈内的话，启动一个实例，就会启动栈内的。这个属性比较好用，对fragment来讲，可以用这个来做很多事情，包括回去的时候直接pop起到了cleartop的效果。

开发中还遇到相似的，一次传入三到四个fragment，如果有其中的fragment在栈上方，立即返回他，中断其余判断。这种操作其实就是遍历栈而已，不过能够在fragment多复用的情况下，找到当前回归路径，省了很多其他的事情。

- singleinstance

每次开辟一个新的栈，这个在fragment中没用，因为只有唯一的一个栈在activity中。



- FLAG_ACTIVITY_EXCLUDE_FROM_RECENTS

对activity来讲，新的Activity不会在最近启动的Activity的列表中保存。对fragment来讲，就是新启动的不入栈，可以考虑设计一个固定大小的栈的时候，只有固定的几个fragment可以用于入栈，其余的都不入。不过这个设计要考虑一下，返回哪些栈。

实际效果可能就是切了很多个页面，最后一个回退，回退到了很久之前的一个页面。使用情景比较局限。

- FLAG_ACTIVITY_FORWARD_RESULT

这个是startactivityforresult的标志位，同样的也可以设计一个startfragmentforresult。事实上也就是在进入这个fragment的时候，basefragment记录一下resultcode的值，然后回退的时候，将这个result值传入到栈顶即可。

- FLAG_ACTIVITY_NO_HISTORY

单纯的不入栈的操作。

- FLAG_ACTIVITY_REORDER_TO_FRONT

该标志位是用于启动时挪动activity栈的，对fragment同样可以设计，不过较为复杂，可以从栈内取出，重新排序之后插入。比较不好的是可能会遗漏状态。

- FLAG_ACTIVITY_NEW_TASK

同 singinstance

- FLAG_ACTIVITY_CLEAR_WHEN_TASK_RESET

对activity来讲，是当新进入的activity携带这个标志时，就会清理栈。fragment同样可以设计，若有个携带该标记的进入，也可以清空。虽然我目前仍然是使用老的回到最起初的fragment，不过效果可能没这个设计的好。


大致就这些。针对fragment设计一个单独的有特征的栈，我个人觉得是使用fragment代替activity的第一步。
