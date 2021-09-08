---
title: Android基础知识点
date: 2018-02-06 18:49:15
tag : "activity"
category : "安卓基础"
description : "本文主要是对Activity与Activity调用栈进行学习和反思"
---

+ 为什么建议只使用默认的构造方法来创建 Fragment？

之所以建议只使用默认的构造方式来创建fragment，是为了避免构造的过程中进行数据的设置。我们在oncreate和oncreateview的过程中可以获取bundle，这个bundle在存储fragment的时候同样可以被存储，而假如构造的模式进行设置参数的话，这些值就不会被系统存储。并且fragment的创建，其实是由fragmentmanager来初始化的，其初始化过程依靠了反射，并且是无参数反射，因此若不使用默认的构造的话会直接编译报错。

+ 为什么 Bundle 被用来传递数据，为什么不能使用简单的 Map 数据结构？

Bundle内部是由ArrayMap实现的，ArrayMap的内部实现是两个数组，一个int数组是存储对象数据对应下标，一个对象数组保存key和value，内部使用二分法对key进行排序，所以在添加、删除、查找数据的时候，都会使用二分法查找，只适合于小数据量操作，如果在数据量比较大的情况下，那么它的性能将退化。而HashMap内部则是数组+链表结构，所以在数据量较少的时候，HashMap的Entry Array比ArrayMap占用更多的内存。因为使用Bundle的场景大多数为小数据量，我没见过在两个Activity之间传递10个以上数据的场景，所以相比之下，在这种情况下使用ArrayMap保存数据，在操作速度和内存占用上都具有优势，因此使用Bundle来传递数据，可以保证更快的速度和更少的内存占用。
另外一个原因，则是在Android中如果使用Intent来携带数据的话，需要数据是基本类型或者是可序列化类型，HashMap使用Serializable进行序列化，而Bundle则是使用Parcelable进行序列化。而在Android平台中，更推荐使用Parcelable实现序列化，虽然写法复杂，但是开销更小，所以为了更加快速的进行数据的序列化和反序列化，系统封装了Bundle类，方便我们进行数据的传输。

+ 什么是 JobScheduler ？

jobscheduler提供了一种不同于alarmmanager的唤醒app的方式，其主要工作场景：应用具有您可以推迟的非面向用户的工作。/应用具有当插入设备时您希望优先执行的工作。/应用具有需要访问网络或 Wi-Fi 连接的任务。/应用具有您希望作为一个批次定期运行的许多任务。

+ 什么是 ANR ？如何避免发生 ANR ？

ANR = application not response

anr一般有三种类型：
KeyDispatchTimeout(5 seconds) --主要类型按键或触摸事件在特定时间内无响应；
BroadcastTimeout(10 seconds) --BroadcastReceiver在特定时间内无法处理完成；
ServiceTimeout(20 seconds) --小概率类型 Service在特定的时间内无法处理完成

避免的方法：

UI线程尽量只做跟UI相关的工作
耗时的工作（比如数据库操作，I/O，连接网络或者别的有可能阻碍UI线程的操作）把它放入单独的线程处理
尽量用Handler来处理UIthread和别的thread之间的交互

措施：

首先分析log
从trace.txt文件查看调用stack.
看代码
仔细查看ANR的成因（iowait?block?memoryleak?）

+ 解释一下 broadcast 和 intent 在 app 内传递消息的工作流程。

广播的注册过程 ：最终在ActivityManagerService中将远程的InnerInnerReceiver以及Intent－filter对象存储起来。 
广播的发送以及接受：内部会首先根据传入的Intent－filter 查找出匹配的广播接受者，并将改接受者放到BroadcastQueue中，紧接着系统会遍历ArrayList中的广播，并将其发送给它们对应的广播接受者，最后调用到广播接受者的onReceiver方法。

Intent传递消息过程：intent在putextra的过程中将消息放入bundle中，bundle由于实现了parcel接口，故可以进行ipc通信，最后通过目标activity从parcel中恢复状态信息，这里面的parcel完成了数据的序列化传输。

