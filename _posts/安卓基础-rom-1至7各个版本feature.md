---
title: 安卓基础-Android1.0-7.0各版本feature
date: 2018-03-23 10:03:03
tag : "Rom"
category : "安卓基础"
description : "本文主要是学习Android1.0-7.0各版本feature"
---

# 5.0 - api 21

## 新出material design
## view增加了z属性

在5.0之前，我们如果想给view添加阴影效果，以体现其层次感，通常的做法是给view设置一个带阴影的背景图片，现在，我们只需要简单的修改view的Z属性，就能让其具备阴影的层次感。

Z属性会扩大view的显示区域，如果它的大小大于或等于父视图的大小，那么它的阴影效果就无法显示了，view并不会因为z属性而把自身缩小腾出空间显示阴影。

Z属性不仅影响着view的阴影效果，还影响着view的绘制顺序，在同一个父view内部，Z属性越小，绘制的时机就越早。也就是优先被绘制，而z属性越大，则绘制时间越晚，后绘制的将会遮盖住先绘制的，只有Z属性相同，才按照添加的顺序绘制。

## view增加了轮廓

在xml布局中，可以通过Android:outlineProvider来指定轮廓的判定方式：

none 即使设置了Z属性，也不会显示阴影
background 会按照背景来设置阴影形状
bounds 会按照View的大小来描绘阴影
paddedBounds 和bounds类似，不过阴影会稍微向右偏移一点

在代码中，我们可以通过setOutlineProvider来指定一个View的轮廓：
```
ViewOutlineProvider viewOutlineProvider = new ViewOutlineProvider() {
    public void getOutline(View view, Outline outline) {
        // 可以指定圆形，矩形，圆角矩形，path
        outline.setOval(0, 0, view.getWidth(), view.getHeight());
    }
};
View.setOutlineProvider(viewOutlineProvider );
```

## view的裁剪

给View指定轮廓，可以决定阴影的显示形状，如果给View指定一个小于自身大小的轮廓，正常情况下阴影会被View遮住，这个时候View的显示内容并没有因为轮廓的缩小而缩小。

如果想根据轮廓来缩小一个View，则可以通过剪裁。如果一个View指定了轮廓，调用setClipToOutline方法，就可以根据轮廓来剪裁一个View。想要剪裁轮廓，必须要给View先指定轮廓，并且轮廓是可以被剪裁的，目前只有圆形，矩形，圆角矩形支持剪裁，可以通过outline.canClip()来判断一个轮廓是否支持剪裁。

Path剪裁不会改变View的大小，但是如果Path的范围比View要的bounds要小，则剪裁后会改变View的位置，位置偏移和Z属性有关，这可能是一个BUG，view的设计者可能在绘制阴影时根据轮廓偏移了画布，而在绘制完后忘记把画布还原了。

剪裁不会改变View的测量大小和布局大小，也不会改变View的触摸区域，剪裁只是在onDraw的时候对画布做了剪裁处理，剪裁也不同于scale，scale是调整画布matrix的缩放属性，调整后，View仍然能完整显示，而剪裁是缩小画布的剪裁区域，剪裁后我们只能看到View的一部分。

试图给View一个比较大的轮廓进行剪裁也是不成功的，实验证明剪裁后的View只能比原有体积小，扩大轮廓只会扩大轮廓的绘制区域。

剪裁是一个非常消耗资源的操作，我们不应该用此来做动画效果，如果要实现这样的动画，可以使用Reveal Effect

## tint属性

tint属性是一个颜色值，可以对图片做颜色渲染，我们可以给view的背景设置tint色值，给ImageView的图片设置tint色值，也可以给任意Drawable或者NinePatchDrawable设置tint色值。

在应用的主题中也可以通过设置 Android:tint 来给主题设置统一的颜色渲染。

tint的渲染模式有总共有16种，xml文件中可以使用6种，代码中我们可以设置16种，渲染模式决定了渲染颜色和原图颜色的取舍和合成规则：

