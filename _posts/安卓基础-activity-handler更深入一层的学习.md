---
title: 安卓基础-activity-handler更深入一层的学习
date: 2021-07-18 16:21:32
tags: "handler"
category: "安卓基础"
description: "本文对最近面试过程中遇到的handler没有答出来的问题做一次系统性的学习"
---

以前学handler还是在做framework的时候，有些细节也没有考虑过

# Handler概览

## Handler构造

```
    /**
     * Default constructor associates this handler with the {@link Looper} for the
     * current thread.
     *
     * If this thread does not have a looper, this handler won't be able to receive messages
     * so an exception is thrown.
     */
    public Handler() {
        this(null, false);
    }

    /**
     * Constructor associates this handler with the {@link Looper} for the
     * current thread and takes a callback interface in which you can handle
     * messages.
     *
     * If this thread does not have a looper, this handler won't be able to receive messages
     * so an exception is thrown.
     *
     * @param callback The callback interface in which to handle messages, or null.
     */
    public Handler(Callback callback) {
        this(callback, false);
    }

    /**
     * Use the provided {@link Looper} instead of the default one.
     *
     * @param looper The looper, must not be null.
     */
    public Handler(Looper looper) {
        this(looper, null, false);
    }

    /**
     * Use the provided {@link Looper} instead of the default one and take a callback
     * interface in which to handle messages.
     *
     * @param looper The looper, must not be null.
     * @param callback The callback interface in which to handle messages, or null.
     */
    public Handler(Looper looper, Callback callback) {
        this(looper, callback, false);
    }

    /**
     * Use the {@link Looper} for the current thread
     * and set whether the handler should be asynchronous.
     *
     * Handlers are synchronous by default unless this constructor is used to make
     * one that is strictly asynchronous.
     *
     * Asynchronous messages represent interrupts or events that do not require global ordering
     * with respect to synchronous messages.  Asynchronous messages are not subject to
     * the synchronization barriers introduced by {@link MessageQueue#enqueueSyncBarrier(long)}.
     *
     * @param async If true, the handler calls {@link Message#setAsynchronous(boolean)} for
     * each {@link Message} that is sent to it or {@link Runnable} that is posted to it.
     *
     * @hide
     */
    public Handler(boolean async) {
        this(null, async);
    }

    /**
     * Use the {@link Looper} for the current thread with the specified callback interface
     * and set whether the handler should be asynchronous.
     *
     * Handlers are synchronous by default unless this constructor is used to make
     * one that is strictly asynchronous.
     *
     * Asynchronous messages represent interrupts or events that do not require global ordering
     * with respect to synchronous messages.  Asynchronous messages are not subject to
     * the synchronization barriers introduced by {@link MessageQueue#enqueueSyncBarrier(long)}.
     *
     * @param callback The callback interface in which to handle messages, or null.
     * @param async If true, the handler calls {@link Message#setAsynchronous(boolean)} for
     * each {@link Message} that is sent to it or {@link Runnable} that is posted to it.
     *
     * @hide
     */
    public Handler(Callback callback, boolean async) {
        if (FIND_POTENTIAL_LEAKS) {
            final Class<? extends Handler> klass = getClass();
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                    klass.getCanonicalName());
            }
        }

        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread " + Thread.currentThread()
                        + " that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }

    /**
     * Use the provided {@link Looper} instead of the default one and take a callback
     * interface in which to handle messages.  Also set whether the handler
     * should be asynchronous.
     *
     * Handlers are synchronous by default unless this constructor is used to make
     * one that is strictly asynchronous.
     *
     * Asynchronous messages represent interrupts or events that do not require global ordering
     * with respect to synchronous messages.  Asynchronous messages are not subject to
     * the synchronization barriers introduced by conditions such as display vsync.
     *
     * @param looper The looper, must not be null.
     * @param callback The callback interface in which to handle messages, or null.
     * @param async If true, the handler calls {@link Message#setAsynchronous(boolean)} for
     * each {@link Message} that is sent to it or {@link Runnable} that is posted to it.
     *
     * @hide
     */
    public Handler(Looper looper, Callback callback, boolean async) {
        mLooper = looper;
        mQueue = looper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
```

handler的构造函数其实就分成两种，一种构造最后走到双参数构造里面，一种走到三参数构造里面。

区别在于双参数里面是通过  `mLooper = Looper.myLooper();`来获取looper。

