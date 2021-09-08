---
title: "安卓基础-Rxjava（1）"
date: 2017-12-21 16:24:02
tag : "Rxjava"
category : "安卓基础"
description : "本文主要是对Rxjava的通用方法进行学习和反思"

---
#[Rxjava2.0](http://blog.csdn.net/flybasker/article/details/78703295)

## 基础概念

Observable：在观察者模式中称为“被观察者”；
Observer：观察者模式中的“观察者”，可接收Observable发送的数据；
subscribe：订阅，观察者与被观察者，通过Observable的subscribe()方法进行订阅；
Subscriber：也是一种观察者，在2.0中 它与Observer没什么实质的区别，不同的是 Subscriber要与Flowable(也是一种被观察者)联合使用，该部分 内容是2.0新增的，后续文章再介绍。Obsesrver用于订阅Observable，而Subscriber用于订阅Flowable.

## Rxjava中定义的事件方法

onNext(),普通事件,按照队列依次进行处理.
onComplete(),事件队列完结时调用该方法
onError(),事件处理过程中出现异常时，onError()触发，同时队列终止,不再有事件发出.
onSubscribe(),RxJava 2.0 中新增的，传递参数为Disposable,可用于切断接收事件让Observable (被观察者)开启子线程执行耗操作，完成耗时操作后，触发回调，通知Observer (观察者)进行主线程UI更新

## observable的几种创建方式
1. just()方式
使用just( )，将创建一个Observable并自动调用onNext( )发射数据。
也就是通过just( )方式 直接触发onNext()，just中传递的参数将直接在Observer的onNext()方法中接收到。

2. fromIterable()方式
使用fromIterable()，遍历集合，发送每个item.多次自动调用onNext()方法，每次传入一个item.
注意：Collection接口是Iterable接口的子接口，所以所有Collection接口的实现类都可以作为Iterable对象直接传入fromIterable()    方法。

3. defer()方式
当观察者订阅时,才创建Observable，并且针对每个观察者创建都是一个新的Observable.
通过Callable中的回调方法call(),决定使用以何种方式来创建这个Observable对象,当订阅后，发送事件.

4. interval( )方式
创建一个按固定时间间隔发射整数序列的Observable，可用作定时器。按照固定时间间隔来调用onNext()方法。

5. timer( )方式
通过此种创建一个Observable,它在一个给定的延迟后发射一个特殊的值，即表示延迟指定时间后，调用onNext()方法。

6. range( )方式,range(x,y)
创建一个发射特定整数序列的Observable，第一个参数x为起始值，第二个y为发送的个数，如果y为0则不发送，y为负数则抛异常。
range(1,5)
上述表示发射1到5的数。即调用5次Next()方法，依次传入1-5数字。

7. repeat( )方式
创建一个Observable，该Observable的事件可以重复调用。

## ObservableEmitter
Emitter是发射器的意思,就是用来发出事件的，它可以发出三种类型的事件 
通过调用onNext(T value),发出next事件 
通过调用onComplete(),发出complete事件 
通过调用onError(Throwable error),发出error事件 
注意事项: 
onComplete和onError唯一并且互斥 
发送多个onComplete, 第一个onComplete接收到,就不再接收了. 
发送多个onError, 则收到第二个onError事件会导致程序会崩溃. 
不可以随意乱七八糟发射事件，需要满足一定的规则： 
上游可以发送无限个onNext, 下游也可以接收无限个onNext. 
当上游发送了一个onComplete后, 上游onComplete之后的事件将会继续发送, 而下游收到onComplete事件之后将不再继续接收事件. 
上游发送了一个onError后, 上游onError之后的事件将继续发送, 而下游收到onError事件之后将不再继续接收事件. 
上游可以不发送onComplete或onError. 
最为关键的是onComplete和onError必须唯一并且互斥, 即不能发多个onComplete, 也不能发多个onError, 也不能先发一个onComplete, 然后再发一个onError 

## Disposable
一次性,它理解成两根管道之间的一个机关, 当调用它的dispose()方法时, 它就会将两根管道切断, 从而导致下游收不到事件. 
在RxJava中,用它来切断Observer(观察者)与Observable(被观察者)之间的连接，当调用它的dispose()方法时, 它就会将Observer(观察者)与Observable(被观察者)之间的连接切断, 从而导致Observer(观察者)收不到事件。 
注意: 调用dispose()并不会导致上游不再继续发送事件, 上游会继续发送剩余的事件 

## 线程调度
1. Schedulers.immediate(): 
直接在当前线程运行，相当于不指定线程。这是默认的Scheduler。

2. Schedulers.newThread(): 
总是启用新线程，并在新线程执行操作。