PorterDuff.Mode.CLEAR 所绘制不会提交到画布上。
PorterDuff.Mode.SRC 显示上层绘制图片
PorterDuff.Mode.DST 显示下层绘制图片
PorterDuff.Mode.SRC_OVER 正常绘制显示，上下层绘制叠盖。
PorterDuff.Mode.DST_OVER 上下层都显示。下层居上显示。
PorterDuff.Mode.SRC_IN 取两层绘制交集。显示上层。
PorterDuff.Mode.DST_IN 取两层绘制交集。显示下层。
PorterDuff.Mode.SRC_OUT 取上层绘制非交集部分。
PorterDuff.Mode.DST_OUT 取下层绘制非交集部分。
PorterDuff.Mode.SRC_ATOP 取下层非交集部分与上层交集部分
PorterDuff.Mode.DST_ATOP 取上层非交集部分与下层交集部分
PorterDuff.Mode.XOR 取两层绘制非交集。两层绘制非交集。
PorterDuff.Mode.DARKEN 上下层都显示。变暗
PorterDuff.Mode.LIGHTEN 上下层都显示。变亮
PorterDuff.Mode.MULTIPLY 取两层绘制交集
PorterDuff.Mode.SCREEN 上下层都显示。

通过tint属性处理后的图片会和原图显示出不一样的颜色，我们可以通过这种方式利用一张图片做出图片选择器的效果，让控件在按压状态下显示另外一种颜色:
```
通过给图片设置tint色生成另外一种图片
<bitmap xmlns:Android="http://schemas.Android.com/apk/res/Android"
        Android:src="@drawable/ring"
        Android:tintMode="multiply"
        Android:tint="#5677fc" />
利用新的图片生成图片选择器
<selector xmlns:Android="http://schemas.Android.com/apk/res/Android">
        <item Android:drawable="@drawable/tint_bitmap" Android:state_pressed="true"/>
        <item Android:drawable="@drawable/ring" />
</selector>
```

## Palette调色版

Palette调色板，可以很方便的让我们从图片中提取颜色。并且可以指定提取某种类型的颜色。
Vibrant 鲜艳的
Vibrant dark鲜艳的暗色
Vibrant light鲜艳的亮色
Muted 柔和的
Muted dark柔和的暗色
Muted light柔和的亮色

对图片取色是一个比较消耗性能的操作，其内部会对图片的像素值进来遍历以分析对比，所以我们要在异步线程中去完成。

```
如果操作本来就属于后台线程，可以使用：
Palette p = Palette.generate(Bitmap bitmap);
如果在主线程中，我们可以使用异步的方式：
Palette.generateAsync(bitmap, new Palette.PaletteAsyncListener() {
        public void onGenerated(Palette palette) {  }
});
```

当操作完成后或者异步回调后，我们就可以使用以下方式来获取对应的色值了，并且可以在没有获取到的情况下之指定默认值：
```
p.getVibrantColor(int defaultColor);
p.getDarkVibrantColor(int defaultColor);
p.getLightVibrantColor(int defaultColor);
p.getMutedColor(int defaultColor);
p.getDarkMutedColor(int defaultColor);
p.getLightMutedColor(int defaultColor);
```

在使用palette之前，bitmap提供获取指定位置的像素值：
```
bitmap.getPixel(x,y)
```

但是该方式只能获取某一点的像素值，palette是对整个bitmap的所有像素值进行分析，并选出几个像素占比比较多的像素值，这样选择出来的色值更符合图片的整体色值。

## vector矢量图

矢量图也称为面向对象的图像或绘图图像，是计算机图形学中用点、直线或者多边形等基于数学方程的几何图元表示的图像。矢量图形最大的优点是无论放大、缩小或旋转等不会失真；最大的缺点是难以表现色彩层次丰富、逼真的图像效果。

Android L开始支持矢量图，我们可以用它来处理一些图形简单的icon，方便我们的适配。

Android L中对矢量图的支持是通过xml文件构建，通过矢量图的path描述来生成一个矢量图，对应的java对象为VectorDrawable。

下面是官方文档提供的一个矢量图，利用改文件，我们可以创建一个随意放大缩小都不会失真的心形。

```
<vector xmlns:Android="http://schemas.Android.com/apk/res/Android"
        Android:height="300dp"
        Android:width="300dp"
        Android:viewportHeight="40"
        Android:viewportWidth="40">
        <path Android:fillColor="#ff00ff"
                Android:pathData="M20.5,9.5
                        c-1.955,0,-3.83,1.268,-4.5,3
                        c-0.67,-1.732,-2.547,-3,-4.5,-3
                        C8.957,9.5,7,11.432,7,14
                        c0,3.53,3.793,6.257,9,11.5
                        c5.207,-5.242,9,-7.97,9,-11.5
                        C25,11.432,23.043,9.5,20.5,9.5z"/>
</vector>
```