### 双参数中looper的来源
```
    /**
     * Return the Looper object associated with the current thread.  Returns
     * null if the calling thread is not associated with a Looper.
     */
    public static @Nullable Looper myLooper() {
        return sThreadLocal.get();
    }
```

`Looper.myLooper()`的方法是如上，会从threadlocal中直接get出来。

`static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();`

这个threadlocal是存在于Looper成员变量区域中

```
    /** Initialize the current thread as a looper.
      * This gives you a chance to create handlers that then reference
      * this looper, before actually starting the loop. Be sure to call
      * {@link #loop()} after calling this method, and end it by calling
      * {@link #quit()}.
      */
    public static void prepare() {
        prepare(true);
    }

    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }
```

在prepare的过程中会创建新的Looper塞到对应的线程threadlocal中

### 双参数和三参数的共同点

```
mQueue = looper.mQueue;
mCallback = callback;
mAsynchronous = async;
```
可以看出messageQueue是在looper内部的一个变量

## async 的作用

```
     * @param async If true, the handler calls {@link Message#setAsynchronous(boolean)} for
     * each {@link Message} that is sent to it or {@link Runnable} that is posted to it.
```

从备注里面可以看到，如果async为true的话，会在message中setAsynchronous

### Message#setAsynchronous()

```
    /**
     * Sets whether the message is asynchronous, meaning that it is not
     * subject to {@link Looper} synchronization barriers.
     * <p>
     * Certain operations, such as view invalidation, may introduce synchronization
     * barriers into the {@link Looper}'s message queue to prevent subsequent messages
     * from being delivered until some condition is met.  In the case of view invalidation,
     * messages which are posted after a call to {@link android.view.View#invalidate}
     * are suspended by means of a synchronization barrier until the next frame is
     * ready to be drawn.  The synchronization barrier ensures that the invalidation
     * request is completely handled before resuming.
     * </p><p>
     * Asynchronous messages are exempt from synchronization barriers.  They typically
     * represent interrupts, input events, and other signals that must be handled independently
     * even while other work has been suspended.
     * </p><p>
     * Note that asynchronous messages may be delivered out of order with respect to
     * synchronous messages although they are always delivered in order among themselves.
     * If the relative order of these messages matters then they probably should not be
     * asynchronous in the first place.  Use with caution.
     * </p>
     *
     * @param async True if the message is asynchronous.
     *
     * @see #isAsynchronous()
     */
    public void setAsynchronous(boolean async) {
        if (async) {
            flags |= FLAG_ASYNCHRONOUS;
        } else {
            flags &= ~FLAG_ASYNCHRONOUS;
        }
    }
```

这个方法备注里面写的还是比较清楚的

message设置了这个方法之后，将会不遵从looper的`同步屏障`。

像一些方法，`view#invalidation`，该操作传入messagequeue的时候会同步往message queue中引入同步屏障，直到满足一些条件。

在`invalidation`这种情况出现的时候，在invalidate之后post的message将会被同步屏障拦截，直到下一帧被绘制完毕。

同步屏障会确保invalidation操作执行完毕之后在执行别的同步消息。

异步消息则会被免除于同步屏障之外，异步消息一般是中断，输入事件，或者别的需要在其他工作停滞状态时独立分开处理的信号等。

> 请注意，尽管异步消息之间总是按顺序传递，但与同步消息相比，异步消息的传递可能是无序的。如果这些消息的相对顺序很重要，那么它们可能一开始就不应该是异步的。谨慎使用。

最后这句话就用谷歌翻译了。..

## handler 另外四种构造方式

### public static Handler createAsync(@NonNull Looper looper)

```
    public static Handler createAsync(@NonNull Looper looper) {
        if (looper == null) throw new NullPointerException("looper must not be null");
        return new Handler(looper, null, true);
    }
```

这个就是创建异步的handler

### public static Handler createAsync(@NonNull Looper looper, @NonNull Callback callback)

```
   @NonNull
    public static Handler createAsync(@NonNull Looper looper, @NonNull Callback callback) {
        if (looper == null) throw new NullPointerException("looper must not be null");
        if (callback == null) throw new NullPointerException("callback must not be null");
        return new Handler(looper, callback, true);
    }
```

上一种handler的变体

### public static Handler getMain()


```
    /** @hide */
    @NonNull
    public static Handler getMain() {
        if (MAIN_THREAD_HANDLER == null) {
            MAIN_THREAD_HANDLER = new Handler(Looper.getMainLooper());
        }
        return MAIN_THREAD_HANDLER;
    }
```

