---
title: AsyncTask初步解析
date: 2015-12-18 20:26:43
categories: Android
tags: [异步IO,AsyncTask]
---
#### AsyncTask  -- 直接继承与Object类 在API-3中开始就被定义

 **一、AsyncTask初步介绍**
　在Android程序开始运行的时候会单独启动一个进程，默认情况下所有这个程序操作都在这个进程中进行。一个Android程序默认情况下只有一个进程，但是一个进程却是可以有许线程的。
　在这些线程中，有一个线程叫做UI线程，也叫做Main Thread，除了Main Thread之外的线程都可称为Worker Thread。Main Thread主要负责控制UI页面的显示、更新、交互等。 因此所有在UI线程中的操作要求越短越好，只有这样用户才会觉得操作比较流畅。一个比较好的做法是把一些比较耗时的操作，例如网络请求、数据库操作、 复杂计算等逻辑都封装到单独的线程，这样就可以避免阻塞主线程，这个时候就用到了异步任务类AsyncTask。
　AsyncTask 是一个综合Thread 和 Handler的辅助类，并不是通用线程框架的一部分。AsyncTask 是短期后台操作的理想选择（最多几秒钟），如果你需要线程长时间的保持运行，强烈建议你使用 java.util.concurrent 包提供的各种API以满足你的需求，比如：Executor, ThreadPoolExecutor and FutureTask.这里我们只讨论AsyncTask的初步使用。
<!-- more -->　　
**二、AsyncTask的内部机制**
　首先我们来看一下AsyncTask的方法定义：

```java
public abstract class AsyncTask<Params, Progress, Result> {
```

　我们可以很明显的看到这里有三个泛型参数，当我们要利用AsyncTask类的时候必须传入如下三个参数：
 1. Parms：启动该异步任务的时候传入的参数类型
 2. Progress：当异步任务在后台执行时返回的进度单位，也可以简单的理解为后台任务执行的进度
 3. Result：后台任务执行结束时返回的结果类型
Android Developer还告知我们：在某些情况有些泛型不需要被指定，这时我们为了简便可以传入一个空类型的参数：

```java
private class MyTask extends AsyncTask<Void, Void, Void> { ... }
```

　接下来，为了执行一个异步任务我们还需要执行如下几个步骤：
 - onPreExecute()：在UI线程中被执行，通常是用来做任务执行前的准备工作，例如加载一个进度条或者在UI界面上显示一个文本等
 - doInBackground(Params...)：在onPreExecute()方法完成之后立即被后台进程调用，用于执行一些耗时操作，启动异步任务时执行的Parms参数也会被传递到该方法，在这一方法中我们可以使用publishProgress(Progress...)方法来更新进度信息，并且在该方法内不可以更新UI信息
 - onProgressUpdate(Progress...)：在UI线程中执行，如果我们在doInBackground(Params...)方法中使用了publishProgress(Progress...)那么进度信息将可以以进度条或者Log的形式显示在UI组件上
 - onPostExecute(Result)：在UI线程中执行，当后台任务操作结束之后该方法会被执行，doInBackground(Params...)方法计算得到的结果将作为该方法的输入参数来进行一些UI更新的操作
 
#### 取消AsyncTask任务操作
　我们可以在异步任务执行的任何时候通过调用cancel(boolean )方法来取消；我们在cancel(boolean)方法中传入true参数则表明取消正在执行的任务，如果调用成功，那么之后我们使用isCancelled()方法来判断任务是否被取消的时候都会返回true，取消之后在执行完doInBackground(Object[])后onCancelled(Object)方法会代替onPostExecute(Object)方法被执行。为了确保能够尽快的取消一个任务，我们应该在doInBackground(Object[])里面周期性的检查isCancelled()的返回值(例如在一个循环里面)
 **注意事项**
 - AsyncTask 类必须在UI主线程中被加载,从Android4.1开始系统上已经帮我们自动完成这一点
 - AsyncTask类的实例必须在UI线程中进行创建
 -  execute(Params...) 必须在UI主线程中被调用
 - 不要手动的去调用 onPreExecute(), onPostExecute(Result), doInBackground(Params...),  onProgressUpdate(Progress...)这些方法
 - 任务只能被执行一次(如果执行第二次则抛出一个异常)
 
**三、AsyncTask的使用**
 　下面给出一个关于AsyncTask的使用的Demo，布局只有两个Button、一个TextView和一个ProgressBar，代码如下：