3. Schedulers.io(): I/O 
操作（读写文件、读写数据库、网络信息交互等）所使用的Scheduler。行为模式和newThread()差不多，区别在于io()的内部实现是是用一个无数量上限的线程池，可以重用空闲的线程，因此多数情况下io()比newThread()更有效率。不要把计算工作放在io()中，可以避免创建不必要的线程。

4. Schedulers.computation(): 
计算所使用的Scheduler。这个计算指的是 CPU 密集型计算，即不会被 I/O 等操作限制性能的操作，例如图形的计算。这个Scheduler使用的固定的线程池，大小为 CPU 核数。不要把 I/O 操作放在computation()中，否则 I/O 操作的等待时间会浪费 CPU。

5. AndroidSchedulers.mainThread()，
Android专用线程，指定操作在主线程运行。

如何切换线程呢？RxJava中提供了两个方法：
    subscribeOn() 和 observeOn() ，
两者的不同点在于：

subscribeOn(): 指定subscribe()订阅所发生的线程，或者叫做事件产生的线程。

observeOn(): 指定Observer所运行在的线程，即onNext()执行的线程。或者叫做事件消费的线程。

## 操作符
操作符就是用于在Observable和最终的Observer之间，通过转换Observable为其他观察者对象的过程，修改发出的事件,最终将最简洁的数据传递给Observer对象. 

1. map()操作符，就是把原来的Observable对象转换成另一个Observable对象，同时将传输的数据进行一些灵活的操作，方便Observer获得想要的数据形式。
举例:
```
Observable<Integer> observable = Observable
        .just("hello")
        .map(new Function<String, Integer>() {
            @Override
            public Integer apply(String s) throws Exception {
                return s.length();
            }
        });
```

2. flatMap()操作符 
flatMap()对于数据的转换比map()更加彻底，如果发送的数据是集合，flatmap()重新生成一个Observable对象，并把数据转换成Observer想 要的数据形式。它可以返回任何它想返回的Observable对象。 
举例:
```
Observable.just(list)
       .flatMap(new Function<List<String>, ObservableSource<?>>() {
            @Override
            public ObservableSource<?> apply(List<String> strings) throws Exception {
                return Observable.fromIterable(strings);
            }
        });
```

3. filter()操作符 
filter()操作符根据它的test()方法中，根据自己想过滤的数据加入相应的逻辑判断，返回true则表示数据满足条件，返回false则表示数据需要被过滤。最后过滤出的数据将加入到新的Observable对象中，方便传递给Observer想要的数据形式。 
举例:
```
Observable
        .just(list)
        .flatMap(new Function<List<String>, ObservableSource<?>>() {
            @Override
            public ObservableSource<?> apply(List<String> strings) throws Exception {
                return Observable.fromIterable(strings);
            }
        }).filter(new Predicate<Object>() {
            @Override
            public boolean test(Object s) throws Exception {
                String newStr = (String) s;
                if (newStr.charAt(5) - '0' > 5) {
                    return true;
                }
                return false;
            }
        }).subscribe(new Consumer<Object>() {
            @Override
            public void accept(Object o) throws Exception {
                System.out.println((String)o);
            }
        });
```

4. take()操作符
输出最多指定数量的结果.(接收指定数量的结果) 
举例:
```
Observable.just(new ArrayList<String>(){
            {
                for (int i = 0; i < 8; i++) {
                    add("data"+i);
                }
            }
        }).flatMap(new Function<List<String>, ObservableSource<?>>() {
            @Override
            public ObservableSource<?> apply(List<String> strings) throws Exception {
                return Observable.fromIterable(strings);
            }
        }).take(5).subscribe(new Consumer<Object>() {
            @Override
            public void accept(Object s) throws Exception {
                DemonstrateUtil.showLogResult(s.toString());
            }
        });
```

5. doOnNext()操作符
允许我们在每次输出一个元素之前做一些额外的事情 
举例:
```
Observable.just(new ArrayList<String>(){
            {
                for (int i = 0; i < 6; i++) {
                    add("data"+i);
                }
            }
        }).flatMap(new Function<List<String>, ObservableSource<?>>() {
            @Override
            public ObservableSource<?> apply(List<String> strings) throws Exception {
                return Observable.fromIterable(strings);
            }
        }).take(5).doOnNext(new Consumer<Object>() {
            @Override
            public void accept(Object o) throws Exception {
                DemonstrateUtil.showLogResult("额外的准备工作!");
            }
        }).subscribe(new Consumer<Object>() {
            @Override
            public void accept(Object s) throws Exception {
                DemonstrateUtil.showLogResult(s.toString());
            }
        });
```