这个方法提供给系统使用

### public static Handler mainIfNull(@Nullable Handler handler)

```
    /** @hide */
    @NonNull
    public static Handler mainIfNull(@Nullable Handler handler) {
        return handler == null ? getMain() : handler;
    }
```

这个方法提供给系统使用

## 创造message的几种方式

### public final Message obtainMessage()

```
    /**
     * Returns a new {@link android.os.Message Message} from the global message pool. More efficient than
     * creating and allocating new instances. The retrieved message has its handler set to this instance (Message.target == this).
     *  If you don't want that facility, just call Message.obtain() instead.
     */
    public final Message obtainMessage()
    {
        return Message.obtain(this);
    }
```

从全局的message pool里面获取一个新的message，比直接创建实例更有效率，并且已经设置过target = this了

### obtaionMessage的一些变种

```
   /**
     * Same as {@link #obtainMessage()}, except that it also sets the what member of the returned Message.
     * 
     * @param what Value to assign to the returned Message.what field.
     * @return A Message from the global message pool.
     */
    public final Message obtainMessage(int what)
    {
        return Message.obtain(this, what);
    }
    
    /**
     * 
     * Same as {@link #obtainMessage()}, except that it also sets the what and obj members 
     * of the returned Message.
     * 
     * @param what Value to assign to the returned Message.what field.
     * @param obj Value to assign to the returned Message.obj field.
     * @return A Message from the global message pool.
     */
    public final Message obtainMessage(int what, Object obj)
    {
        return Message.obtain(this, what, obj);
    }

    /**
     * 
     * Same as {@link #obtainMessage()}, except that it also sets the what, arg1 and arg2 members of the returned
     * Message.
     * @param what Value to assign to the returned Message.what field.
     * @param arg1 Value to assign to the returned Message.arg1 field.
     * @param arg2 Value to assign to the returned Message.arg2 field.
     * @return A Message from the global message pool.
     */
    public final Message obtainMessage(int what, int arg1, int arg2)
    {
        return Message.obtain(this, what, arg1, arg2);
    }
    
    /**
     * 
     * Same as {@link #obtainMessage()}, except that it also sets the what, obj, arg1,and arg2 values on the 
     * returned Message.
     * @param what Value to assign to the returned Message.what field.
     * @param arg1 Value to assign to the returned Message.arg1 field.
     * @param arg2 Value to assign to the returned Message.arg2 field.
     * @param obj Value to assign to the returned Message.obj field.
     * @return A Message from the global message pool.
     */
    public final Message obtainMessage(int what, int arg1, int arg2, Object obj)
    {
        return Message.obtain(this, what, arg1, arg2, obj);
    }
```

### Message#obtain()

```
    /**
     * Return a new Message instance from the global pool. Allows us to
     * avoid allocating new objects in many cases.
     */
    public static Message obtain() {
        synchronized (sPoolSync) {
            if (sPool != null) {
                Message m = sPool;
                sPool = m.next;
                m.next = null;
                m.flags = 0; // clear in-use flag
                sPoolSize--;
                return m;
            }
        }
        return new Message();
    }
```

obtain方法其实就是判断pool是否有值，有值的话返回出来，没值就新增。

sPool的赋值过程如下

` Message#recycle() => Message#recycleUnchecked() => sPool = this`

这也代表着Message其实是一个单链表节点，其会静态缓存一系列的被recycle的Message，这也代表平时使用完Message之后，记得recycle一下

## Post

Post其实就是核心中的核心，其有很多post的方法

### 第一种：public final boolean post(Runnable r)

```
    /**
     * Causes the Runnable r to be added to the message queue.
     * The runnable will be run on the thread to which this handler is 
     * attached. 
     *  
     * @param r The Runnable that will be executed.
     * 
     * @return Returns true if the Runnable was successfully placed in to the 
     *         message queue.  Returns false on failure, usually because the
     *         looper processing the message queue is exiting.
     */
    public final boolean post(Runnable r)
    {
       return  sendMessageDelayed(getPostMessage(r), 0);
    }
```

这里调用getPostMessage方法，将runnable改为Message，然后执行sendMessageDelayed

### 第二种：public final boolean postAtTime(Runnable r, long uptimeMillis)

```
    public final boolean postAtTime(Runnable r, long uptimeMillis)
    {
        return sendMessageAtTime(getPostMessage(r), uptimeMillis);
    }
```

该方法和是在固定的时间点执行

