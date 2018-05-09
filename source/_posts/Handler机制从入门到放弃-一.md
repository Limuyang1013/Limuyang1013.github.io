---
title: Handler机制从入门到放弃(一)
date: 2016-10-29T22:41:23.000Z
categories: Android
tags:
  - Android
  - Handler
---

闲来无事，准备好好梳理一下Handler机制，之前分析过没有写成博客，结果就是慢慢的淡忘了，这次趁着刚分析完，赶紧写下来。

在开始分析之前先打打基础，理解理解什么是线程以及什么是Handler，这里大部分内容引用一篇来自伯乐在线的文章，因为看来看去关于基础的部分这个人已经说得很好了，我就负责把主要的部分抽取出来。

原文地址：**[Android线程和Handler基础入门](http://blog.jobbole.com/73267/)** <!-- more -->

现在大多数的移动设备已经变得越来越快，但是它们其实也不算是非常快。如果你想让你的APP既可以承受一些繁杂的工作而又不影响用户体验的话，那么必须把任务并行执行。在Android上，我们使用线程。

#### 什么是线程？

线程或者线程执行本质上就是一串命令（也是程序代码），然后我们把它发送给操作系统执行。

![线程](http://oasusatoz.bkt.clouddn.com/16-10-28/18731384.jpg)

一般来说，我们的CPU在任何时候一个核只能处理一个线程。多核处理器（目前大多数Android设备已经都是多核）顾名思义，就是可以同时处理多线程（通俗地讲就是可以同时处理多件事）。

#### 多核处理与单核多任务处理的实质

上面我说的是一般情况，并不是所有的描述都是一定正确的。因为单核也可以用多任务模拟出多线程。

每个运行在线程中的任务都可以分解成多条指令，而且这些指令不用同时执行。所以，单核设备可以首先切换到线程1去执行指令1A，然后切换到线程2去执行指令2A，接着返回到线程1再去执行1B、1C、1D，然后继续切换到线程2，执行2B、2C等等，以此类推。

这个线程之间的切换十分迅速，以至于在单核的设备中也会发生。几乎所有的线程都在相同的时间内进行任务处理。其实，这都是因为速度太快造成的假象，就像电影《黑客帝国》里的特工Brown一样，可以变幻出很多的头和手。

#### Java核心里的线程

在Java中，如果要想做平行任务处理的话，会在Runnable里面执行你的代码。可以继承Thread类，或者实现Runnable接口：

```java
// 1
public class IAmAThread extends Thread {
    public IAmAThread() {
        super("IAmAThread");
    }

    @Override
    public void run() {

// your code (sequence of instructions)
    }
}
// to execute this sequence of instructions in a separate thread.
new IAmAThread().start();

// 2
public class IAmARunnable implements Runnable {
    @Override
    public void run() {

// your code (sequence of instructions)
    }
}
// to execute this sequence of instructions in a separate thread.
IAmARunnable myRunnable = new IAmARunnable();
new Thread(myRunnable).start();
```

这两个方法基本上是一样的。第一个版本是创建一个Thread类，第二个版本是需要创建一个Runnable对象，然后也需要一个Thread类来调用它。

#### Android上的线程

无论何时启动APP，所有的组件都会运行在一个单独的线程中（默认的）----叫做主线程。这个线程主要用于处理UI的操作并为视图组件和小部件分发事件等，因此主线程也被称作UI线程(Main Thread)。除了Main Thread之外的线程都可称为Worker Thread。Main Thread主要负责控制UI页面的显示、更新、交互等。 因此所有在UI线程中的操作要求越短越好，只有这样用户才会觉得操作比较流畅。一个比较好的做法是把一些比较耗时的操作，例如网络请求、数据库操作、 复杂计算等逻辑都封装到单独的线程，这样就可以避免阻塞主线程，这样就需要用到了Android的Handler机制。

**这里划重点：Handler负责与子线程进行通讯，从而让子线程与主线程之间建立起协作的桥梁，使Android的UI更新的问题得到完美的解决**

#### 怎么创建Handler

既然Handler有这样的好处，那么看Handler怎么用，官方给出了两种方式创建一个Handler：

1、使用默认的构造方法：new Handler()。 2、使用带参的构造方法，参数是一个Runnable对象或者回调对象。

```java
//第一种方法
private Handler handler = new Handler();  
   private Runnable myRunnable= new Runnable() {    
        public void run() {  
             //一些耗时操作
        }  
    };
 //其他地方调用
 handler.post(xxx);
 这里就写一个post方法，实际上还有很多，诸如postDelayed、postAtTime
//第二种方法
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

#### 如何使用Handler

这里使用一个简单的Demo来演示Handler的用法，界面偏简单就不贴了，直接贴代码，模拟的是点击Button执行下载，下载完成后更新UI。

```java
public class MainActivity extends Activity implements Button.OnClickListener {

    private TextView statusTextView = null;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        statusTextView = (TextView)findViewById(R.id.statusTextView);
        Button btnDownload = (Button)findViewById(R.id.btnDownload);
        btnDownload.setOnClickListener(this);
    }

    @Override
    public void onClick(View v) {
        DownloadThread downloadThread = new DownloadThread();
        downloadThread.start();
    }

    class DownloadThread extends Thread{
        @Override
        public void run() {
            try{
                System.out.println("开始下载文件");
                //此处让线程DownloadThread休眠5秒中，模拟文件的耗时过程
                Thread.sleep(5000);
                System.out.println("文件下载完成");
                //文件下载完成后更新UI
                MainActivity.this.statusTextView.setText("文件下载完成");
            }catch (InterruptedException e){
                e.printStackTrace();
            }
        }
    }
}
```

按照以前写Java的思路的话可能会这么写，但是运行程序时候会发现控制台报错：

```xml
android.view.ViewRootImpl$CalledFromWrongThreadException: Only the original thread that created a view hierarchy can touch its views.
```

错误的意思是只有创建View的原始线程才能更新View。出现这样错误的原因是Android中的View不是线程安全的，下面给出合理的解释：

> 因为UI访问是没有加锁的，在多个线程中访问UI是不安全的，如果有多个子线程都去更新UI，会导致界面不断改变而混乱不堪。所以最好的解决办法就是只有一个线程有更新UI的权限，所以这个时候就只能有一个线程振臂高呼：放开那女孩，让我来！那么最合适的人选只能是主线程。

> 来自---[**Android中线程那些事**](http://blog.csdn.net/lfdfhl/article/details/51279160)

那么为了规避Android的这种机制，我们这里分别采用Handler的两种方式来实现上面的代码：

```java
A、使用post方式

public class MainActivity extends Activity implements Button.OnClickListener {

    private TextView statusTextView = null;

    //uiHandler在主线程中创建，所以自动绑定主线程
    private Handler uiHandler = new Handler();

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        statusTextView = (TextView)findViewById(R.id.statusTextView);
        Button btnDownload = (Button)findViewById(R.id.btnDownload);
        btnDownload.setOnClickListener(this);
    }

    @Override
    public void onClick(View v) {
        DownloadThread downloadThread = new DownloadThread();
        downloadThread.start();
    }

    class DownloadThread extends Thread{
        @Override
        public void run() {
            try{
                System.out.println("开始下载文件");
                //此处让线程DownloadThread休眠5秒中，模拟文件的耗时过程
                Thread.sleep(5000);
                System.out.println("文件下载完成");
                //文件下载完成后更新UI
                Runnable runnable = new Runnable() {
                    @Override
                    public void run() {
                        MainActivity.this.statusTextView.setText("文件下载完成");
                    }
                };
                uiHandler.post(runnable);
            }catch (InterruptedException e){
                e.printStackTrace();
            }
        }
    }
}