矢量图的pathData数据就是用来描述矢量图的数学公式，其含义如下表：

命令类型	使用描述	代表含义					举例说明
移动指令	M x,y	M移动绝对位置				M 100,240
移动指令	m x,y	m移动相对于上一个点		m 100,240
绘制		L 或 l	从当前点绘制直线到指定点	L 100,100
绘制		H 或 h	水平直线					h 100
绘制		V 或 v	垂直直线					v 100
绘制		C 或 c	三次方程式贝塞尔曲线		C 100,200 200,400 300,200
绘制		Q 或 q	二次方程式贝塞尔曲线		Q 100,200 300,200
绘制		S 或 s	平滑三次方程式贝塞尔曲线	S 100,200 200,400 300,200
绘制		T 或 t	平滑二次方程式贝塞尔曲线	T 100,200 300,200
绘制		A 或 a	椭圆						A 5,5 0 0 1 10,10
关闭指令	Z 或 z	将图形的首、尾点用直线连接	Z
填充		F0	EvenOdd 填充规则	
填充		F1	Nonzero 填充规则	

通过path命令来进行简单的图形还是可行的，但是复杂的图形我们就需要借助工具来生成了，比如使用 Expression Design，就可以直接粘贴来自其它软件的矢量图形，然后选择导出，导出时做如后选择：文件->导出->导出属性->格式->XAML Silverlight 画布，即可得到XAML格式的矢量图形，也就是Path。

更多矢量图学习可参考：http://www.w3.org/TR/SVG11/paths.html#PathData 我们可以访问http://editor.method.ac 在线制作矢量图并导出path。


## 新增widget

## RecyclerView

RecyclerView是ListView的升级版，它具备了更好的性能，且更容易使用。和ListView一样，RecyclerView是用来显示大量数据的容器，并通过复用有限数量的View，来提高滚动时的性能。当你的视图上的元素经常动态的且有规律的改变时候，可以使用RecyclerView控件。

与ListView不同的是RecyclerView不再负责布局，只专注于复用机制，布局交由LayoutManager来管理。 RecyclerView仍然通过Adapter来获取需要显示的对象。

要使用RecyclerView组件，创建Adapter不再继承自BaseAdapter，而是应该继承自RecyclerView.Adapter类，并且最好指定一个继承自RecyclerView.ViewHolder的范型，Adapter不再要求你返回一个View，而是一个ViewHolder。

继承自Adapter后，需要实现3个抽象方法：

```
// 当RecyclerView需要一个ViewHolder时会回调该方法，如果有可复用的View则该方法不会得倒回调
public RecyclerView.ViewHolder onCreateViewHolder(ViewGroup viewGroup, int i)；
// 当一个View需要出现在屏幕上时，该方法会被回调，你需要在该方法中根据数据来更改视图
public void onBindViewHolder(RecyclerView.ViewHolder viewHolder, int i)；
// 用于告诉RecyclerView有多个视图需要显示
public int getItemCount()；
```

新的Adapter和原有的Adapter并没有太多的差别，只是不再需要我们写复用判断的逻辑，因为复用逻辑其实都是相似的，它已经有了自身的实现。和原有的Adapter一样，仍然可以通过notifyDataSetChanged来刷新UI，通过getItemViewType来获取对应位置的类型，但是它不再需要你指定有多少类型了，因为该方法已经能够判断出有多少类型。新增的onViewRecycled方法可以让使用者监听View被移除屏幕的时机，并且还提供了一个AdapterDataObserver的观察者，对外提供数据改变时的回调。

ViewHolder是对所有的单个item的封装，不仅包含了item需要显示的View，并且还包含和item相关的其它数据，例如：当前的position、之前的position、即将显示的position、被回收的次数、View的类型、是否处于显示中等信息。创建一个ViewHolder需要传递一个View对象，这个View就是该holder的显示视图，该View中通常会包含一些子视图，我们最好把这些子视图都记录在holder中，便于复用时设置不同的数据。