### 第三种：public final boolean postAtTime(Runnable r, Object token, long uptimeMillis)

```
    public final boolean postAtTime(Runnable r, Object token, long uptimeMillis)
    {
        return sendMessageAtTime(getPostMessage(r, token), uptimeMillis);
    }
```

和第二种方式类似，token的作用是用于cancel这个message

### 第四种：public final boolean postDelayed(Runnable r, long delayMillis)

```
    public final boolean postDelayed(Runnable r, long delayMillis)
    {
        return sendMessageDelayed(getPostMessage(r), delayMillis);
    }
```

该方式和第一种类似，加上了delaymillis

### 第五种：public final boolean postDelayed(Runnable r, Object token, long delayMillis)

```
    public final boolean postDelayed(Runnable r, Object token, long delayMillis)
    {
        return sendMessageDelayed(getPostMessage(r, token), delayMillis);
    }
```

加上了token的第四种方式

### 第六种：public final boolean postAtFrontOfQueue(Runnable r)

```
    public final boolean postAtFrontOfQueue(Runnable r)
    {
        return sendMessageAtFrontOfQueue(getPostMessage(r));
    }
```

往头部插入message，虽好不要使用，特殊场景除外，会造成顺序问题以及别的影响


### 第七种：public final boolean executeOrSendMessage(Message msg)

```
    public final boolean executeOrSendMessage(Message msg) {
        if (mLooper == Looper.myLooper()) {
            dispatchMessage(msg);
            return true;
        }
        return sendMessage(msg);
    }
```
如果当前的线程就是handler的线程，直接执行message，否则塞入该handler的线程

### sendMessageDelayed

```
    /**
     * Enqueue a message into the message queue after all pending messages
     * before (current time + delayMillis). You will receive it in
     * {@link #handleMessage}, in the thread attached to this handler.
     *  
     * @return Returns true if the message was successfully placed in to the 
     *         message queue.  Returns false on failure, usually because the
     *         looper processing the message queue is exiting.  Note that a
     *         result of true does not mean the message will be processed -- if
     *         the looper is quit before the delivery time of the message
     *         occurs then the message will be dropped.
     */
    public final boolean sendMessageDelayed(Message msg, long delayMillis)
    {
        if (delayMillis < 0) {
            delayMillis = 0;
        }
        return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
    }
```

事实上sendMessageDelayed也是执行了sendMessageAtTime

SystemClock.uptimeMillis()是指从开机到现在的毫秒数，并非System.currentTimeMillis()这种从1970年1月1日 UTC到现在的毫秒数

### sendMessageAtTime

```
    public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, uptimeMillis);
    }
```

如果messageQueue不存在就报错，否则就执行enqueueMessage操作，mq的初始化在handler构造器里面赋值的。

### enqueueMessage

```
    private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }
```

如果消息是异步消息，设置一下状态，然后塞到MessageQueue里面

# Looper概览

刚才看完handler相关，其实looper大部分的已经经过了。在提一下。

## prepare()

prepare事实上有一个参数，是是否允许退出，mainlooper不允许，其他的是允许的

prepare核心是创建一个looper塞到threadlocal里面

## myLooper()

Looper.myLooper()就是当前线程的looper

## 核心方法loop()