+ 当 Bitmap 占用较多内存时，你是怎么处理的？

由于内存管理上将外部内存完全当成了当前堆的一部分，也就是说Bitmap对象通过栈上的引用来指向堆上的Bitmap对象，而堆上的Bitmap对象又对应了一个使用了外部存储的native图像，也就是实际上使用的字节数组byte[]来存储的位图信息，因此解码之后的Bitmap的总大小就不能超过8M了。

设置系统的最小堆大小：
```
	int newSize = 4 * 1024 * 1024 ; //设置最小堆内存大小为4MB  
	VMRuntime.getRuntime().setMinimumHeapSize(newSize);  
	VMRuntime.getRuntime().setTargetHeapUtilization(0.75); // 设置堆内存的利用率为75%  
```

对图片的大小进行控制:
```
	BitmapFactory.Options options = new BitmapFactory.Options();  
	options.inSampleSize = 2; //图片宽高都为原来的二分之一，即图片为原来的四分之一  
	Bitmap bitmap = BitmapFactory.decodeFile("/mnt/sdcard/a.jpg",options);  
```

对bitmapfactory解码的参数进行设置：
```
	BitmapFactory.Options options = new BitmapFactory.Options();  
	options.inTempStorage = new byte[1024*1024*5]; //5MB的临时存储空间  
	Bitmap bm = BitmapFactory.decodeFile("/mnt/sdcard/a.jpg",options);  
```

+ 什么是 Dalvik 虚拟机？

每一个Android应用在底层都会对应一个独立的Dalvik虚拟机实例，其代码在虚拟机的解释下得以执行。
dalvik虚拟机使用dex文件的java文件格式，class文件中会附带着不少额外信息，dex文件对其进行精简，将所有的class文件整合一起，减少了文件尺寸和io操作的同时也提高了类的加载速度。

每一个应用都运行在一个dalvik虚拟机里面，而每一个虚拟机都有一个独立的进程空间

+ 什么是 Sticky Intent？

在MainActivity里面会有sendBroadcast和sendStickyBroacat.在ReceverActivity里面通 过BroadcastReceiver来接收这两个消息，在ReceiverActivity里是通过代码来注册Recevier而不是在 Manifest里面注册的。所以通过sendBroadcast中发出的intent在ReceverActivity不处于onResume状态是无 法接受到的，即使后面再次使其处于该状态也无法接受到。而sendStickyBroadcast发出的Intent当ReceverActivity重 新处于onResume状态之后就能重新接受到其Intent.这就是the Intent will be held to be re-broadcast to future receivers这句话的表现。就是说sendStickyBroadcast发出的最后一个Intent会被保留，下次当Recevier处于活跃的 时候，又会接受到它。

+ Android 的权限有多少个不同的保护等级？

四种，normal，dangerous，signature，signatureOrSystem

普通权限 会在App安装期间被默认赋予。这类权限不需要开发人员进行额外操作。

危险权限是在开发6.0程序时，必须要注意的。这些权限处理不好，程序可能会直接被系统干掉。危险权限以组进行划分，对该组内的一个权限授权视为对整个组进行授权，但是对开发来讲，仍然需要正对每个需要的权限进行获取，否则后期版本的变更会导致权限组划分更改。

签名级别权限和系统签名级别权限需要拥有platform级别的认证才能申请。

+ 在转屏时你如何保存 Activity 的状态？

不设置Activity的Android:configChanges时，切屏会重新调用各个生命周期，切横屏时会执行一次，切竖屏时会执行两次

设置Activity的Android:configChanges="orientation"时，切屏还是会重新调用各个生命周期，切横、竖屏时只会执行一次

设置Activity的Android:configChanges="orientation|keyboardHidden"时，切屏不会重新调用各个生命周期，只会执行onConfigurationChanged方法