RecyclerView不再对布局进行管理，而是通过LayoutManager管理布局，我们可以通过继承自LayoutManager来实现特殊的布局，系统提供了三种常用的布局管理器：

LinearLayoutManager 线性布局
GridLayoutManager 九宫格布局
StaggeredGridLayoutManager 瀑布流布局

并且每一种都可以设置横行和纵向的布局，可惜的均不能添加header，如果要添加header，我们可以在Adapter中使用不同的类型来达到该效果。

RecyclerView默认提供了item的增加和删除的动画效果，如果我们使用自定义的动画，需要继承继承RecyclerView.ItemAnimator类，通过RecyclerView.setItemAnimator()方法来设置我们自定义的动画。

## cardview

在实现扁平化的UI处理上，通常离不开阴影和圆角，我们通常是让美工提供一个带有阴影和圆角效果的背景图片，现在我们有了更好的实现方式，那就是CardView。

CardView实际是一个FrameLayout类的子类，它为视图提供卡片样式，并保持在不同平台上拥有统一的风格。CardView组件可以设定阴影和圆角。

我们可以使用cardElevation属性在xml布局中设置阴影效果，在代码中可以通过setCardElevation达到同样的效果。阴影的设置和Android L中的Z属性类似。

设置圆角也相当容易，在xml中通过cardCornerRadius来设置，在代码中则使用setRadius，圆角的设置和Android L中的剪裁很相似。

如果我们想设置cardview的背景，请注意使用carBackgroundColor方法，setBackgroundColor也许会影响我们的圆角效果

## toolbar

Toolbar是Android L引入的一个新控件，用于取代ActionBar，它提供了ActionBar类似的功能，但是更灵活。不像ActionBar那么固定，Toolbar更像是一般的View元素，可以被放置在view树体系的任意位置，可以应用动画，可以跟着ScrollView滚动，可以与布局中的其他View交互。当然，你还可以用Toolbar替换掉ActionBar，只需调用Activity.setActionBar()。

为了兼容更多的设备一般我们都是通过AppCompat中的Android.support.v7.widget.Toolbar来使用Toolbar。

有两种使用Toolbar的方式：

将Toolbar当作actionbar来使用。这种情况一般发生在你想利用actionbar现有的一些功能（比如能够显示菜单中的操作项，响应菜单点击事件，使用ActionBarDrawerToggle等），但是又想获得比actionbar更多的控制权限。
将Toolbar当作一个独立的控件来使用，这种方式又名Standalone。

如果你要将Toolbar当作actionbar来使用，你首先要去掉actionbar，最简单的方法是使用Theme.AppCompat.NoActionBar主题。或者是设置主题的属性Android:windowNoTitle为true。然后在Activity的onCreate中调用setSupportActionBar(toolbar)，原本应该出现在ActionBar上的menu会自动出现在actionbar上。

Toolbar的高度、宽度、背景颜色等等一切View的属性完全取决于你，这都是因为Toolbar本质上只是个ViewGroup。将Toolbar当作一个独立的控件来使用是不需要去掉actionbar的（两者可以共存），可以使用任意主题。但是在这种情况下，menu菜单并不会自动的显示在Toolbar上，Toolbar也不会响应菜单的回调函数，如果你想让menu菜单项显示在Toolbar上，必须手动inflate menu。

```
toolbar.setOnMenuItemClickListener(new Toolbar.OnMenuItemClickListener() {
    @Override
    public boolean onMenuItemClick(MenuItem item) {
        // 处理menu事件
        return true;
    }
});
// 创建一个menu添加到toolbar上
toolbar.inflateMenu(R.menu.your_toolbar_menu);
```

## 兼容性

虽然Material Design新增了许多新特性，但是并不是所有新内容对对下保持了兼容。

### 使用v7包
v7 support libraries r21 及更高版本包含了以下Material Design特性：

使用Theme.AppCompat主题包含调色板主体属性，可以对应用的主题做统一的配色，但是不包括状态栏和底部操作栏
RecyclerView和CardView被独立出来，只要引入jar包，即可适配7以上的所有版本。
Palette类用于从图片提取主色调

### 系统组件

Theme.AppCompat主题中提供了这些组件的Material Design style：

EditText
Spinner
CheckBox
RadioButton
SwitchCompat
CheckedTextView
Color Palette