```
    /**
     * Run the message queue in this thread. Be sure to call
     * {@link #quit()} to end the loop.
     */
    public static void loop() {
        //获取当前线程的looper
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        //获取该looper对应的mq
        final MessageQueue queue = me.mQueue;

        // Make sure the identity of this thread is that of the local process,
        // and keep track of what that identity token actually is.
        //重设置远端的uid和pid，用当前本地进程的uid和pid替代
        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();

        // Allow overriding a threshold with a system prop. e.g.
        // adb shell 'setprop log.looper.1000.main.slow 1 && stop && start'
        final int thresholdOverride =
                SystemProperties.getInt("log.looper."
                        + Process.myUid() + "."
                        + Thread.currentThread().getName()
                        + ".slow", 0);
        //用于监控慢函数
        boolean slowDeliveryDetected = false;

        for (;;) {
            Message msg = queue.next(); // might block
            //next之后拿到的msg是null，这时候代表其实是要退出的，因为普通时候这个next是会阻塞的
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }
            // This must be in a local variable, in case a UI event sets the logger
            final Printer logging = me.mLogging;
            if (logging != null) {
                logging.println(">>>>> Dispatching to " + msg.target + " " +
                        msg.callback + ": " + msg.what);
            }
            final long traceTag = me.mTraceTag;
            long slowDispatchThresholdMs = me.mSlowDispatchThresholdMs;
            long slowDeliveryThresholdMs = me.mSlowDeliveryThresholdMs;
            if (thresholdOverride > 0) {
                slowDispatchThresholdMs = thresholdOverride;
                slowDeliveryThresholdMs = thresholdOverride;
            }
            final boolean logSlowDelivery = (slowDeliveryThresholdMs > 0) && (msg.when > 0);
            final boolean logSlowDispatch = (slowDispatchThresholdMs > 0);

            final boolean needStartTime = logSlowDelivery || logSlowDispatch;
            final boolean needEndTime = logSlowDispatch;

            if (traceTag != 0 && Trace.isTagEnabled(traceTag)) {
                Trace.traceBegin(traceTag, msg.target.getTraceName(msg));
            }

            final long dispatchStart = needStartTime ? SystemClock.uptimeMillis() : 0;
            final long dispatchEnd;
            try {
                //target就是handler，传给handler去处理
                msg.target.dispatchMessage(msg);
                dispatchEnd = needEndTime ? SystemClock.uptimeMillis() : 0;
            } finally {
                if (traceTag != 0) {
                    Trace.traceEnd(traceTag);
                }
            }
            if (logSlowDelivery) {
                if (slowDeliveryDetected) {
                    if ((dispatchStart - msg.when) <= 10) {
                        Slog.w(TAG, "Drained");
                        slowDeliveryDetected = false;
                    }
                } else {
                    if (showSlowLog(slowDeliveryThresholdMs, msg.when, dispatchStart, "delivery",
                            msg)) {
                        // Once we write a slow delivery log, suppress until the queue drains.
                        slowDeliveryDetected = true;
                    }
                }
            }
            if (logSlowDispatch) {
                showSlowLog(slowDispatchThresholdMs, dispatchStart, dispatchEnd, "dispatch", msg);
            }

            if (logging != null) {
                logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
            }

            // Make sure that during the course of dispatching the
            // identity of the thread wasn't corrupted.
            final long newIdent = Binder.clearCallingIdentity();
            if (ident != newIdent) {
                Log.wtf(TAG, "Thread identity changed from 0x"
                        + Long.toHexString(ident) + " to 0x"
                        + Long.toHexString(newIdent) + " while dispatching to "
                        + msg.target.getClass().getName() + " "
                        + msg.callback + " what=" + msg.what);
            }
            //清理msg
            msg.recycleUnchecked();
        }
    }
```
这里就是进入到消息循环中去了，它不断地从消息队列mQueue中去获取下一个要处理的消息msg，如果消息的target成员变量为null，就表示要退出消息循环了，否则的话就要调用这个target对象的dispatchMessage成员函数来处理这个消息，这个target对象的类型为Handler

Looper核心没有太多东西，主要是Loop。

还有一个问题，loop是谁执行的？

主线程的looper其实在main函数里面执行的。

子线程需要改写一下，手动调用

```
  *  class LooperThread extends Thread {
  *      public Handler mHandler;
  *
  *      public void run() {
  *          Looper.prepare();
  *
  *          mHandler = new Handler() {
  *              public void handleMessage(Message msg) {
  *                  // process incoming messages here
  *              }
  *          };
  *
  *          Looper.loop();
  *      }
```

# MessageQueue

MessageQueue可以说是最重要的，其next方法核心才是loop不阻塞的根源。

## 构造

```
    MessageQueue(boolean quitAllowed) {
        mQuitAllowed = quitAllowed;
        mPtr = nativeInit();
    }
```

quitAllowed主线程是false，子线程是true。

nativeInit是native的构造方法，其会在native这一层构造一个nativeMessageQueue。

同时NativeMessageQueue也会创建一个Native的Looper，该looper的实现在jni层并非java层的looper

创建完之后会将nativeMQ存入到mPtr中

### JNI层的Looper

```
Looper::Looper(bool allowNonCallbacks) :
	mAllowNonCallbacks(allowNonCallbacks),
	mResponseIndex(0) {
	int wakeFds[2];
	int result = pipe(wakeFds);
	......
 
	mWakeReadPipeFd = wakeFds[0];
	mWakeWritePipeFd = wakeFds[1];
 
	......
 
#ifdef LOOPER_USES_EPOLL
	// Allocate the epoll instance and register the wake pipe.
	mEpollFd = epoll_create(EPOLL_SIZE_HINT);
	......
 
	struct epoll_event eventItem;
	memset(& eventItem, 0, sizeof(epoll_event)); // zero out unused members of data field union
	eventItem.events = EPOLLIN;
	eventItem.data.fd = mWakeReadPipeFd;
	result = epoll_ctl(mEpollFd, EPOLL_CTL_ADD, mWakeReadPipeFd, & eventItem);
	......
#else
	......
#endif
 
	......
}
```