但是，自从Android 3.2（API 13），在设置Activity的Android:configChanges="orientation|keyboardHidden"后，还是一样会重新调用各个生命周期的。因为screen size也开始跟着设备的横竖切换而改变。所以，在AndroidManifest.xml里设置的MiniSdkVersion和 TargetSdkVersion属性大于等于13的情况下，如果你想阻止程序在运行时重新加载Activity，除了设置"orientation"，你还必须设置"ScreenSize"。
解决方法：

AndroidManifest.xml中设置Android:configChanges="orientation|screenSize"

+ 如何实现 XML 命名空间？

常见命名空间
Android：xmlns:Android=”http://schemas.Android.com/apk/res/Android”
解析：xmlns:即xml namespace，声明我们要开始定义一个命名空间了 
Android：称作namespace-prefix，它是命名空间的名字 
http://schemas.Android.com/apk/res/Android：这看起来是一个URL，但是这个地址是不可访问的。实际上这是一个URI(统一资源标识符),所以它的值是固定不变的,相当于一个常量)。

```
	<LinearLayout xmlns:Android="http://schemas.Android.com/apk/res/Android"
    Android:layout_width="match_parent"
    Android:layout_height="match_parent">
    <TextView
        Android:layout_width="wrap_content"
        Android:layout_height="wrap_content"
        Android:layout_gravity="center"
        Android:text="New Text"
        Android:id="@+id/textView" />
	</LinearLayout>
```

亦可以写成

```
	<LinearLayout xmlns:myns="http://schemas.Android.com/apk/res/Android"
    myns:layout_width="match_parent"
    myns:layout_height="match_parent" >
    <TextView
        myns:layout_width="wrap_content"
        myns:layout_height="wrap_content"
        myns:layout_gravity="center"
        myns:text="New Text"
        myns:id="@+id/textView" />
	</LinearLayout>
```


tools:xmlns:tools=”http://schemas.Android.com/tools”

tools只作用于开发阶段
我们可以把他理解为一个工具(tools)的命名空间,它的只作用于开发阶段,当app被打包时,所有关于tools属性将都会被摒弃掉！

tools:context开发中查看Activity布局效果
context的用法，在后面跟一个Activtiy的完整包名,它有什么作用呢?

当我们设置一个Activity主题时,是在AndroidManifest.xml中设置中,而主题的效果又只能在运行后在Activtiy中显示

使用context属性, 可以在开发阶段中看到设置在Activity中的主题效果

tools:context=”com.littlehan.myapplication.MainActivity”

在布局中加入这行代码,就可以在design视图中看到与MainActivity绑定主题的效果。

tools:layout开发中查看fragment布局效果
当我们在Activity上加载一个fragment时，是需要在运行后才可以看到加载后的效果,有没有方法在测试阶段就在布局预览窗口上显示呢?

答案是有的,借助layout属性,例如,在布局中加入这样一行代码: 
tools:layout=@layout/yourfragmentlayoutname 
这样你的编写的fragment布局就会预览在指定主布局上了

app:xmlns:app=”http://schemas.Android.com/apk/res-auto”

app命名空间为用户自定义，通过attrs进行设置，然后通过自定义view进行解析。


+ Application 和 Activity 的 Context 对象的区别

这是两种不同的context，也是最常见的两种.第一种中context的生命周期与Application的生命周期相关的，context随着Application的销毁而销毁，伴随application的一生，与activity的生命周期无关.第二种中的context跟Activity的生命周期是相关的，但是对一个Application来说，Activity可以销毁几次，那么属于Activity的context就会销毁多次.至于用哪种context，得看应用场景，个人感觉用Activity的context好一点，不过也有的时候必须使用Application的context.application context可以通过
Context.getApplicationContext或者Activity.getApplication方法获取.

还有就是，在使用context的时候，小心内存泄露，防止内存泄露，注意一下几个方面：

　1. 不要让生命周期长的对象引用activity context，即保证引用activity的对象要与activity本身生命周期是一样的

　2. 对于生命周期长的对象，可以使用application context

　3. 避免非静态的内部类，尽量使用静态类，避免生命周期问题，注意内部类对外部对象引用导致的生命周期变化