### 创建多个value和layout

针对Android L我们可以创建value-v21指定Material Design主题，而在其他value中指定Theme.AppCompat。layout布局也可以采用该方式，在Android L中使用系统控件，在低版本中使用我们自定义的控件活着第三方包来达到该效果。

### 版本检查

以下特性只在Android 5.0 (API level 21) 及以上版本中可用：

转场动画
触摸反馈
圆形展示动画
路径动画
矢量图
tint染色
所以在代码中遇上使用这些api的地方需要进行版本判断：

```
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
    // 使用新特性
} else {
    // 用其他替代方式
}
```

## 支持64位art虚拟机

# 6.0 - api 23

## 指纹身份验证

6.0上面可以使用指纹进行身份验证

## 应用链接

此版本通过提供功能更强大的应用链接，增强了 Android 的 intent 系统。您可以利用此功能将应用与您拥有的某个 Web 域关联。平台可以根据此关联确定在处理特定 Web 链接时默认使用的应用，跳过提示用户选择应用的步骤。

## 自动备份应用

现在，系统可以自动为应用执行完整数据备份和恢复。您的应用的目标平台必须是 Android 6.0（API 级别 23），才能启用此行为；您无需额外添加任何代码。如果用户删除其 Google 帐户，其备份数据也会随之删除。要了解该功能的工作方式以及配置文件系统备份内容的方法，请参阅配置应用自动备份。

## 语音交互

调用 isVoiceInteraction() 方法可确定是否是响应语音操作触发了您的 Activity。如果是这样，则您的应用可以使用 VoiceInteractor 类请求用户进行语音确认、从选项列表中进行选择以及执行其他操作。
大多数语音交互都由用户语音操作发起。但语音交互 Activity 也可在没有用户输入的情况下启动。例如，通过语音交互启动的另一应用也可发送 intent 来启动语音交互。要确定您的 Activity 是由用户语音查询还是另一语音交互应用启动，请调用 isVoiceInteractionRoot() 方法。如果另一应用启动了您的 Activity，该方法会返回 false。您的应用可能随即提示用户确认其有意执行此操作。

## Assist API

此版本提供了一种让用户通过助手程序与应用进行互动的新方式。要使用此功能，用户必须启用助手以使用当前上下文。启用助手后，用户可通过长按首页按钮在任何应用内召唤助手。

您的应用可通过设置 FLAG_SECURE 标记选择不与助手共享当前上下文。除了平台传递给助手的一组标准信息外，您的应用还可利用新增的 AssistContent 类共享其他信息。

要为助手提供您的应用内的其他上下文，请执行以下步骤：

实现 Application.OnProvideAssistDataListener 接口。
利用 registerOnProvideAssistDataListener() 注册此侦听器。
要提供特定于 Activity 的上下文信息，请重写 onProvideAssistData() 回调和新的 onProvideAssistContent() 回调（可选操作）

## 可采用的存储设备

使用此版本时，用户可以采用 SD 卡等外部存储设备。采用外部存储设备可加密和格式化设备，使其具有类似内部存储设备的行为。用户可以利用此特性在存储设备之间移动应用及其私有数据。移动应用时，系统会遵守清单中的 Android:installLocation 首选项。

请注意，在内部存储设备与外部存储设备之间移动应用时，如果您的应用访问以下 API 或字段，它们返回的文件路径将会动态变化。强烈建议：在生成文件路径时，请始终动态调用这些 API。请勿使用硬编码文件路径或之前生成的永久性完全限定文件路径。

Context 方法：
getFilesDir()
getCacheDir()
getCodeCacheDir()
getDatabasePath()
getDir()
getNoBackupFilesDir()
getFileStreamPath()
getPackageCodePath()
getPackageResourcePath()
ApplicationInfo 字段：
dataDir
sourceDir
nativeLibraryDir
publicSourceDir
splitSourceDirs
splitPublicSourceDirs

## 通知

此版本针对通知功能引入了下列 API 变更：