这个Looper重要的就是

```
int wakeFds[2];
int result = pipe(wakeFds);
......
 
mWakeReadPipeFd = wakeFds[0];
mWakeWritePipeFd = wakeFds[1];
```
使用Pipe创建了一个管道对应的两个句柄，一个读一个写，中间还需要加一层等待状态的机制。就是Epoll，Epoll是select/poll的增强版本。

```
	mEpollFd = epoll_create(EPOLL_SIZE_HINT);
```

这段代码是注册了一个EpollFd

使用Pipe和Epoll的核心关键作用，也就是这个native的Looper的作用。

这个native的Looper的作用是，当Java层的消息队列中没有消息时，就使Android应用程序主线程进入等待状态，而当Java层的消息队列中来了新的消息后，就唤醒Android应用程序的主线程来处理这个消息。

## MessageQueue#next()

```
    Message next() {
        // Return here if the message loop has already quit and been disposed.
        // This can happen if the application tries to restart a looper after quit
        // which is not supported.
        final long ptr = mPtr;
        if (ptr == 0) {
            return null;
        }

        int pendingIdleHandlerCount = -1; // -1 only during first iteration
        int nextPollTimeoutMillis = 0;
        for (;;) {
            if (nextPollTimeoutMillis != 0) {
                Binder.flushPendingCommands();
            }
            //关键点方式
            nativePollOnce(ptr, nextPollTimeoutMillis);
            synchronized (this) {
                // Try to retrieve the next message.  Return if found.
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                Message msg = mMessages;
                //同步屏障相关
                if (msg != null && msg.target == null) {
                    // Stalled by a barrier.  Find the next asynchronous message in the queue.
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                //找到了这个msg，进行处理
                if (msg != null) {
                    //时间延时问题
                    if (now < msg.when) {
                        // Next message is not ready.  Set a timeout to wake up when it is ready.
                        //改变一下poll time out
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        // Got a message.
                        mBlocked = false;
                        //之前的msg不是空，直接拼接
                        if (prevMsg != null) {
                            prevMsg.next = msg.next;
                        //之前的msg是空，直接赋值
                        } else {
                            mMessages = msg.next;
                        }
                        msg.next = null;
                        if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                        msg.markInUse();
                        //返回msg
                        return msg;
                    }
                } else {
                    // No more messages.
                    //没有msg，mills改成-1
                    nextPollTimeoutMillis = -1;
                }

                // Process the quit message now that all pending messages have been handled.
                if (mQuitting) {
                    dispose();
                    return null;
                }
                //处理idlehandler
                // If first time idle, then get the number of idlers to run.
                // Idle handles only run if the queue is empty or if the first message
                // in the queue (possibly a barrier) is due to be handled in the future.
                if (pendingIdleHandlerCount < 0
                        && (mMessages == null || now < mMessages.when)) {
                    pendingIdleHandlerCount = mIdleHandlers.size();
                }
                if (pendingIdleHandlerCount <= 0) {
                    // No idle handlers to run.  Loop and wait some more.
                    mBlocked = true;
                    continue;
                }

                if (mPendingIdleHandlers == null) {
                    mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
                }
                mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
            }

            // Run the idle handlers.
            // We only ever reach this code block during the first iteration.
            //逐个处理idlehandler
            for (int i = 0; i < pendingIdleHandlerCount; i++) {
                final IdleHandler idler = mPendingIdleHandlers[i];
                mPendingIdleHandlers[i] = null; // release the reference to the handler

                boolean keep = false;
                try {
                    keep = idler.queueIdle();
                } catch (Throwable t) {
                    Log.wtf(TAG, "IdleHandler threw exception", t);
                }

                if (!keep) {
                    synchronized (this) {
                        mIdleHandlers.remove(idler);
                    }
                }
            }

            // Reset the idle handler count to 0 so we do not run them again.
            pendingIdleHandlerCount = 0;

            // While calling an idle handler, a new message could have been delivered
            // so go back and look again for a pending message without waiting.
            nextPollTimeoutMillis = 0;
        }
    }
```

