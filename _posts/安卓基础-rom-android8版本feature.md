---
title: 安卓基础-Android8.0-feature一览
date: 2018-03-22 22:16:52
tag : "Rom"
category : "安卓基础"
description : "本文主要是学习Android8.0各版本feature"
---

Android p出来了，是时候了解一波新特性了。

# Notification Channels

这个是从Android 8.0 引入的概念，目的是提供统一的系统来帮助用户管理通知，开发者可以为需要发送的每个不同的通知类型创建一个通知渠道。还可以创建通知渠道来反映应用的用户做出的选择。例如，可以为聊天应用的用户创建的每个聊天组建立单独的通知渠道。

假如不使用channel的话，会不给发通知。

## 创建流程

- 创建渠道

```
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            NotificationManager manager = (NotificationManager) context.getSystemService(Context.NOTIFICATION_SERVICE);

            NotificationChannel mChannel = new NotificationChannel("channel_01",
                    "消息推送", NotificationManager.IMPORTANCE_DEFAULT);
            manager.createNotificationChannel(mChannel);
        }
```

- 构建通知

```
		Context context = DJApplication.getInstance();
        Notification.Builder builder = new Notification.Builder(context);
        builder.setTicker("开始下载");
        builder.setSmallIcon(R.mipmap.ic_launcher);
        builder.setLargeIcon(BitmapFactory.decodeResource(DJApplication.getInstance().getResources(), R.mipmap.ic_launcher));
        builder.setAutoCancel(true);
        PendingIntent pIntent = PendingIntent.getActivity(context, 0, new Intent(), PendingIntent.FLAG_UPDATE_CURRENT);
        builder.setContentTitle("下载中");
        builder.setContentIntent(pIntent);
        builder.setContentText(text);
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            builder.setChannelId("channel_01");
        }
        manager.notify(1,  builder.build());
```

# 安装权限问题

这次更新之后，下载和安装权限分离了，安装需要使用

```
<uses-permission Android:name="Android.permission.REQUEST_INSTALL_PACKAGES" />
```

该权限可以确保下载完成之后吊起安装程序