B、使用sendMessage方式实现

public class MainActivity extends Activity implements Button.OnClickListener {

    private TextView statusTextView = null;

    //uiHandler在主线程中创建，所以自动绑定主线程
    private Handler uiHandler = new Handler(){
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what){
                case 1:
                    System.out.println("msg.arg1:" + msg.arg1);
                    System.out.println("msg.arg2:" + msg.arg2);
                    MainActivity.this.statusTextView.setText("文件下载完成");
                    break;
            }
        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        statusTextView = (TextView)findViewById(R.id.statusTextView);
        Button btnDownload = (Button)findViewById(R.id.btnDownload);
        btnDownload.setOnClickListener(this);
        System.out.println("Main thread id " + Thread.currentThread().getId());
    }

    @Override
    public void onClick(View v) {
        DownloadThread downloadThread = new DownloadThread();
        downloadThread.start();
    }

    class DownloadThread extends Thread{
        @Override
        public void run() {
            try{
                System.out.println("开始下载文件");
                //此处让线程DownloadThread休眠5秒中，模拟文件的耗时过程
                Thread.sleep(5000);
                System.out.println("文件下载完成");
                //文件下载完成后更新UI
                Message msg = new Message();
                //虽然Message的构造函数式public的，我们也可以通过以下两种方式通过循环对象获取Message
                //msg = Message.obtain(uiHandler);
                //msg = uiHandler.obtainMessage();

                //what是我们自定义的一个Message的识别码，以便于在Handler的handleMessage方法中根据what识别
                //出不同的Message，以便我们做出不同的处理操作
                msg.what = 1;

                //我们可以通过arg1和arg2给Message传入简单的数据
                msg.arg1 = 123;
                msg.arg2 = 321;
                //我们也可以通过给obj赋值Object类型传递向Message传入任意数据
                //msg.obj = null;
                //我们还可以通过setData方法和getData方法向Message中写入和读取Bundle类型的数据
                //msg.setData(null);
                //Bundle data = msg.getData();
                //将该Message发送给对应的Handler
                uiHandler.sendMessage(msg);
            }catch (InterruptedException e){
                e.printStackTrace();
            }
        }
    }
}
```

以上代码来自博客：[**Android中Handler的使用**](http://blog.csdn.net/iispring/article/details/47115879)

上面这两种形式都能达到我们的要求，在此不一一测验，注释写的很详细了，看到这里应该已经大致知道了如何使用Handler，但是我想我们应该远远不满足于此，下一篇博客将带着大家从源码一起看看Handler机制到底是怎么实现的。