next的一整体主要是返回了msg，以前也只会这些，其实还有idleHandler的处理，处理就是直接处理的过程，根据queueIdle的返回值，false就会移除，不是false就不移除。

最核心的是nativePollOnce()和nextPollTimeoutMillis的方法把控

### nativePollOnce(mPtr, nextPollTimeoutMillis)

mPtr：之前的native的mq的应用。
nextPollTimeoutMillis：表示如果当前消息队列中没有消息，它要等待的时候

#### nextPollTimeoutMillis

postDelay的message什么时候会执行呢？这个其实就是取决于这个nextPollTimeoutMillis参数
取出来一条消息发现这条消息是未来需要执行的，计算好时间，会作为nextPollTimeoutMills的参数启动下一次nativePollOnce

另外可以看到，刚开始的时候nextPollTimeoutMillis 是0，其实就代表第一次的时候直接poll。

另外还可以看到，当没有消息的时候，nextPollTimeoutMillis是-1，这时候代表空闲

#### nativePollOnce 的native代码

```
static void android_os_MessageQueue_nativePollOnce(JNIEnv* env, jobject obj,
        jint ptr, jint timeoutMillis) {
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    nativeMessageQueue->pollOnce(timeoutMillis);
}
```

```
void NativeMessageQueue::pollOnce(int timeoutMillis) {
    mLooper->pollOnce(timeoutMillis);
}
```

```
int Looper::pollOnce(int timeoutMillis, int* outFd, int* outEvents, void** outData) {
	int result = 0;
	for (;;) {
		......
 
		if (result != 0) {
			......
 
			return result;
		}
 
		result = pollInner(timeoutMillis);
	}
}
```

```
int Looper::pollInner(int timeoutMillis) {
	......
 
	int result = ALOOPER_POLL_WAKE;
 
	......
 
#ifdef LOOPER_USES_EPOLL
	struct epoll_event eventItems[EPOLL_MAX_EVENTS];
	int eventCount = epoll_wait(mEpollFd, eventItems, EPOLL_MAX_EVENTS, timeoutMillis);
	bool acquiredLock = false;
#else
	......
#endif
 
	if (eventCount < 0) {
		if (errno == EINTR) {
			goto Done;
		}
 
		LOGW("Poll failed with an unexpected error, errno=%d", errno);
		result = ALOOPER_POLL_ERROR;
		goto Done;
	}
 
	if (eventCount == 0) {
		......
		result = ALOOPER_POLL_TIMEOUT;
		goto Done;
	}
 
	......
 
#ifdef LOOPER_USES_EPOLL
	for (int i = 0; i < eventCount; i++) {
		int fd = eventItems[i].data.fd;
		uint32_t epollEvents = eventItems[i].events;
		if (fd == mWakeReadPipeFd) {
			if (epollEvents & EPOLLIN) {
				awoken();
			} else {
				LOGW("Ignoring unexpected epoll events 0x%x on wake read pipe.", epollEvents);
			}
		} else {
			......
		}
	}
	if (acquiredLock) {
		mLock.unlock();
	}
Done: ;
#else
	......
#endif
 
	......
 
	return result;
}
```

nativePollOnce => NativeMessageQueue::android_os_MessageQueue_nativePollOnce => Looper::pollOnce => Looper::pollInner

过程虽然长，核心就是将nextPollTimeoutMillis传到native层，然后进行设置一次epoll_wait等待操作。

```
void Looper::awoken() {
	......
 
	char buffer[16];
	ssize_t nRead;
	do {
		nRead = read(mWakeReadPipeFd, buffer, sizeof(buffer));
	} while ((nRead == -1 && errno == EINTR) || nRead == sizeof(buffer));
}
```
、
wake操作如上，直接读信道

话虽然是这么讲，但是难免会有队列中只有一个未来发生的msg，此时头插一个新的msg。此时会直接wake。

### messageQueue#IdleHandler

之前看到在设置nextPollTimeoutMillis = -1的时候和msg在未来才会发生的时候，并不会return，而是会继续走下去。

```

                // If first time idle, then get the number of idlers to run.
                // Idle handles only run if the queue is empty or if the first message
                // in the queue (possibly a barrier) is due to be handled in the future.
                if (pendingIdleHandlerCount < 0
                        && (mMessages == null || now < mMessages.when)) {
                    pendingIdleHandlerCount = mIdleHandlers.size();
                }
                if (pendingIdleHandlerCount <= 0) {
                    // No idle handlers to run.  Loop and wait some more.
                    mBlocked = true;
                    continue;
                }

                if (mPendingIdleHandlers == null) {
                    mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
                }
                mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
```
这里会遍历mIdleHandlers的接口，然后生成mPendingIdleHandlers，注意，很明显，idleHandler最大数量是4，也就是多余的事实上是会被去掉的

