---
title: 安卓基础-ARouter组件化框架学习笔记
date: 2018-03-14 16:29:53
tag : "Rxjava"
category : "安卓基础"
description : "本文主要是对Rxjava的通用方法进行学习和反思"
---

ARouter是用于组件化的比较好的一个方案，不过性能方面肯定不如直接跳转。


# 基本使用方法

- 添加依赖

```
android {
    defaultConfig {
	...
	javaCompileOptions {
	    annotationProcessorOptions {
		arguments = [ moduleName : project.getName() ]
	    }
	}
    }
}

dependencies {
    // 替换成最新版本, 需要注意的是api
    // 要与compiler匹配使用，均使用最新版可以保证兼容
    compile 'com.alibaba:arouter-api:x.x.x'
    annotationProcessor 'com.alibaba:arouter-compiler:x.x.x'
    ...
}
// 旧版本gradle插件(< 2.2)，可以使用apt插件，配置方法见文末'其他#4'
// Kotlin配置参考文末'其他#5'
```

- 添加注解

```
@Route(path = "/test/activity")
public class YourActivity extend Activity {
    ...
}
```

- 初始化sdk

```
if (isDebug()) {           // 这两行必须写在init之前，否则这些配置在init过程中将无效
    ARouter.openLog();     // 打印日志
    ARouter.openDebug();   // 开启调试模式(如果在InstantRun模式下运行，必须开启调试模式！线上版本需要关闭,否则有安全风险)
}
ARouter.init(mApplication); // 尽可能早，推荐在Application中初始化
```

- 进行路由操作

```
// 1. 应用内简单的跳转(通过URL跳转在'进阶用法'中)
ARouter.getInstance().build("/test/activity").navigation();

// 2. 跳转并携带参数
ARouter.getInstance().build("/test/1")
			.withLong("key1", 666L)
			.withString("key3", "888")
			.withObject("key4", new Test("Jack", "Rose"))
			.navigation();

```

- 混淆

```
-keep public class com.alibaba.android.arouter.routes.**{*;}
-keep class * implements com.alibaba.android.arouter.facade.template.ISyringe{*;}

# 如果使用了 byType 的方式获取 Service，需添加下面规则，保护接口
-keep interface * implements com.alibaba.android.arouter.facade.template.IProvider

# 如果使用了 单类注入，即不定义接口实现 IProvider，需添加下面规则，保护实现
-keep class * implements com.alibaba.android.arouter.facade.template.IProvider
```

- 使用gradle进行路由表的自动加载

```
apply plugin: 'com.alibaba.arouter'

buildscript {
    repositories {
        jcenter()
    }

    dependencies {
        classpath "com.alibaba:arouter-register:1.0.0"
    }
}
```

# 进阶使用方法

- 通过url跳转
```
// 新建一个Activity用于监听Schame事件,之后直接把url传递给ARouter即可
public class SchameFilterActivity extends Activity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
	super.onCreate(savedInstanceState);

	Uri uri = getIntent().getData();
	ARouter.getInstance().build(uri).navigation();
	finish();
    }
}
```

mainfest
```
<activity android:name=".activity.SchameFilterActivity">
	<!-- Schame -->
	<intent-filter>
	    <data
		android:host="m.aliyun.com"
		android:scheme="arouter"/>

	    <action android:name="android.intent.action.VIEW"/>

	    <category android:name="android.intent.category.DEFAULT"/>
	    <category android:name="android.intent.category.BROWSABLE"/>
	</intent-filter>
</activity>
```

- 解析url中的参数

```
// 为每一个参数声明一个字段，并使用 @Autowired 标注
// URL中不能传递Parcelable类型数据，通过ARouter api可以传递Parcelable对象
@Route(path = "/test/activity")
public class Test1Activity extends Activity {
    @Autowired
    public String name;
    @Autowired
    int age;
    @Autowired(name = "girl") // 通过name来映射URL中的不同参数
    boolean boy;
    @Autowired
    TestObj obj;    // 支持解析自定义对象，URL中使用json传递

    @Override
    protected void onCreate(Bundle savedInstanceState) {
	super.onCreate(savedInstanceState);
	ARouter.getInstance().inject(this);

	// ARouter会自动对字段进行赋值，无需主动获取
	Log.d("param", name + age + boy);
    }
}


// 如果需要传递自定义对象，需要实现 SerializationService,并使用@Route注解标注(方便用户自行选择序列化方式)，例如：
@Route(path = "/service/json")
public class JsonServiceImpl implements SerializationService {
    @Override
    public void init(Context context) {

    }

    @Override
    public <T> T json2Object(String text, Class<T> clazz) {
        return JSON.parseObject(text, clazz);
    }

    @Override
    public String object2Json(Object instance) {
        return JSON.toJSONString(instance);
    }
}
```