新增了 INTERRUPTION_FILTER_ALARMS 过滤级别，它对应于新增的“仅闹铃”免打扰模式。
新增了 CATEGORY_REMINDER 类别值，用于区分用户安排的提醒与其他事件 (CATEGORY_EVENT) 和闹铃 (CATEGORY_ALARM)。
新增了 Icon 类，您可以通过 setSmallIcon()方法和 setLargeIcon()方法将其附加到通知上。同理，addAction() 方法现在接受 Icon 对象，而不接受可绘制资源 ID。
新增了 getActiveNotifications() 方法，让您的应用能够了解哪些通知目前处于活动状态。要查看使用此功能的应用实现，请参阅 ActiveNotifications 示例。


## 相机功能

此版本提供了下列用于访问相机闪光灯和相机图像再处理的新 API：

Flashlight API
如果相机设备带有闪光灯，您可以通过调用 setTorchMode() 方法，在不打开相机设备的情况下打开或关闭闪光灯的火炬模式。应用对闪光灯或相机设备不享有独占所有权。每当相机设备不可用，或者开启火炬的其他相机资源不可用时，火炬模式即会被关闭并变为不可用状态。其他应用也可调用 setTorchMode() 来关闭火炬模式。当最后一个开启火炬模式的应用关闭时，火炬模式就会被关闭。

您可以注册一个回调，通过调用 registerTorchCallback() 方法接收有关火炬模式状态的通知。第一次注册回调时，系统会立即调用它，并返回所有当前已知配备闪光灯的相机设备的火炬模式状态。如果成功开启或关闭火炬模式，系统会调用 onTorchModeChanged() 方法。

Reprocessing API
Camera2 API 进行了扩展，以支持 YUV 和专用不透明格式图像再处理。要确定这些再处理功能是否可用，请调用 getCameraCharacteristics() 并检查有无 REPROCESS_MAX_CAPTURE_STALL 密钥。如果设备支持再处理，您可以通过调用 createReprocessableCaptureSession() 创建一个可再处理的相机采集会话并创建输入缓冲区再处理请求。

使用 ImageWriter 类可将输入缓冲区流与相机再处理输入相连。要获得空白缓冲区，请遵循以下编程模型：

调用 dequeueInputImage() 方法。
在输入缓冲区中填充数据。
通过调用 queueInputImage() 方法将缓冲区发送至相机。
如果您将 ImageWriter 对象与 PRIVATE 图像一起使用，您的应用并不能直接访问图像数据。请改为调用 queueInputImage() 方法，将 PRIVATE 图像直接传递给 ImageWriter，而不进行任何缓冲区复制。

ImageReader 类现在支持 PRIVATE 格式图像流。凭借此支持特性，您的应用可使 ImageReader 输出图像保持为循环图像队列，还可选择一个或多个图像并将其发送给 ImageWriter 进行相机再处理。

# 7.0

## 多窗口支持

就是分屏

## 通知增强功能

在 Android 7.0 中，我们重新设计了通知，使其更易于使用并且速度更快。部分变更包括：

模板更新：我们正在更新通知模板，新强调了英雄形象和化身。开发者将能够充分利用新模板，只需进行少量的代码调整。
消息传递样式自定义：您可以自定义更多与您的使用 MessagingStyle 类的通知相关的用户界面标签。您可以配置消息、会话标题和内容视图。
捆绑通知：系统可以将消息组合在一起（例如，按消息主题）并显示组。用户可以适当地进行拒绝或归档等操作。如果您已实现 Android Wear 的通知，那么您已经很熟悉此模式。
直接回复：对于实时通信应用，Android 系统支持内联回复，以便用户可以直接在通知界面中快速回复短信。
自定义视图：两个新的 API 让您在通知中使用自定义视图时可以充分利用系统装饰元素，如通知标题和操作。

## Android 中的 ICU4J API

Android 7.0 目前在 Android 框架（位于 Android.icu 软件包下）中提供 ICU4J API 的子集。迁移很简单，主要是需要从 com.java.icu 命名空间更改为 Android.icu。如果您已在您的应用中使用 ICU4J 捆绑包，切换到 Android 框架中提供的 Android.icu API 可以大量节省 APK 大小。

## webview

Javascript 在页面加载之前运行
从以 Android 7.0 为目标平台的应用开始，JavaScript 上下文会在加载新页面时重置。目前，新 WebView 实例中加载的第一个页面会继承上下文。

想要在 WebView 中注入 Javascript 的开发者应在页面开始加载后执行脚本。












