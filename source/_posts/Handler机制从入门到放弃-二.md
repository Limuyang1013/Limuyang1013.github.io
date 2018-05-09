---
title: Handler机制从入门到放弃(二)
date: 2016-10-29T22:48:19.000Z
categories: Android
tags:
  - Android
  - Handler
  - 源码分析
---

#### 从注释看起

Hander的源码只有不到800行，而且大多数代码相对来说还是比较好理解的，尤其是相对于其他更加接近底层的代码来说，在看源码时候有一点挺重要的就是不要忽略注释的作用，Handler类开头有这么几行注释：

```javascript
<p>There are two main uses for a Handler: (1) to schedule messages and
 runnables to be executed as some point in the future; and (2) to enqueue
 an action to be performed on a different thread than your own.
```

<!-- more --> 归纳一下就是：

- 安排消息和任务在将来的某一个点执行
- 使一个动作进入队列为了能够在另一个线程中执行

回顾一下我们为什么要用Handler：

> 在Android中，当要更新UI的时候，我们必须要在主线程中进行更新，原因时当主线程被阻塞了5s以上就会出现ANR异常，会导致程序崩溃。所以一些耗时的操作必须要放在子线程中，但是在子线程中又不能做更新UI的操作，所以为了解决这个问题，Android设计了handler机制。

这么一对比，很容易的印证了这段话：使一个动作进入队列在另一个线程中执行：这不就是异步执行耗时任务么；安排消息和任务在将来的某一个点执行：联想一下postDelayed之类的延时操作的方法，或者给出一个很常见的例子，比如说引导页延时启动：

```java
new Handler().postDelayed(new Runnable() {
            @Override
            public void run() {
                Log.d("ThreadName 1",Thread.currentThread().getName());
                //第一次登陆扫描本地音乐
                if (SPUtils.getValue(SplashActivity.this, "isFirst", "First", true)) {
                    new Thread(new Runnable() {
                        @Override
                        public void run() {
                            //耗时操作
                            //清空表
                            Log.d("ThreadName 2",Thread.currentThread().getName());
                            DataSupport.deleteAll(MusicInfoDetail.class);
                            MusicUtils.scanMusic(SplashActivity.this, musicInfo);
                            DataSupport.saveAll(musicInfo);
                            SPUtils.putValue(SplashActivity.this, "isFirst", "First", false);
                        }
                    }).start();
               }, 2000);
```

这里是我自己的Demo里面的一部分代码，这里使用`postDelayed`延时2s启动，然后在子线程执行更新数据库的操作，很好的印证了上面两点。

#### 创建Handler

在上一篇文章Handler机制从入门到放弃(一)里面我们已经演示了两种创建Handler的方法并且给出了部分实际操作的代码，但是都是在主线程也就是UI线程创建的，我们可以尝试一下在子线程中创建Handler：

```java
public class MainActivity extends AppCompatActivity {

    private Handler mainHandler;
    private Handler childHandler;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        mainHandler=new Handler();
        new Thread(new Runnable() {
            @Override
            public void run() {
                childHandler=new Handler();
            }
        }).start();
    }
}
```

运行一下，果不其然代码蹦了：