- 声明拦截器

```
// 比较经典的应用就是在跳转过程中处理登陆事件，这样就不需要在目标页重复做登陆检查
// 拦截器会在跳转之间执行，多个拦截器会按优先级顺序依次执行
@Interceptor(priority = 8, name = "测试用拦截器")
public class TestInterceptor implements IInterceptor {
    @Override
    public void process(Postcard postcard, InterceptorCallback callback) {
	...
	callback.onContinue(postcard);  // 处理完成，交还控制权
	// callback.onInterrupt(new RuntimeException("我觉得有点异常"));      // 觉得有问题，中断路由流程

	// 以上两种至少需要调用其中一种，否则不会继续路由
    }

    @Override
    public void init(Context context) {
	// 拦截器的初始化，会在sdk初始化的时候调用该方法，仅会调用一次
    }
}
```

- 处理跳转结果

```
// 使用两个参数的navigation方法，可以获取单次跳转的结果
ARouter.getInstance().build("/test/1").navigation(this, new NavigationCallback() {
    @Override
    public void onFound(Postcard postcard) {
      ...
    }

    @Override
    public void onLost(Postcard postcard) {
	...
    }
});
```

- 自定义全局降级策略

```
// 实现DegradeService接口，并加上一个Path内容任意的注解即可
@Route(path = "/xxx/xxx")
public class DegradeServiceImpl implements DegradeService {
  @Override
  public void onLost(Context context, Postcard postcard) {
	// do something.
  }

  @Override
  public void init(Context context) {

  }
}
```

- 为目标页面声明更多信息

```
// 我们经常需要在目标页面中配置一些属性，比方说"是否需要登陆"之类的
// 可以通过 Route 注解中的 extras 属性进行扩展，这个属性是一个 int值，换句话说，单个int有4字节，也就是32位，可以配置32个开关
// 剩下的可以自行发挥，通过字节操作可以标识32个开关，通过开关标记目标页面的一些属性，在拦截器中可以拿到这个标记进行业务逻辑判断
@Route(path = "/test/activity", extras = Consts.XXXX)
```

- 通过依赖注入解耦:服务管理(一) 暴露服务

```
// 声明接口,其他组件通过接口来调用服务
public interface HelloService extends IProvider {
    String sayHello(String name);
}

// 实现接口
@Route(path = "/service/hello", name = "测试服务")
public class HelloServiceImpl implements HelloService {

    @Override
    public String sayHello(String name) {
	return "hello, " + name;
    }

    @Override
    public void init(Context context) {

    }
}
```

- 通过依赖注入解耦:服务管理(二) 发现服务

```
public class Test {
    @Autowired
    HelloService helloService;

    @Autowired(name = "/service/hello")
    HelloService helloService2;

    HelloService helloService3;

    HelloService helloService4;

    public Test() {
	ARouter.getInstance().inject(this);
    }

    public void testService() {
	 // 1. (推荐)使用依赖注入的方式发现服务,通过注解标注字段,即可使用，无需主动获取
	 // Autowired注解中标注name之后，将会使用byName的方式注入对应的字段，不设置name属性，会默认使用byType的方式发现服务(当同一接口有多个实现的时候，必须使用byName的方式发现服务)
	helloService.sayHello("Vergil");
	helloService2.sayHello("Vergil");

	// 2. 使用依赖查找的方式发现服务，主动去发现服务并使用，下面两种方式分别是byName和byType
	helloService3 = ARouter.getInstance().navigation(HelloService.class);
	helloService4 = (HelloService) ARouter.getInstance().build("/service/hello").navigation();
	helloService3.sayHello("Vergil");
	helloService4.sayHello("Vergil");
    }
}
```

# 原理解析