## Flowable的理解 
Flowable是一个被观察者，与Subscriber(观察者)配合使用，解决Backpressure问题 
Backpressure(背压)。所谓背压，即生产者的速度大于消费者的速度带来的问题。

> 什么情况下才会产生Backpressure问题？
1.如果生产者和消费者在一个线程的情况下，无论生产者的生产速度有多快，每生产一个事件都会通知消费者，等待消费者消费完毕，再生产下一个事件。
所以在这种情况下，根本不存在Backpressure问题。即同步情况下，Backpressure问题不存在。
2.如果生产者和消费者不在同一线程的情况下，如果生产者的速度大于消费者的速度，就会产生Backpressure问题。
即异步情况下，Backpressure问题才会存在。

现象演示说明:
被观察者是事件的生产者,观察者是事件的消费者.假如生产者无限生成事件,而消费者以很缓慢的节奏来消费事件,会造成事件无限堆积,形成背压,最后造成OOM!
Flowable悠然而生，专门用来处理这类问题。
Flowable是为了应对Backpressure而产生的。Flowable是一个被观察者，
与Subscriber(观察者)配合使用，解决Backpressure问题。
注意：处理Backpressure的策略仅仅是处理Subscriber接收事件的方式，并不影响Flowable发送事件的方法。
即使采用了处理Backpressure的策略，Flowable原来以什么样的速度产生事件，现在还是什么样的速度不会变化，主要处理的是Subscriber接收事件的方式。

处理Backpressure问题的策略,或者来解决Backpressure问题

    BackpressureStrategy.ERROR
    如果缓存池溢出,就会立刻抛出MissingBackpressureException异常
    request()用来向生产者申请可以消费的事件数量,这样我们便可以根据本身的消费能力进行消费事件.
    虽然并不限制向request()方法中传入任意数字，但是如果消费者并没有这么多的消费能力，依旧会造成资源浪费，最后产生OOM
    at java.lang.OutOfMemoryError.<init>(OutOfMemoryError.java:33)
    在异步调用时，RxJava中有个缓存池，用来缓存消费者处理不了暂时缓存下来的数据，缓存池的默认大小为128，即只能缓存128个事件。
    无论request()中传入的数字比128大或小，缓存池中在刚开始都会存入128个事件。
    当然如果本身并没有这么多事件需要发送，则不会存128个事件。
    应用举例:


    BackpressureStrategy.BUFFER
    是把RxJava中默认的只能存128个事件的缓存池换成一个大的缓存池,支持存更多的数据.
    消费者通过request()即使传入一个很大的数字，生产者也会生产事件,并将处理不了的事件缓存.
    注意:
    这种方式任然比较消耗内存，除非是我们比较了解消费者的消费能力，能够把握具体情况，不会产生OOM。
    BUFFER要慎用

    BackpressureStrategy.DROP
    顾名思义,当消费者处理不了事件，就丢弃!
    例如,当数据源创建了200个事件,先不进行消费临时进行缓存实际缓存128个,我们第一次申请消费了100个,再次申请消费100个,
    那么实际只消费了128个,而其余的72个被丢弃了!

    BackpressureStrategy.LATEST
    LATEST与DROP功能基本一致,当消费者处理不了事件，就丢弃!
    唯一的区别就是LATEST总能使消费者能够接收到生产者产生的最后一个事件。
    例如,当数据源创建了200个事件,先不进行消费临时进行缓存,我们第一次申请消费了100个,再次申请消费100个,
    那么实际只消费了129个,而其余的71个被丢弃了,但是第200个(最后一个)会被消费.


    BackpressureStrategy.MISSING
    生产的事件没有进行缓存和丢弃,下游接收到的事件必须进行消费或者处理!



在RxJava中会经常遇到一种情况就是被观察者发送消息十分迅速以至于观察者不能及时的响应这些消息
举例:
Observable.create(new ObservableOnSubscribe<Integer>() {
        @Override
        public void subscribe(ObservableEmitter<Integer> e) throws Exception {
            while (true){
                e.onNext(1);
            }
        }
    })
            .subscribeOn(Schedulers.io())
            .observeOn(AndroidSchedulers.mainThread())
            .subscribe(new Consumer<Integer>() {
        @Override
        public void accept(Integer integer) throws Exception {
            Thread.sleep(2000);
            System.out.println(integer);
        }
    });
    被观察者是事件的生产者，观察者是事件的消费者。上述例子中可以看出生产者无限生成事件，而消费者每2秒才能消费一个事件，这会造成事件无限堆积，最后造成OOM。
Flowable就是由此产生，专门用来处理这类问题