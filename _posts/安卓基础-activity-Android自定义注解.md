---
title: Android自定义注解
date: 2018-02-10 19:55:54
tag : "activity"
category : "安卓基础"
description : "本文主要是对Android绘图机制与处理技巧进行学习和反思"
---

注解是一种元数据，可以添加到java代码中，类、方法、变量、参数、包都可以被注解，注解对注解的代码没有直接的影响。

注解是在解析的过程中做出了相应的处理，注解仅仅是一个标记。

定义一个注解的关键字是@interface

### 元注解

元注解共有四种 @Retention, @Target, @Inherited, @Documented

+ @Retention 保留的范围，默认值为class，可选值有三种

SOURCE:只在源码中可用
CLASS:在源码和字节码中可用
RUNTIME:在源码、字节码、运行时均可用

+ @Target: 表示可以用来修饰哪些元素，如TYPE/METHOD/CONSTRUCTOR/FIELD/PARAMETER等，未标识及代表可以修饰所有

+ @Inherited:是否可以被继承，默认为false

+ @Documented: 是否会被保存到javadoc文档中

### 自定义注解--实现findViewById()

第一步定义注解：
```
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
public @interface ViewInject {

    int value();

    /* parent view id */
    int parentId() default 0;
}
```

第二步处理注解：
```
public class ViewUtils {

    private ViewUtils() {
    }

    public static void inject(Activity activity) {
        injectObject(activity, new ViewFinder(activity));
    }

    @SuppressWarnings("ConstantConditions")
    private static void injectObject(Object handler, ViewFinder finder) {

        Class<?> handlerType = handler.getClass();

        // inject view
        Field[] fields = handlerType.getDeclaredFields();
        if (fields != null && fields.length > 0) {
            for (Field field : fields) {
                ViewInject viewInject = field.getAnnotation(ViewInject.class);
                if (viewInject != null) {
                    try {
                        View view = finder.findViewById(viewInject.value(), viewInject.parentId());
                        if (view != null) {
                            field.setAccessible(true);
                            field.set(handler, view);
                        }
                    } catch (Throwable e) {
                        e.printStackTrace();
                    }
                }
            }
        }

    }

}
```

```
public class ViewFinder {


    private Activity activity;


    public ViewFinder(Activity activity) {
        this.activity = activity;
    }

    public View findViewById(int id) {
        return  activity.findViewById(id);
    }

    public View findViewById(int id, int pid) {
        View pView = null;
        if (pid > 0) {
            pView = this.findViewById(pid);
        }

        View view = null;
        if (pView != null) {
            view = pView.findViewById(id);
        } else {
            view = this.findViewById(id);
        }
        return view;
    }


}
```

第三步 activity调用
```
public class DIYAnnotationActivity extends AppCompatActivity {


    @ViewInject(R.id.textView)
    private TextView textView;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_annotation);

        ViewUtils.inject(this);

        textView.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                textView.setText("成功了！");
            }
        });

    }


}
```

基本上是通过获取activity实例，然后通过反射遍历field找到用到这个注解的地方，然后进行findviewbyid的进行，之后返回view的实例