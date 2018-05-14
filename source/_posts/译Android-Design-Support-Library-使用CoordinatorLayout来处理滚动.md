---
title: 译Android Design Support Library--使用CoordinatorLayout来处理滚动
date: 2016-05-03 20:32:58
categories: Android
tags: [Material Design,CoordinatorLayout]
---
#### 引言
本来这一次想写关于`SnackBar`的，但是因为官方都推荐使用Material Design控件最好使用`CoordinatorLayout` 来作为它们的父布局，所以就先讲解一下`CoordinatorLayout` 的知识，本来想自己去理解的，但是发现网上已经有一份很好的材料了就给搬过来了，原文是CodePath的，我给翻译了一遍，如果有出入的话欢迎指正---[英文原文地址](https://guides.codepath.com/android/Handling-Scrolls-with-CoordinatorLayout)

<!-- more -->
**概述**
[CoordinatorLayout](https://developer.android.com/intl/zh-cn/reference/android/support/design/widget/CoordinatorLayout.html) 可以实现在Google Material Design中提到的[滚动特效](http://www.google.com/design/spec/patterns/scrolling-techniques.html) ，目前，这个框架提供了好几种让你不用去写自定义动画效果代码就能实现的特效，这些特效包括如下几个方面：
 - 可以自动的让浮动按钮上下滑动，来为SnackBar预留出一定的空间

 ![这里写图片描述](http://oasusatoz.bkt.clouddn.com/donghua1.gif)
 
 - 扩大或者缩小ToolBar或者头部来为主要内容布局预留出一定的空间
 
 ![这里写图片描述](http://oasusatoz.bkt.clouddn.com/X5AIH0P.gif)

 - 控制哪一个View需要展开或者折叠以及展开或者折叠的速率，包括视察滚动的动画效果
 
 ![这里写图片描述](http://oasusatoz.bkt.clouddn.com/1JHP0cP.gif)

**代码示例**

来自Google的Chris Banes已经把CoordinatorLayout和其他[Design Support Library](https://guides.codepath.com/android/Design-Support-Library) 控件结合在一起写了一个Demo。

![这里写图片描述](http://i.imgur.com/aA8aGSg.png)

你可以在[GitHub](https://github.com/chrisbanes/cheesesquare)上找到这个Demo的源码，通过这个源码你可以很好的理解CoordinatorLayout的相关知识。

**配置**
首先你得保证你遵循了[Design Support Library](https://guides.codepath.com/android/Design-Support-Library) 规范。

**浮动操作按钮和SnackBar**

CoordinatorLayout可以结合`layout_anchor` 和`layout_gravity` 属性来做出浮动特效，想了解更多的话可以查看[浮动操作按钮指南](https://guides.codepath.com/android/Floating-Action-Buttons) 。

当显示一个[SnackBar](https://guides.codepath.com/android/Displaying-the-Snackbar)的时候，它通常出现在我们屏幕的底部，为了预留出足够的空间，我们的浮动操作按钮不得不向上移动一段距离：

![这里写图片描述](http://imgur.com/zF9GGsK.gif)

只要你把CoordinatorLayout作为你的根布局，这个动画效果会自动产生，浮动操作按钮有一个[预设的行为属性](https://developer.android.com/intl/zh-cn/reference/android/support/design/widget/FloatingActionButton.Behavior.html) ，那就是自动检测SnackBar是否被添加到屏幕上，如果是则浮动操作按钮会产生一个向上移动一个等于SnackBar高度的距离的动画效果。

```html
 <android.support.design.widget.CoordinatorLayout
        android:id="@+id/main_content"
        xmlns:android="http://schemas.android.com/apk/res/android"
        xmlns:app="http://schemas.android.com/apk/res-auto"
        android:layout_width="match_parent"
        android:layout_height="match_parent">

   <android.support.v7.widget.RecyclerView
         android:id="@+id/rvToDoList"
         android:layout_width="match_parent"
         android:layout_height="match_parent"></android.support.v7.widget.RecyclerView>

   <android.support.design.widget.FloatingActionButton
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_gravity="bottom|right"
        android:layout_margin="16dp"
        android:src="@mipmap/ic_launcher"
        app:layout_anchor="@id/rvToDoList"
        app:layout_anchorGravity="bottom|right|end"/>
 </android.support.design.widget.CoordinatorLayout>
```
**展开或折叠ToolBar**

![这里写图片描述](http://imgur.com/X5AIH0P.gif)

首先要注意的一点是你不是使用已经过时的ActionBar，确保遵守了[用ToolBar代替ActionBar](https://guides.codepath.com/android/Using-the-App-ToolBar#using-toolbar-as-actionbar) 的指南，同样，你也得确保使用CoordinatorLayout作为主布局容器。

```java
<android.support.design.widget.CoordinatorLayout xmlns:android="http://schemas.android.com/apk/res/android"
 xmlns:app="http://schemas.android.com/apk/res-auto"
    android:id="@+id/main_content"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:fitsSystemWindows="true">

      <android.support.v7.widget.Toolbar
                android:id="@+id/toolbar"
                android:layout_width="match_parent"
                android:layout_height="?attr/actionBarSize"
                app:popupTheme="@style/ThemeOverlay.AppCompat.Light" />

</android.support.design.widget.CoordinatorLayout>
```
**对滚动事件作出响应**

接下来，我们需要使用一个叫[AppBarLayout](http://developer.android.com/intl/zh-cn/reference/android/support/design/widget/AppBarLayout.html)的容器布局来为ToolBar添加对滚动事件的响应：

```html
<android.support.design.widget.AppBarLayout
        android:id="@+id/appbar"
        android:layout_width="match_parent"
        android:layout_height="@dimen/detail_backdrop_height"
        android:theme="@style/ThemeOverlay.AppCompat.Dark.ActionBar"
        android:fitsSystemWindows="true">

  <android.support.v7.widget.Toolbar
                android:id="@+id/toolbar"
                android:layout_width="match_parent"
                android:layout_height="?attr/actionBarSize"
                app:popupTheme="@style/ThemeOverlay.AppCompat.Light" />

 </android.support.design.widget.AppBarLayout>
```
**注意**：根据[Google官方文档](http://developer.android.com/intl/zh-cn/reference/android/support/design/widget/AppBarLayout.html) ，AppBarLayout目前被指定为第一个嵌套在CoordinatorLayout里面的子布局。

接下来，我们需要定义出AppBarLayout和滚动视图之间的联系，给RecyclerView 或者任意其他一个可以实现嵌套滚动的View比如说[NestedScrollView](http://stackoverflow.com/questions/25136481/what-are-the-new-nested-scrolling-apis-for-android-l) 添加一个`app:layout_behavior` 属性，support library包含了一个特殊的同[AppBarLayout.ScrollingViewBehavior](https://developer.android.com/intl/zh-cn/reference/android/support/design/widget/AppBarLayout.ScrollingViewBehavior.html) 一一对应的字符串资源文件`@string/appbar_scrolling_view_behavior` ，用来通知`AppBarLayout` 这个特殊的View何时发生了滚动事件，这个behavior 需要建立在触发了这个滚动事件的View上。

```html
 <android.support.v7.widget.RecyclerView
        android:id="@+id/rvToDoList"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        app:layout_behavior="@string/appbar_scrolling_view_behavior">
```
当一个CoordinatorLayout 发现在RecyclerView中定义了这个属性之后，它会在自己所包含的组件中逐个搜索看是否有与这个behavior相关联的View，在此个别情况中，`AppBarLayout.ScrollingViewBehavior` 描述了RecyclerView和AppBarLayout中的一种依赖关系，RecyclerView 的任何滚动事件都将会触发AppBarLayout布局或它包含的子View 的改变。
要想让RecyclerView 的滚动事件触发AppBarLayout内部声明的View的改变只需要用到`app:layout_scrollFlags` 这个属性：

```html
<android.support.design.widget.AppBarLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:fitsSystemWindows="true"
        android:theme="@style/ThemeOverlay.AppCompat.Dark.ActionBar">

            <android.support.v7.widget.Toolbar
                android:id="@+id/toolbar"
                android:layout_width="match_parent"
                android:layout_height="?attr/actionBarSize"
                app:layout_scrollFlags="scroll|enterAlways"/>

 </android.support.design.widget.AppBarLayout>
```

在`app:layout_scrollFlags` 属性中我们必须要设定`scroll` 这个滑动标志来使任何滑动事件生效，这个标志可以配合`enterAlways`、`enterAlwaysCollapsed`、`exitUntilCollapsed`、或者 `snap` 这几种标志来一起使用。

 - enterAlways：当向上滑动的时候View就会变为可见，这个标志在你从一个列表的底部向上滚动并且想要立刻显示ToolBar的时候会很有用：
 
 ![这里写图片描述](http://imgur.com/sGltNwr.png)

- 一般情况下，ToolBar只有当你滚动到列表顶部的时候才会显示：

 ![这里写图片描述](http://i.imgur.com/IZzcL1C.png)

- enterAlwaysCollapsed：正常情况下，只有当你用了enterAlways这个标志位，你的ToolBar才会随着你的向下滚动继续扩展：

 ![这里写图片描述](http://imgur.com/nVtheyw.png)

- 假定你已经声明了enterAlways 标志位，并且你已经制定了一个最小高度minHeight，你还可以指定enterAlwaysCollapsed，这样的话你的View将在达到这个最小高度minHeight时候开始显示，并且随着你的滚动你的View会慢慢的展开直到你滑动到了View的顶部：

 ![这里写图片描述](http://imgur.com/HqR8Nx5.png)

- exitUntilCollapsed：当你已经设置了scroll 标志位，向下滚动会导致整个内容视图产生滚动：
 ![这里写图片描述](http://imgur.com/qpEr4x5.png)

- 通过指定最小高度minHeight和exitUntilCollapsed标志，ToolBar会隐藏到minHeight的高度：

 ![这里写图片描述](http://imgur.com/dTDPztp.png)

 - 注意：如果滚动结束View视图尺寸减小少于最开始部分的50%，那么这个View会回到原始大小，但是如果大于原始尺寸的50%的话，那么这个View会完全消失：
 ![这里写图片描述](http://i.imgur.com/9hnupWJ.png)
 - 记住你所有的View都需要把scroll 标志放在第一位，这样需要折叠的View会先行退出然而固定的元素会留在顶部，此时你应该已经注意到了我们的ToolBar响应滚动事件。

 ![这里写图片描述](http://imgur.com/Hl2Asb1.gif)

**制造出折叠效果**

如果我们想做出ToolBar的折叠效果，我们必须使用CollapsingToolbarLayout布局来包裹ToolBar：

```html
<android.support.design.widget.CollapsingToolbarLayout
            android:id="@+id/collapsing_toolbar"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:fitsSystemWindows="true"
            app:contentScrim="?attr/colorPrimary"
            app:expandedTitleMarginEnd="64dp"
            app:expandedTitleMarginStart="48dp"
            app:layout_scrollFlags="scroll|exitUntilCollapsed">
            
            <android.support.v7.widget.Toolbar
                android:id="@+id/toolbar"
                android:layout_width="match_parent"
                android:layout_height="?attr/actionBarSize"
                app:layout_scrollFlags="scroll|enterAlways"></android.support.v7.widget.Toolbar>

</android.support.design.widget.CollapsingToolbarLayout>
```
结果显示如下：

![这里写图片描述](http://imgur.com/X5AIH0P.gif)

通常我们都是设置ToolBar的Title，但是现在我们需要把Title设置在CollapsingToolBarLayout 上：

```java
 CollapsingToolbarLayout collapsingToolbar =
              (CollapsingToolbarLayout) findViewById(R.id.collapsing_toolbar);
 collapsingToolbar.setTitle("Title");
```
注意当我们使用CollapsingToolbarLayout的时候，我们的状态栏需要设置为半透明(API 19)或者透明(API 21),就好像[这个文件](https://github.com/chrisbanes/cheesesquare/blob/master/app/src/main/res/values-v21/styles.xml)展示的，特别的，我们还需要在`res/values-xx/styles.xml` 设置如下的style：

```html
<!-- res/values-v19/styles.xml -->
<style name="AppTheme" parent="Base.AppTheme">
    <item name="android:windowTranslucentStatus">true</item>
</style>

<!-- res/values-v21/styles.xml -->
<style name="AppTheme" parent="Base.AppTheme">
    <item name="android:windowDrawsSystemBarBackgrounds">true</item>
    <item name="android:statusBarColor">@android:color/transparent</item>
</style>
```
如果你照着上面来设置，你的布局有一部分会隐藏在system bar后面，这个时候你需要设置`android:fitsSystemWindow` 属性，[详情查看](http://blog.raffaeu.com/archive/2015/04/11/android-and-the-transparent-status-bar.aspx)

**制造视差滚动动画效果**

CollapsingToolbarLayout 布局还允许让我们做出更多更高级的动画效果，譬如在它内部加入一个ImageView，当它折叠的时候这个ImageView会产生一个淡出的效果，当用户滚动的时候title的高度也能随之改变：

![这里写图片描述](http://imgur.com/ah4l5oj.gif)

为了生成这种效果，我们加入了一个ImageView并且声明了`app:layout_collapseMode="parallax"` 属性：

```html
<android.support.design.widget.CollapsingToolbarLayout
    android:id="@+id/collapsing_toolbar"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:fitsSystemWindows="true"
    app:contentScrim="?attr/colorPrimary"
    app:expandedTitleMarginEnd="64dp"
    app:expandedTitleMarginStart="48dp"
    app:layout_scrollFlags="scroll|exitUntilCollapsed">

            <android.support.v7.widget.Toolbar
                android:id="@+id/toolbar"
                android:layout_width="match_parent"
                android:layout_height="?attr/actionBarSize"
                app:layout_scrollFlags="scroll|enterAlways"></android.support.v7.widget.Toolbar>
            <ImageView
                android:src="@drawable/cheese_1"
                app:layout_scrollFlags="scroll|enterAlways|enterAlwaysCollapsed"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:scaleType="centerCrop"
                app:layout_collapseMode="parallax"
                android:minHeight="100dp"/>

</android.support.design.widget.CollapsingToolbarLayout>
```
**自定义Behaviors**

一个关于自定义behavior的例子在我们讨论[CoordinatorLayou和浮动操作按钮](https://guides.codepath.com/android/Floating-Action-Buttons#using-coordinatorlayout)的时候，CoordinatorLayout是通过搜索任意一个包含了 [CoordinatorLayout Behavior](http://developer.android.com/intl/zh-cn/reference/android/support/design/widget/CoordinatorLayout.Behavior.html) 的子View来工作的，无论是通过在XML文件中使用`app:layout_behavior` 属性来定义还是以编码的方式对View类使用`@DefaultBehavior` 注释，当滚动事件发生的时候，CoordinatorLayout 将会尝试去触发那些声明了依赖的子View。

想要定义你自己的CoordinatorLayout Behavior，你需要实现`layoutDependsOn()`和`onDependentViewChanged()` 这两个方法，比如，AppBarLayout.Behavior就定义了这两个关键的方法，这个behavior 用来触发AppBarLayout 的变化当滚动事件发生的时候：

```java
public boolean layoutDependsOn(CoordinatorLayout parent, View child, View dependency) {
          return dependency instanceof AppBarLayout;
      }

 public boolean onDependentViewChanged(CoordinatorLayout parent, View child, View dependency) {
          // check the behavior triggered
          android.support.design.widget.CoordinatorLayout.Behavior behavior = ((android.support.design.widget.CoordinatorLayout.LayoutParams)dependency.getLayoutParams()).getBehavior();
          if(behavior instanceof AppBarLayout.Behavior) {
          // do stuff here
          }
 }       
```
理解如何实现这些自定义behavior的最好途径是研究AppBarLayout.Behavior 和 FloatingActionButtion.Behavior。虽然这些源代码还没有放出来，但是你可以使用Android Studio 1.2集成的反编译器来查看。