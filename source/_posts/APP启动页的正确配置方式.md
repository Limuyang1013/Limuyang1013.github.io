---
title: APP启动页的正确配置方式
date: 2016-03-01T20:23:31.000Z
categories: Android
tags:
  - 冷热启动
  - app
---

在APP的启动页面(Splash Screen)好多都是等待3秒，好一点的还可以跳过，但是有的跳过也是假的按钮。当然像一些大厂的APP，像网易新闻等启动页面都是广告，人家要收广告费的。但是，对于一些普通的APP，有的也出现等待三秒的启动画面，出现一个大大的logo,好像告诉用户他打开的是什么应用，加深用户的映像，这完全是浪费用户的时间，给用户很差的体验！其实我只想快点进入APP啊！！！而且有些APP启动时候都会出现一个短暂的空白界面，现在我们就来避免这些已知的问题。 

# <!-- more -->

你看到在这个APP的启动页面所花费的时间正是APP所初始化配置自己的时间，第一次启动也是这样的，所以第一次是最慢的，但是如果有缓存了，那么每次再打开应该是立即打开了吧 **实现一个启动页面(Splash Screen)** 实现一个启动页面可能和你想象的有点不一样。这个启动的页面必须是立即准备好的页面，即使是在Activity中加载一个xml页面也要是立刻加载好的。 所以，一般不会用layout来当启动页面。取而代之的是用一个颜色作为你的Activity的主题背景，接下来，在你的res/drawable文件夹下创建一个XML的drawable。

```java
<?xml version="1.0" encoding="utf-8"?>
<layer-list xmlns:android="http://schemas.android.com/apk/res/android">

    <item
        android:drawable="@color/gray"/>

    <item>
        <bitmap
            android:gravity="center"
            android:src="@mipmap/ic_launcher"/>
    </item>

</layer-list>
```

这里，我设置了背景颜色和一张居中的图片。 然后，在主题中，将这个设置为Activity的背景。打开你的styles.xml然后为你的Activity添加一个新的主题。

```java
  <!-- Base application theme. -->
    <style name="AppTheme" parent="Theme.AppCompat.Light.DarkActionBar">
        <!-- Customize your theme here. -->
    </style>

    <style name="SplashTheme" parent="Theme.AppCompat.NoActionBar">
        <item name="android:windowBackground">@drawable/background_splash</item>
    </style>

</resources>
```

在你新的SplashTheme中，设置窗口背景属性为我们之前写的XML的drawable，就是layer-list的xml。然后在你的AndroidManifest.xml中配置一下就好了。

```java
<activity
    android:name=".SplashActivity"
    android:theme="@style/SplashTheme">
    <intent-filter>
        <action android:name="android.intent.action.MAIN" />

        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>
</activity>
```

最后，在你的SplashActivity.class文件中，编码直接进入主页面就行了。

```java
public class SplashActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        Intent intent = new Intent(this, MainActivity.class);
        startActivity(intent);
        finish();
    }
}
```

这里发现你并没有为Activity设置layout视图，视图来自于主题！所以，这应该是最快的方法启动页面了（相比较加载layout视图）。如果你一定要通过加载layout来显示页面，可能你初始化完了才跳出页面，这时已经有点迟了，所以，你应该考虑用极短的时间来显示加载layout视图. **正确的打开它** 当你完成这些步骤，你就正确的完成了启动页面。 ![这里写图片描述](http://img.blog.csdn.net/20151210222140337)
