---
title: 安卓基础-使用浏览器唤起app
date: 2018-01-06 18:38:38
tag : "webview"
category : "安卓基础"
description : "本文主要是学习使用浏览器唤起app"
---
浏览器唤起app，其实很简单，是使用manifest中注册scheme的方式来设置。

manifest中注册如下

```
<intent-filter>
    <action Android:name="Android.intent.action.VIEW"/>
    <category Android:name="Android.intent.category.DEFAULT" />
    <category Android:name="Android.intent.category.BROWSABLE" />
    <data Android:scheme="myapp" Android:host="jp.app" Android:pathPrefix="/openwith"/>
</intent-filter>
```

这段代表了唤起方式可以由浏览器唤起，data方面重要的是写scheme，之后的host和pathprefix都是无关紧要的。

assets文件夹里面放静态html页面，使用webview来load。

```
<a href="myapp://jp.app/openwith?name=zhangsan&age=26">启动应用程序</a>
```

启动的activity里面接受intent
```
if(Intent.ACTION_VIEW.equals(action)){
    Uri uri = i_getvalue.getData();
    if(uri != null){
        String name = uri.getQueryParameter("name");
        String age= uri.getQueryParameter("age");
        Log.d(getClass().getName(),name+age);
	}
}
```

可以接收到intent里面包含的信息。

针对昨天的问题，如何从html中get一个接口的数据，然后返回吊起activity，可以直接使用script来fetch（url），针对内容对唤起的activity后缀加内容。

目前关于这个问题还是涉及到跨域的情况，张凯说使用301转发，周宇说跨域需要接口支持。我也不知道该怎么办了。周一去公司的时候查一下接口能不能跨域再说。