![Crash](http://oasusatoz.bkt.clouddn.com/16-10-28/55948496.jpg)

报错信息：

```
java.lang.RuntimeException: Can't create handler inside thread that has not called Looper.prepare()
```

告诉我们说在Thread里面创建Handler需要调用`Looper.prepare( )`，那把这一句加上试试：

```java
public class MainActivity extends AppCompatActivity {

    private Handler mainHandler;
    private Handler childHandler;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        mainHandler=new Handler();
        new Thread(new Runnable() {
            @Override
            public void run() {
                Looper.prepare();
                childHandler=new Handler();
            }
        }).start();
    }
}
```

![运行结果](http://oasusatoz.bkt.clouddn.com/16-10-28/83297625.jpg)

果然很成功的运行了，但是这是为什么，来看一下Handler的源码：

这里提供一个简便的方法，为了快速找到原因可以在打开的源码(我这里使用sublimeText查看)里使用ctrl+f快捷键搜索Looper.prepare( )出现的地方：

```java
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
                "Can't create handler inside thread that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
```

这是Handler的其中一个构造方法，看到这么一段：

```java
mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread that has not called Looper.prepare()");
        }
```

在构造方法里通过`Looper.myLooper()`获取到一个Looper对象mLooper，如果为空则报错，找到`Looper.myLooper()`方法：

```java
/**
     * Return the Looper object associated with the current thread.  Returns
     * null if the calling thread is not associated with a Looper.
     */
    public static @Nullable Looper myLooper() {
        return sThreadLocal.get();
    }
```

注释给出的解释是这个方法回返回跟当前线程相关联的Looper对象，如果没有则返回空，还是没找到答案，接着找Looper类里面对sThreadLocal的定义：

```java
// sThreadLocal.get() will return null unless you've called prepare().
    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
```

又是注释里面告诉了我们重要信息，这里告诉我们只有你调用了`Looper.prepare()`方法`sThreadLocal.get()`才不会返回空，那么说来说去还是要看`Looper.prepare()`的代码：

```java
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

看下面那个，Looper.prepare()调用了prepare()的重载方法prepare(boolean quitAllowed)并且传入了true参数，这个方法判断sThreadLocal.get()是否会返回一个Looper对象，如果没有的话就set一个新的Looper进去，如果已经有了再调用prepare()方法的话就会报错，不信邪的可以在mainHandler创建之前也调用一个Looper.prepare()，控制台就会出现这个错误：

![Crash](http://oasusatoz.bkt.clouddn.com/4.png)

那么问题来了，为什么我们在主线程创建Handler不需要调用`Looper.prepare()`，而在子线程中需要呢，可以合理的猜想是不是系统给我们主动调用了，毕竟我们大部分的操作还是在主线程上，每次都要那么`Looper.prepare()`来一次多麻烦，有了猜想还要去源码寻求验证，主线程是ActivityThread，从ActivityThread类里搜索相关信息，用跟上面一样的方法：

```java
public static void main(String[] args) {
        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ActivityThreadMain");
        SamplingProfilerIntegration.start();

        // CloseGuard defaults to true and can be quite spammy.  We
        // disable it here, but selectively enable it later (via
        // StrictMode) on debug builds, but using DropBox, not logs.
        CloseGuard.setEnabled(false);

        Environment.initForCurrentUser();

        // Set the reporter for event logging in libcore
        EventLogger.setReporter(new EventLoggingReporter());

        // Make sure TrustedCertificateStore looks in the right place for CA certificates
        final File configDir = Environment.getUserConfigDirectory(UserHandle.myUserId());
        TrustedCertificateStore.setDefaultUserDirectory(configDir);

        Process.setArgV0("<pre-initialized>");

        Looper.prepareMainLooper();

        ActivityThread thread = new ActivityThread();
        thread.attach(false);

        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }

        if (false) {
            Looper.myLooper().setMessageLogging(new
                    LogPrinter(Log.DEBUG, "ActivityThread"));
        }

        // End of event ActivityThreadMain.
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
        Looper.loop();

        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
```

抓重点：

```
Looper.prepareMainLooper();
```

找到Looper类中关于这个方法的定义：

```
/**
     * Initialize the current thread as a looper, marking it as an
     * application's main looper. The main looper for your application
     * is created by the Android environment, so you should never need
     * to call this function yourself.  See also: {@link #prepare()}
     */
public static void prepareMainLooper() {
        prepare(false);
        synchronized (Looper.class) {
            if (sMainLooper != null) {
                throw new IllegalStateException("The main Looper has already been prepared.");
            }
            sMainLooper = myLooper();
        }
    }
```

这里又会主动调用`prepare(boolean quitAllowed)`方法，通过注释也了解到我们不需要主动去创建UI线程的looper，系统自动会给我们创建好了，这里印证了前面的猜想。

这里得出一个结论：

**在主线程中可以直接创建Handler对象，而在子线程中需要先调用Looper.prepare()才能创建Handler对象。**

这里先不管Looper是什么，暂时知道有这个东西，下面可以看一下如何发送消息。

#### 如何发送消息

这里就用到了第二种创建Handler的方法：

```java
Handler myHandler = new Handler() {  
          public void handleMessage(Message msg) {   
               switch (msg.what) {   
                   //根据参数进行操作
                         break;   
               }   
               super.handleMessage(msg);   
          }   
     };  
  //其他地方调用
myHandler.sendMessage(xxx);
```

这里的其他地方调用指的就是在子线程里面，当我们在子线程里面执行完耗时操作之后如果需要传递一些数据给主线程，比如通知主线程更新UI之类的，就可以这么做：

```java
final  Handler myHandler = new Handler() {
            public void handleMessage(Message msg) {
                switch (msg.what) {
                    //根据参数进行操作
                }
            }
        };

        new Thread(new Runnable() {
            @Override
            public void run() {
               Message msg=new Message();
                //what是用户自定义的识别码
                msg.what=1;
                //通过arg1和arg2可以给Message传递简单的int型数据
                msg.arg1=123;
                msg.arg2=456;
                //通过给obj赋值Object类型传递向Message传入任意数据
                msg.obj=null;
                myHandler.sendMessage(msg);
            }
        }).start();
```

当然除了传递这些简单数据之外Message类还能以setData方式携带Bundle数据：

```java
Bundle bundle = new Bundle();  
    bundle.putString("data", "data");  
    message.setData(bundle);
```

我们看到这里是在子线程中调用了`sendMessage(msg)`方法，然而我们却在主线程中使用`handleMessage(Message msg)`接受消息，这之间一定发生了一些不可描述的事情，让我们来找找看，当然除了`sendMessage(msg)`方法Message类还有许多其他发送消息的方法：

```java
public final boolean sendMessage(Message msg)
    {
        return sendMessageDelayed(msg, 0);
    }

  public final boolean sendEmptyMessage(int what)
    {
        return sendEmptyMessageDelayed(what, 0);
    }

  public final boolean sendEmptyMessageDelayed(int what, long delayMillis) {
        Message msg = Message.obtain();
        msg.what = what;
        return sendMessageDelayed(msg, delayMillis);
    }

  public final boolean sendEmptyMessageAtTime(int what, long uptimeMillis) {
        Message msg = Message.obtain();
        msg.what = what;
        return sendMessageAtTime(msg, uptimeMillis);
    }

 .....
```

巧的是，这些方法无论转折多少次都走向了同一个方法：

```java
/**
     * Enqueue a message into the message queue after all pending messages
     * before the absolute time (in milliseconds) <var>uptimeMillis</var>.
     * <b>The time-base is {@link android.os.SystemClock#uptimeMillis}.</b>
     * Time spent in deep sleep will add an additional delay to execution.
     * You will receive it in {@link #handleMessage}, in the thread attached
     * to this handler.
     *
     * @param uptimeMillis The absolute time at which the message should be
     *         delivered, using the
     *         {@link android.os.SystemClock#uptimeMillis} time-base.
     *         
     * @return Returns true if the message was successfully placed in to the
     *         message queue.  Returns false on failure, usually because the
     *         looper processing the message queue is exiting.  Note that a
     *         result of true does not mean the message will be processed -- if
     *         the looper is quit before the delivery time of the message
     *         occurs then the message will be dropped.
     */
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

这个方法接受两个参数，msg和`uptimeMillis`，msg就是我们需要传递的消息，`uptimeMillis`则是发送消息时候的绝对时刻，它的值等于自系统开机到当前时间的毫秒数再加上延迟时间，这个延迟时间就是我们调用sendxxxDelayed里面传入的时间参数，这个方法会把一个消息放入消息队列(message queue)，然后把这个方法的两个参数加上新建的MessageQueue 对象传入`enqueueMessage(queue, msg, uptimeMillis)`方法里，从字面上理解MessageQueue 是一个消息队列，那么队列就会有入队和出队的方法，这个`enqueueMessage(queue, msg, uptimeMillis)`应该就是入队的方法：

```java
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }
```

没有发现什么，这里又调用了`enqueueMessage(msg, uptimeMillis)`方法，这个方法在`MessageQueue`类里面：

```java
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

代码有点长，一步一步看，先看前面一部分：

```java
if (msg.target == null) {
            throw new IllegalArgumentException("Message must have a target.");
        }
        if (msg.isInUse()) {
            throw new IllegalStateException(msg + " This message is already in use.");
        }
```

这里判断了一下msg.target对象是否为空，还记得之前的`enqueueMessage(queue, msg, uptimeMillis)`方法吗：

```java
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }
```

这个方法是在Handler中执行的，这里把一个this对象赋值给`msg.target`，那么从Message类找一下这个target到底是什么，找到这个：

```java
/*package*/ Handler target;
```

这样脉络就很清晰了，这里是把Handler跟Message对象绑定起来，接着往下看：

```java
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
```

这个`msg.when`就是用传入的`uptimeMillis`参数赋值，表示入队时间，看到这个if判断：

```java
if (p == null || when == 0 || when < p.when) {
                // New head, wake up the event queue if blocked.
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
            ...
            }
```

有人可能会好奇这个when怎么会为0呢，这里提一嘴，Handler除了有正常的sendMessage之流的方法还有一个比较特殊的方法：

```java
public final boolean sendMessageAtFrontOfQueue(Message msg) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, 0);
    }