之后便会执行idlehandlers里面的pendingIdleHandlers

## MessageQueue#同步屏障

next方法中获得消息之后

```
                if (msg != null && msg.target == null) {
                    // Stalled by a barrier.  Find the next asynchronous message in the queue.
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
```

这里会有一个鉴别是否是开启同步屏障。


`同步屏障`其实就是一个message的target为空的msg而已。

如果target为空，这个if就会走进去，他会不断地往后遍历，直到寻找到异步msg，或者遍历到尾部。

可是每次native

### postSyncBarrier & removeSyncBarrier

```
public int postSyncBarrier() {
    return postSyncBarrier(SystemClock.uptimeMillis());
}
​
private int postSyncBarrier(long when) {
    // Enqueue a new sync barrier token.
    // We don't need to wake the queue because the purpose of a barrier is to stall it.
    synchronized (this) {
        final int token = mNextBarrierToken++;
        final Message msg = Message.obtain();
        msg.markInUse();
        msg.when = when;
        msg.arg1 = token;
​
        Message prev = null;
        Message p = mMessages;
        if (when != 0) {
            while (p != null && p.when <= when) {
                prev = p;
                p = p.next;
            }
        }
        if (prev != null) { // invariant: p == prev.next
            msg.next = p;
            prev.next = msg;
        } else {
            msg.next = p;
            mMessages = msg;
        }
        return token;
    }
}
```

这个方法会构造一个没有target的msg传入到mq中，并且获取一个token

由于塞入一个同步屏障且没有异步msg在mq中的话，主线程会一直休眠，直到处理一个异步mq。因此不需要的时候记得务必要remove

### 同步屏障会导致idlehandler无法被调用

当mMessages是同步屏障，且后续没有异步消息，那么获取异步消息和获取同步消息这两步都会失败了，即nextPollTimeoutMillis会被赋值为-1，表示无限制的休眠

pendingIdleHandlerCount 默认是-1，所以会尝试着赋值。其中，由于同步屏障的存在，所以mMessages肯定不为空，一旦屏障的时间比现在要早，那么就不会进行赋值，之后也就不会走idle的实例运行。

### 判断mq是否是idle状态

```
    public boolean isIdle() {
        synchronized (this) {
            final long now = SystemClock.uptimeMillis();
            return mMessages == null || now < mMessages.when;
        }
    }
```

就是当前信息是空且是未来才会发生，这时候就是idle状态

### enqueueMessage

```
    boolean enqueueMessage(Message msg, long when) {
        if (msg.target == null) {
            throw new IllegalArgumentException("Message must have a target.");
        }
        if (msg.isInUse()) {
            throw new IllegalStateException(msg + " This message is already in use.");
        }

        synchronized (this) {
            if (mQuitting) {
                IllegalStateException e = new IllegalStateException(
                        msg.target + " sending message to a Handler on a dead thread");
                Log.w(TAG, e.getMessage(), e);
                msg.recycle();
                return false;
            }

            msg.markInUse();
            msg.when = when;
            Message p = mMessages;
            boolean needWake;
            if (p == null || when == 0 || when < p.when) {
                // New head, wake up the event queue if blocked.
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
                // Inserted within the middle of the queue.  Usually we don't have to wake
                // up the event queue unless there is a barrier at the head of the queue
                // and the message is the earliest asynchronous message in the queue.
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
                for (;;) {
                    prev = p;
                    p = p.next;
                    if (p == null || when < p.when) {
                        break;
                    }
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                msg.next = p; // invariant: p == prev.next
                prev.next = msg;
            }

            // We can assume mPtr != 0 because mQuitting is false.
            if (needWake) {
                nativeWake(mPtr);
            }
        }
        return true;
    }

```

每次都会按照时间进行重排序

block有两种情况，一种是当前mq就是空或者老的msg的whn是在过去，这时候需要塞入尾部或者说重创一个头，然后如果mq是阻塞状态，就需要唤醒

第二种是塞入的是异步消息并且mq的消息头是同步屏障，且mq的队列内部没有异步消息，这时候也需要一次唤醒