```html
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent" >

    <ProgressBar
        android:id="@+id/progressBar"
        android:layout_width="match_parent"
        android:layout_height="25dp"
        android:layout_centerInParent="true"
        android:layout_margin="5dip"
        android:layout_marginTop="60dp"
        android:background="@drawable/btn_style"
        android:indeterminate="false"
        android:indeterminateOnly="false"
        android:max="100"
        android:progress="0"
        android:progressDrawable="@drawable/bakground_progress" />

    <Button
        android:id="@+id/Cancle"
        android:layout_width="100dp"
        android:layout_height="wrap_content"
        android:layout_alignRight="@+id/progressBar"
        android:layout_below="@+id/progressBar"
        android:layout_marginRight="24dp"
        android:background="@drawable/btn_style"
        android:text="Cancle"
        android:textColor="#ffffff"
        android:textStyle="bold" />

    <Button
        android:id="@+id/Execute"
        android:layout_width="100dp"
        android:layout_height="wrap_content"
        android:layout_alignLeft="@+id/progressBar"
        android:layout_below="@+id/progressBar"
        android:layout_marginLeft="42dp"
        android:background="@drawable/btn_style"
        android:text="Excuse"
        android:textColor="#ffffff"
        android:textStyle="bold" />

    <TextView
        android:id="@+id/tv"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_above="@+id/progressBar"
        android:layout_centerHorizontal="true"
        android:layout_marginBottom="31dp"
        android:text="AsyncTask测试"
        android:textSize="20sp"
        android:textColor="#000000"
        android:textStyle="bold" />

</RelativeLayout>
```
　结构相对简单，接下里让我们看看TestAsyncTask.java的代码：
```java
package com.example.testasynctask;

import android.app.Activity;
import android.os.AsyncTask;
import android.os.Bundle;
import android.view.View;
import android.widget.Button;
import android.widget.ProgressBar;
import android.widget.TextView;
import android.widget.Toast;

public class TestAsyncTask extends Activity {

	private Button execute, cancle;
	private ProgressBar progressBar;
	private TextView tv;
	private TestAsyncTaskProgressBar testAsyncTaskProgressBar;

	@Override
	protected void onCreate(Bundle savedInstanceState) {
		super.onCreate(savedInstanceState);
		setContentView(R.layout.activity_main);
		execute = (Button) findViewById(R.id.Execute);
		tv = (TextView) findViewById(R.id.tv);
		progressBar = (ProgressBar) findViewById(R.id.progressBar);
		execute.setOnClickListener(new View.OnClickListener() {

			@Override
			public void onClick(View v) {
				/**
				 * 这里注意新建的任务只能执行一次，否则会出现异常
				 */
				testAsyncTaskProgressBar = new TestAsyncTaskProgressBar();
				testAsyncTaskProgressBar.execute();
				execute.setEnabled(false);
				cancle.setEnabled(true);
			}
		});
		cancle = (Button) findViewById(R.id.Cancle);
		cancle.setOnClickListener(new View.OnClickListener() {

			@Override
			public void onClick(View v) {
				/**
				 * 使用该方法可以尝试取消正在执行的任务
				 */
				testAsyncTaskProgressBar.cancel(true);

			}
		});
	}

	/**
	 * 要使用AsyncTask必须创建它的子类，并且该子类至少需要覆盖doInBackground(Params...)方法
	 */
	class TestAsyncTaskProgressBar extends AsyncTask<Void, Integer, Void> {
		/**
		 * 当onPreExecute()方法结束执行时该方法立即在后台执行，用于进行一些耗时的操作，并且在执行后台操作的同时
		 * 还可以通过publishProgress
		 * (Progress...)将执行进度实时传送到UI线程，利用onProgressUpdate(Progress...)方法
		 * 可以进行动态展示，而且在该方法内不允许修改UI
		 */
		@Override
		protected Void doInBackground(Void... params) {
			while (!isCancelled() && progressBar.getProgress() < 100) {
                publishProgress(new Integer(5));
                try {
                    Thread.sleep(500);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
			return null;
		}
		/**
		 * 该方法在UI线程中被执行，主要用在执行后台操作之前做一些UI操作譬如显示一个进度条
		 */
		@Override
		protected void onPreExecute() {
			super.onPreExecute();
			tv.setText("开始执行");
		}

		/**
		 * 后台任务执行完毕之后该方法立即执行，后台任务执行结果的参数会被传递到该方法中
		 */
		@Override
		protected void onPostExecute(Void result) {
			super.onPostExecute(result);
			tv.setText("执行完毕");
		}

		/**
		 * 当publishProgress(Progress...)方法被执行的时候被调用，并且接受传入第二个参数
		 */
		@Override
		protected void onProgressUpdate(Integer... values) {
			super.onProgressUpdate(values);
			progressBar.setProgress(progressBar.getProgress() + values[0]);
		}

		/**
		 * cancel(boolean)方法被调用并且doInBackground(Object[])方法执行完毕时执行
		 */
		@Override
		protected void onCancelled() {
			super.onCancelled();
			tv.setText("执行中断");
			progressBar.setProgress(0);
			execute.setEnabled(true);
			cancle.setEnabled(false);
		}
	}
}

```
总体执行效果如下：
![这里写图片描述](http://img.blog.csdn.net/20151218234420164)
[AsyncTask Demo源码](http://download.csdn.net/detail/wei_smile/9367489)
#### 参考链接：
 - http://blog.csdn.net/ahuier/article/details/16953793
 - http://blog.csdn.net/liuhe688/article/details/6532519