```

这个方法特殊在什么地方呢，打个比方，如果说我们正常的`sendMessage`之流的方法是一群正常排队的人，按照来的时间先后有序排队，但是`sendMessageAtFrontOfQueue`就是那种个别不老实的，它能直接插队到最前面，然后他传递的`uptimeMillis`为0，这也是唯一一个特殊的发送消息的方法。

这个判断语句成立的条件有三点：`p == null || when == 0 || when < p.when`

- p == null说明当前looper处于空闲状态，也就是没有什么消息需要处理
- when == 0说明有消息插队插到了MessageQueue最前面
- when < p.when指的是新入队的消息队列需要排队的时间比正在执行的消息排队的时间短

综合来说就是，如果这时候新进来一个消息，这时候消息队列里面没有需要执行的消息，或者新进来的这个消息是通过`sendMessageAtFrontOfQueue(Message msg)`方法传进来的，或者说新进来的这个消息需要等待的时间比之前在等待的消息等待的时间短，那么就把这个消息插入链表的表头，此时系统会唤醒这个消息队列无论队列是否堵塞。

```java
// Got a message.
 mBlocked = false;
```

这一行代码说明只要消息队列有消息，这个队列就不阻塞，然后把这个布尔值传递：

```java
boolean needWake
needWake = mBlocked;
```

那么这一块代码就打通了，下面这块else语句块：

```java
else {
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
```

讲的是如何把消息插入链表的内部，这时候就不需要去调整唤醒消息队列的时间，因为唤醒的时间是跟表头有关的，这样整个入队的操作差不多就过了一遍.

#### 出队操作

既然有入队操作那么肯定也有出队操作，如果你还记得我们最开始使用的Looper类的话，那么这里不妨直接告诉你，出队的方法就在Looper类里面，这里有个`loop()`方法：

```java
/**
     * Run the message queue in this thread. Be sure to call
     * {@link #quit()} to end the loop.
     */
    public static void loop() {
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;
        // Make sure the identity of this thread is that of the local process,
        // and keep track of what that identity token actually is.
        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();
        //死循环
        for (;;) {
        //把消息从队列取出
            Message msg = queue.next(); // might block
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
            if (traceTag != 0) {
                Trace.traceBegin(traceTag, msg.target.getTraceName(msg));
            }
            try {
                msg.target.dispatchMessage(msg);
            } finally {
                if (traceTag != 0) {
                    Trace.traceEnd(traceTag);
                }
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
            msg.recycleUnchecked();
        }
    }
```

注意这两段代码：

```java
final Looper me = myLooper();
final MessageQueue queue = me.mQueue;
```

之前说过一个线程必须有一个Looper，这里不仅获取到了Looper，还获取到了当前线程绑定的MessageQueue也就是消息队列，然后`loop()`方法最开始是判断当前线程是否有Looper对象，之后进入一个死循环，在循环体内不断的从消息队列(Message queue)中取出消息对象，为什么这么说，看这个`next()`方法：

```java
Message next()
{
    int pendingIdleHandlerCount = -1; // -1 only during first iteration
    int nextPollTimeoutMillis = 0;

    for (;;) {
        . . . . . .
        nativePollOnce(mPtr, nextPollTimeoutMillis);    // 阻塞于此
        . . . . . .
            // 获取next消息，如能得到就返回之。
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;  // 先尝试拿消息队列里当前第一个消息

            if (msg != null && msg.target == null) {
                // 如果从队列里拿到的msg是个“同步分割栏”，那么就寻找其后第一个“异步消息”
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }

            if (msg != null) {
                if (now < msg.when) {
                    // Next message is not ready.  Set a timeout to wake up when it is ready.
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now,
                                                                   Integer.MAX_VALUE);
                } else {
                    // Got a message.
                    mBlocked = false;
                    if (prevMsg != null) {
                        prevMsg.next = msg.next;
                    } else {
                        mMessages = msg.next;  // 重新设置一下消息队列的头部
                    }
                    msg.next = null;
                    if (false) Log.v("MessageQueue", "Returning message: " + msg);
                    msg.markInUse();
                    return msg;     // 返回得到的消息对象
                }
            } else {
                // No more messages.
                nextPollTimeoutMillis = -1;
            }

            // Process the quit message now that all pending messages have been handled.
            if (mQuitting) {
                dispose();
                return null;
            }
            if (pendingIdleHandlerCount < 0
                        && (mMessages == null || now < mMessages.when)) {
                    pendingIdleHandlerCount = mIdleHandlers.size();
            }
            if (pendingIdleHandlerCount <= 0) {
                // No idle handlers to run.  Loop and wait some more.
                mBlocked = true;
                continue;
            }
        . . . . . .
        // 处理idle handlers部分
        for (int i = 0; i < pendingIdleHandlerCount; i++) {
            final IdleHandler idler = mPendingIdleHandlers[i];
            mPendingIdleHandlers[i] = null; // release the reference to the handler

            boolean keep = false;
            try {
                keep = idler.queueIdle();
            } catch (Throwable t) {
                Log.wtf("MessageQueue", "IdleHandler threw exception", t);
            }

            if (!keep) {
                synchronized (this) {
                    mIdleHandlers.remove(idler);
                }
            }
        }

        pendingIdleHandlerCount = 0;
        nextPollTimeoutMillis = 0;
    }
}
```

注释已经很详细了，现在知道了哪里把消息取出来，但是还不知道消息是哪里处理的，接着上面的`loop()`方法的代码往下看：

```
msg.target.dispatchMessage(msg);
```

这一行很关键，字面意思都可以看出来这里是分发消息，找到源码查看一下，之前说过msg.target就是与Message绑定的Handler，所以在Handler的源码里面找：

```java
/**
     * Handle system messages here.
     */
    public void dispatchMessage(Message msg) {
        if (msg.callback != null) {
            handleCallback(msg);
        } else {
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            handleMessage(msg);
        }
    }
```

代码很简单，但是答案就快揭晓了，Handler通过post和sendMessage之类的方法把消息发出去，绕了一大圈又回到了Handler，先别激动，看看代码到底说了什么：

```
if (msg.callback != null) {
            handleCallback(msg);
```

这里的`msg.callback`其实就是一个`Runnable`对象，可以通过查看Message源码发现：

```java
/**
     * Same as {@link #obtain(Handler)}, but assigns a callback Runnable on
     * the Message that is returned.
     * @param h  Handler to assign to the returned Message object's <em>target</em> member.
     * @param callback Runnable that will execute when the message is handled.
     * @return A Message object from the global pool.
     */
    public static Message obtain(Handler h, Runnable callback) {
        Message m = obtain();
        m.target = h;
        //创建Message类时候系统建议使用Message msg=Message.obtain()；形式
        m.callback = callback;
        return m;
    }
```

你想到了什么，在回想一遍我们使用Handler的两种方式，一种是`post(Runnable r)`的形式，一种是`sendMessage(Message msg)`形式，第一种方式刚好传递的就是一个`Runnable`对象，看一下这个`handleCallback(msg)`方法：

```
private static void handleCallback(Message message) {
        message.callback.run();
}
```

简单粗暴，走的就是`post(Runnable r)` 所传递参数的 `run()`方法，那么第二种形式呢：

```java
if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
```

这个`mCallback`是`Hanlder.Callback`类对象，这个`Callback`是`Handler`的内部的一个接口：

```java
public interface Callback {
        public boolean handleMessage(Message msg);
}
```

这就对应了`sendMessage(Message msg)`的形式，到此，真相大白，这里还有一点要注意的是，ActivityThread也就是UI线程是自动调用了`Looper.loop()`方法的所以在主线程使用Handler是不需要再去调用了，但是在子线程中却是要自己调用的，否则不会进入MessageQueue，而且Handler不仅仅只有更新UI的作用，它是与所创建的线程所绑定的，所以可以使用它在主线程向子线程发送消息，反过来也一样，关于这点的使用详见 [**Android主线程.子线程通信（Thread+handler）**](http://gqdy365.iteye.com/blog/2109453)

#### 总结

从最开始的使用到从源码的角度去分析，写这篇博客花了很长的时间，最后做个总结： 在整个Android内部通信进程中，Handler机制如果捋顺了相互之间的关系的话其实不难理解，下面上一张图帮助理解： ![Handler机制](http://oasusatoz.bkt.clouddn.com/handler.png)

套用一段很形象的话解释这幅图：

> 我们可以把传送带上的货物看做是一个个的Message，而承载这些货物的传送带就是装载Message的消息队列MessageQueue。传送带是靠发送机滚轮带动起来转动的，我们可以把发送机滚轮看做是Looper，而发动机的转动是需要电源的，我们可以把电源看做是线程Thread，所有的消息循环的一切操作都是基于某个线程的。一切准备就绪，我们只需要按下电源开关发动机就会转动起来，这个开关就是Looper的loop方法，当我们按下开关的时候，我们就相当于执行了Looper的loop方法，此时Looper就会驱动着消息队列循环起来。

> 那Hanlder在传送带模型中相当于什么呢？我们可以将Handler看做是放入货物以及取走货物的管道：货物从一端顺着管道划入传送带，货物又从另一端顺着管道划出传送带。我们在传送带的一端放入货物的操作就相当于我们调用了Handler的sendMessageXXX、sendEmptyMessageXXX或postXXX方法，这就把Message对象放入到了消息队列MessageQueue中了。当货物从传送带的另一端顺着管道划出时，我们就相当于调用了Hanlder的dispatchMessage方法，在该方法中我们完成对Message的处理。

[**这段话出自**](http://blog.csdn.net/iispring/article/details/47180325)

#### 参考博客

- [深入源码解析Android中的Handler,Message,MessageQueue,Looper](http://blog.csdn.net/iispring/article/details/47180325)
- [从Handler+Message+Looper源码带你分析Android系统的消息处理机制](http://blog.csdn.net/feiduclear_up/article/details/46817283)
- [Android异步消息处理机制完全解析，带你从源码的角度彻底理解](http://blog.csdn.net/guolin_blog/article/details/9991569)
- [聊一聊Android的消息机制](https://my.oschina.net/youranhongcha/blog/492591?_t=t)
