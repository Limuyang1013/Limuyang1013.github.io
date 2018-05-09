---
title: Android Design Support Library--简约而不简单的SnackBar
date: 2016-05-07 20:15:16
categories: Android
tags: [Material Design,Google,Android]
---
#### 引言
在之前我有提到这一篇Android Design Support Library系列文章是关于SnackBar的，但是由于要用到CoordinatorLayout所以先翻译了一篇相关文章，如果还不了解的可以先看一下**[Android Design Support Library--使用CoordinatorLayout来处理滚动](http://blog.csdn.net/wei_smile/article/details/51306688)** ，这一篇我们讲SnackBar，SnackBar其实就是Toast的升级版，他们之间最大的不同就是：SnackBar会对我们的操作提供一个轻量级的反馈，并且可以对点击事件做出响应，如果是在手机上使用一个SnackBar的话，我们会看到在屏幕底部出现一条简短的信息，如果是在更大的屏幕上这条信息应该会显示在左下角，并且当一个SnackBar显示的时候它是凌驾于当前所有屏幕元素之上的，我们在屏幕上一次只能显示一个SnackBar，如果这么讲不是很清楚的话，我们先来看一个小Demo，通过代码驱动理解是比较好的方式。
<!-- more -->

#### 示例

根据SnackBar的特点，在屏幕上显示出不同的SnackBar，效果如下：

![这里写图片描述](http://img.blog.csdn.net/20160507110336476)

先看一下相关的API文档：


|     方法类型    |                             方法                            |                                 作用                                |
|:---------------:|:-----------------------------------------------------------:|:-------------------------------------------------------------------:|
|       void      |                          dismiss()                          |                            使SnackBar消失                           |
|       int       |                        getDuration()                        |                        返回SnackBar的持续时间                       |
|       View      |                          getView()                          |                        返回当前SnackBar的View                       |
|     boolean     |                          isShown()                          |                      判断该SnackBar是否正在显示                     |
|     boolean     |                      isShownOrQueued()                      |           判断该SnackBar是否正在显示或者排队等待即将要显示          |
| static Snackbar |           make(View view, int resId, int duration)          |                    新建一个用来显示信息的SnackBar                   |
| static Snackbar |       make(View view, CharSequence text, int duration)      |                                 同上                                |
|     Snackbar    |     setAction(int resId, View.OnClickListener listener)     |                   设置这个即将显示的SnackBar的动作                  |
|     Snackbar    | setAction(CharSequence text, View.OnClickListener listener) |                                 同上                                |
|     Snackbar    |          setActionTextColor(ColorStateList colors)          |                     设置action的文字颜色(右边的)                    |
|     Snackbar    |                setActionTextColor(int color)                |                                 同上                                |
|     Snackbar    |           setCallback(Snackbar.Callback callback)           |            设置一个回调，当SnackBar的可见性改变的时候调用           |
|     Snackbar    |                  setDuration(int duration)                  |                      设置SnackBar信息的显示时间                     |
|     Snackbar    |                      setText(int resId)                     |                       更新SnackBar上显示的文字                      |
|     Snackbar    |                setText(CharSequence message)                |                                 同上                                |
|       void      |                            show()                           | 显示SnackBar，最后一定要调用这个方法，不然SnackBar不显示，联想Toast |


----------


可以看到Demo上显示了三种不同的SnackBar，我们都知道SnackBar是Toast的升级版，但也说明了一个问题那就是SnackBar是用来显示消息的，同时根据你的需求不同可以对这些消息做出一定的响应动作，下面分析三种显示消息方式的不同：

 - 普通的SnackBar

也许有的人并没有过多的需求，只是单纯地想把SnackBar当作一个显示消息的控件而已，那么可以很简单的在代码中这么使用：

```java
Snackbar.make(mCoor, R.string.normal, Snackbar.LENGTH_SHORT).show();
```
对比一下我们的Toast方式：

```java
Toast.makeText(MainActivity.this,R.string.normal,Toast.LENGTH_SHORT).show();
```

是不是很像，没错简单的使用的话SnackBar跟Toast并没有多大区别，但是动画效果上是有差异的，如果你注意到了这一点：

![这里写图片描述](http://img.blog.csdn.net/20160507111906669)

看，这个侧边滑动消失的效果只有当你使用CoordinatorLayout作为根布局才有，这就是为什么在写SnackBar之前我要先说明一下CoordinatorLayout的原因，如果你使用普通的LinearLayout或者RelativeLayout是不会有这种动画交互效果的，另外，**注意**
SnackBar的make方法有两种重载方法，分别是：

```java
make(View view, int resId, int duration)
```
和

```java
make(View view, CharSequence text, int duration)
```
这里有三个参数，第一个参数View表示的意思是我们传入一个View，然后SnackBar会遍历整个View Tree来找到一个合适的View承载SnackBar的View，如果你想要实现上面的动画交互效果的话最好是传入CoordinatorLayout对象，第二个参数的话是两个重载方法不同的地方，有一种是我们熟知的：

```java
Snackbar.make(mCoor, "普通的SnackBar", Snackbar.LENGTH_SHORT).show();
```
还有一种要求传入一个ID，注意这个ID并不是指其他的什么，就是你在string.xml文件中定义的字符串资源的ID，比如这样：

```java
Snackbar.make(mCoor, R.string.normal, Snackbar.LENGTH_SHORT).show();
```
然后第三个参数是SnackBar的持续时间，只有三种：

```
1、Snackbar.LENGTH_INDEFINITE 一直显示直到另一个SnackBar出现或者主动调用了dismiss()方法
2、Snackbar.LENGTH_SHORT 显示较短的时间
3、Snackbar.LENGTH_LONG  显示较长的时间
```
但是官方文档是这么描述的：

```
either be one of the predefined lengths: LENGTH_SHORT, LENGTH_LONG, or a custom duration in milliseconds.
```
说是可以自定义显示时间，但是我自己试了确实不可以，应该是API文档的一个小bug，如果谁试成功了赶紧告诉我~~
如果使用过Toast的话上面的应该很好理解，好了，如果你的业务中对SnackBar并没有更多的要求，那么最普通的SnackBar应该满足了，接下来看稍微高级一点的：

 - 带回调的SnackBar：

如果还不太清楚回调的话可以看看这个**[Android回调函数机制那点事](http://blog.csdn.net/wei_smile/article/details/51040034)** ，讲这个之前先提一点，如果我们想更加灵活的使用Snackbar的话最好是先持有它的引用，也就是：

```java
private Snackbar mSnackBar
```
原因很简单，你会发现上面提供的常用API中很多方法都是非静态方法并不是静态方法，你要调用的话只能通过SnackBar对象去调用。

然后说SnackBar回调之前先说一下Action，SnackBar提供了一个setAction方法：

```java
1、setAction(int resId, View.OnClickListener listener)
2、setAction(CharSequence text, View.OnClickListener listener)
```
同样是两个重载方法，第一个参数跟前面解释的一样，第二个参数是我们熟知的对点击事件的监听，使用方法如下：

```java
Snackbar.make(mCoor,R.string.callback,Snackbar.LENGTH_SHORT)
                        .setAction(R.string.UNDO, new View.OnClickListener() {
                            @Override
                            public void onClick(View v) {
        // do something
                            }
                        }).show();
```

看一下效果：

![这里写图片描述](http://img.blog.csdn.net/20160507114025959)

当我们调用了setAction方法并且传入一个字符串之后，SnackBar的右下角就会呈现出我们传入的字符串，并且这个字符串是可点击的，我们可以在点击事件里面做出响应，比如说跳转Activity或者弹出一个Toast等等，这里默认你点击了这个Action这个SnackBar是会消失的。也就是无论你的duration参数设置的是一直显示还是显示多长时间都会消失。

有些人可能对右下角这个文字的颜色不满足想要改变，没问题，你想到的Google都给你想好了，SnackBar专门提供了方法来更改Action的文字颜色:

```
1、setActionTextColor(ColorStateList colors)
2、setActionTextColor(int color)
```
这里第一种方式不建议用，太复杂，你要想这么用也行：

```java
Resources resource = (Resources) getBaseContext().getResources();
       ColorStateList csl = (ColorStateList) resource.getColorStateList(R.color.PeachPuff);
       mSnackBar.setActionTextColor(csl);
```
这是网上找到的一种方式，但是我还是推荐使用第二种方式来更改Action的文字颜色，可以看到是我们熟悉的传入一个int型的值，我提供如下几种方式更改：

```
1、mSnackBar.setActionTextColor(Color.rgb(232,44,123))
2、mSnackBar.setActionTextColor(Color.BLUE)
3、mSnackBar.setActionTextColor(Color.parseColor("#FFDAB9"));
```
对了，我还发现一种额外的方式，我们现在使用Android Studio创建新的Project时候系统都会默认在style.xml文件夹下面生成这个：

```java
<style name="AppTheme" parent="Theme.AppCompat.Light.DarkActionBar">
        <!-- Customize your theme here. -->
        <item name="colorPrimary">@color/colorPrimary</item>
        <item name="colorPrimaryDark">@color/colorPrimaryDark</item>
        <item name="colorAccent">@color/colorAccent</item>
    </style>
```
这里的

```
<item name="colorAccent">@color/colorAccent</item>
```
其实也可以更改Action文字颜色，而且默认的Action文字颜色就是这里设置的颜色，但是有一个缺点就是如果你改动了这里，那么很多Material Design控件的相关颜色都会改变，如果你看过我之前写的[**Android Design Support Library--TextInputLayout的使用**](http://blog.csdn.net/wei_smile/article/details/51284964)你会知道TextInputLayout下划线的颜色也是通过这个属性来更改的，所以为了稳定起见还是使用官方提供的方法去更改吧，我这纯属抖个机灵。

那么回到正题，讲讲SnackBar的回调，眼尖的朋友可能发现了，我的Demo里面带回调的SnackBar在弹出和消失的时候都会有Toast通知出现，其实就是使用了SnackBar自带的

```
setCallback(Snackbar.Callback callback)
```
方法，这里需要传入一个`Snackbar.Callback callback` 参数，其实这个，这个Callback 是SnackBar内部的一个抽象类，它内部有两个空实现的方法：

```java
onDismissed(Snackbar snackbar, int event)

onShown(Snackbar snackbar)
```
顾名思义，我们可以可以分别在这两个方法中定义出当SnackBar消失和产生时我们需要做的事，这两个方法会在SnackBar消失和产生时被回调，打个比方：

```java
mSnackBar.setCallback(new Snackbar.Callback() {
            @Override
            public void onDismissed(Snackbar snackbar, int event) {
                super.onDismissed(snackbar, event);
                Toast.makeText(MainActivity.this,"SnackBar Dismiss",Toast.LENGTH_SHORT).show();
            }

            @Override
            public void onShown(Snackbar snackbar) {
                super.onShown(snackbar);
                Toast.makeText(MainActivity.this,"SnackBar Show",Toast.LENGTH_SHORT).show();
            }
        });
```

这样就实现了在SnackBar消失和产生时弹出Toast通知的动作，其他具体的逻辑可以自己去实现。

#### 完全自定义你自己的SnackBar
如果你对上述使用还是不甚满意，那么接下来我教你怎么自定义你自己的SnackBar，说实话用到的场景并不多，但是学了就学个透彻，这一部分知识的灵感来自于[**没时间解释了，快使用Snackbar!**](http://www.jianshu.com/p/cd1e80e64311) ，SnackBar并没有提供更改背景或者其他样式的方法，但是我们可以通过查看源码来试试可不可以自定义自己样式，我们找到这么一段代码：

```java
    private Snackbar(ViewGroup parent) {
        mTargetParent = parent;
        mContext = parent.getContext();

        ThemeUtils.checkAppCompatTheme(mContext);

        LayoutInflater inflater = LayoutInflater.from(mContext);
        mView = (SnackbarLayout) inflater.inflate(
                R.layout.design_layout_snackbar, mTargetParent, false);
    }
```
最后一行的inflate是不是很熟悉，我们可不可以认为Snackbar的布局就是这么加载的，这个SnackBarLayout是在SnackBar内部定义的一个继承自LinearLayout的内部类：

```java
 public static class SnackbarLayout extends LinearLayout {
        private TextView mMessageView;
        private Button mActionView;

        private int mMaxWidth;
        private int mMaxInlineActionWidth;
```
看到这几个变量的定义，我已经确定了上面的想法，接下来我们找到上面代码加载的那段布局：

```java
<merge xmlns:android="http://schemas.android.com/apk/res/android">
<TextView
        android:id="@+id/snackbar_text"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_weight="1"
        android:paddingTop="14dp"
        android:paddingBottom="14dp"
        android:paddingLeft="12dp"
        android:paddingRight="12dp"
        android:textAppearance="@style/TextAppearance.Design.Snackbar.Message"
        android:maxLines="2"
        android:layout_gravity="center_vertical|left|start"
        android:ellipsize="end"
        android:textAlignment="viewStart"/>

<Button
        android:id="@+id/snackbar_action"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginLeft="0dp"
        android:layout_marginStart="0dp"
        android:layout_gravity="center_vertical|right|end"
        android:paddingTop="14dp"
        android:paddingBottom="14dp"
        android:paddingLeft="12dp"
        android:paddingRight="12dp"
        android:visibility="gone"
        android:textColor="?attr/colorAccent"
        style="?attr/borderlessButtonStyle"/>
</merge>
```
看到这两个控件的ID了么

```java
android:id="@+id/snackbar_text"
android:id="@+id/snackbar_action"

```

那么第一个就是SnackBar左边显示的message，第二个就是我们设置了action时候显示的Button咯，这就简单了，如果你仔细看了上面提供的API文档你会发现有这么一个方法：

```java
public View getView ()

Returns the Snackbar's view.
```
这个方法可以返回我们SnackBar的View，那么这个View是什么，看源码：

```java
    /**
     * Returns the {@link Snackbar}'s view.
     */
    @NonNull
    public View getView() {
        return mView;
    }
```
找一下mView在哪里定义的：

```java
private final SnackbarLayout mView;
mView = (SnackbarLayout) inflater.inflate(
                R.layout.design_layout_snackbar, mTargetParent, false);
```
好了，这下一切都清楚了，接下里示范一下怎么自定义你自己的SnackBar：

```java
private View view;

     ....省略中间代码
     
 view = mCustomSnackBar.getView();
        if (view != null) {
            view.setBackgroundColor(Color.parseColor("#7B68EE"));
            //获取Snackbar的message控件，修改字体颜色
            ((TextView) view.findViewById(R.id.snackbar_text)).setTextColor(Color.parseColor("#FFDAB9"));
            //添加图标
            Snackbar.SnackbarLayout snackbarLayout = (Snackbar.SnackbarLayout) view;
            //添加自定义布局，这里布局就包含了一个ImageView
            //custom_layout是你自定义的布局
            View add_view = LayoutInflater.from(view.getContext()).inflate(R.layout.custom_layout, null);
            LinearLayout.LayoutParams p = new LinearLayout.LayoutParams(LinearLayout.LayoutParams.WRAP_CONTENT, LinearLayout.LayoutParams.WRAP_CONTENT);
            p.gravity = Gravity.CENTER_VERTICAL;
            //数字表示新加的布局在SnackBar中的位置，从0开始,取决于你SnackBar里面有多少个子View
            snackbarLayout.addView(add_view, 0, p);
        }


```
**最后一行，addView方法第二个参数表示新加的布局在SnackBar中的位置，注意不要超过总的View的个数不然会报错**，message和Action text分别算一个View，其他的话注释已经写得很清楚就不一一解释了，这个代码呈现的效果如下：

![这里写图片描述](http://img.blog.csdn.net/20160507124757478)

为了方便自定义样式，发现这一特性的作者还给我们封装了成为一个工具类：
```java
/**
 * Created by 赵晨璞 on 2016/5/1.
 */
public class SnackbarUtil {

public static final   int Info = 1;
public static final  int Confirm = 2;
public static final  int Warning = 3;
public static final  int Alert = 4;


public static  int red = 0xfff44336;
public static  int green = 0xff4caf50;
public static  int blue = 0xff2195f3;
public static  int orange = 0xffffc107;

/**
 * 短显示Snackbar，自定义颜色
 * @param view
 * @param message
 * @param messageColor
 * @param backgroundColor
 * @return
 */
public static Snackbar ShortSnackbar(View view, String message, int messageColor, int backgroundColor){
    Snackbar snackbar = Snackbar.make(view,message, Snackbar.LENGTH_SHORT);
    setSnackbarColor(snackbar,messageColor,backgroundColor);
    return snackbar;
}

/**
 * 长显示Snackbar，自定义颜色
 * @param view
 * @param message
 * @param messageColor
 * @param backgroundColor
 * @return
 */
public static Snackbar LongSnackbar(View view, String message, int messageColor, int backgroundColor){
    Snackbar snackbar = Snackbar.make(view,message, Snackbar.LENGTH_LONG);
    setSnackbarColor(snackbar,messageColor,backgroundColor);
    return snackbar;
}

/**
 * 自定义时常显示Snackbar，自定义颜色
 * @param view
 * @param message
 * @param messageColor
 * @param backgroundColor
 * @return
 */
public static Snackbar IndefiniteSnackbar(View view, String message,int duration,int messageColor, int backgroundColor){
    Snackbar snackbar = Snackbar.make(view,message, Snackbar.LENGTH_INDEFINITE).setDuration(duration);
    setSnackbarColor(snackbar,messageColor,backgroundColor);
    return snackbar;
}

/**
 * 短显示Snackbar，可选预设类型
 * @param view
 * @param message
 * @param type
 * @return
 */
public static Snackbar ShortSnackbar(View view, String message, int type){
    Snackbar snackbar = Snackbar.make(view,message, Snackbar.LENGTH_SHORT);
    switchType(snackbar,type);
    return snackbar;
}

/**
 * 长显示Snackbar，可选预设类型
 * @param view
 * @param message
 * @param type
 * @return
 */
public static Snackbar LongSnackbar(View view, String message,int type){
    Snackbar snackbar = Snackbar.make(view,message, Snackbar.LENGTH_LONG);
    switchType(snackbar,type);
    return snackbar;
}

/**
 * 自定义时常显示Snackbar，可选预设类型
 * @param view
 * @param message
 * @param type
 * @return
 */
public static Snackbar IndefiniteSnackbar(View view, String message,int duration,int type){
    Snackbar snackbar = Snackbar.make(view,message, Snackbar.LENGTH_INDEFINITE).setDuration(duration);
    switchType(snackbar,type);
    return snackbar;
}

//选择预设类型
private static void switchType(Snackbar snackbar,int type){
    switch (type){
        case Info:
            setSnackbarColor(snackbar,blue);
            break;
        case Confirm:
            setSnackbarColor(snackbar,green);
            break;
        case Warning:
            setSnackbarColor(snackbar,orange);
            break;
        case Alert:
            setSnackbarColor(snackbar,Color.YELLOW,red);
            break;
    }
}

/**
 * 设置Snackbar背景颜色
 * @param snackbar
 * @param backgroundColor
 */
public static void setSnackbarColor(Snackbar snackbar, int backgroundColor) {
    View view = snackbar.getView();
    if(view!=null){
        view.setBackgroundColor(backgroundColor);
    }
}

/**
 * 设置Snackbar文字和背景颜色
 * @param snackbar
 * @param messageColor
 * @param backgroundColor
 */
public static void setSnackbarColor(Snackbar snackbar, int messageColor, int backgroundColor) {
    View view = snackbar.getView();
    if(view!=null){
        view.setBackgroundColor(backgroundColor);
        ((TextView) view.findViewById(R.id.snackbar_text)).setTextColor(messageColor);
    }
}

/**
 * 向Snackbar中添加view
 * @param snackbar
 * @param layoutId
 * @param index 新加布局在Snackbar中的位置
 */
public static void SnackbarAddView( Snackbar snackbar,int layoutId,int index) {
    View snackbarview = snackbar.getView();
    Snackbar.SnackbarLayout snackbarLayout=(Snackbar.SnackbarLayout)snackbarview;

    View add_view = LayoutInflater.from(snackbarview.getContext()).inflate(layoutId,null);

    LinearLayout.LayoutParams p = new LinearLayout.LayoutParams( LinearLayout.LayoutParams.WRAP_CONTENT,LinearLayout.LayoutParams.WRAP_CONTENT);
    p.gravity= Gravity.CENTER_VERTICAL;

    snackbarLayout.addView(add_view,index,p);
}

}
```
使用示例如下：

```java
SnackbarUtil.ShortSnackbar(coordinator,"妹子向你发来一条消息",SnackbarUtil.Info).show();
```
在此要非常感谢[**简名**](http://www.jianshu.com/users/990c16f1edc0/latest_articles) 给我们提供这么好的工具类，那么还有什么不懂得可以留言探讨，下面上整个项目的代码：

**MainActivity.java**
``` java
package com.muyang.snackbardemo;

import android.graphics.Color;
import android.os.Bundle;
import android.support.design.widget.CoordinatorLayout;
import android.support.design.widget.Snackbar;
import android.support.v7.app.AppCompatActivity;
import android.view.Gravity;
import android.view.LayoutInflater;
import android.view.View;
import android.widget.Button;
import android.widget.LinearLayout;
import android.widget.TextView;
import android.widget.Toast;

public class MainActivity extends AppCompatActivity implements View.OnClickListener {

    private Button btn_normal, btn_callback, btn_custom;
    private CoordinatorLayout mCoor;
    private Snackbar mSnackBar, mCustomSnackBar;
    private View view;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        initWidget();
        mSnackBar = Snackbar.make(mCoor, R.string.callback, Snackbar.LENGTH_SHORT)
                .setAction(R.string.UNDO, new View.OnClickListener() {
                    @Override
                    public void onClick(View v) {

                    }
                });
        //SnackBar回调方法
        mSnackBar.setCallback(new Snackbar.Callback() {
            @Override
            public void onDismissed(Snackbar snackbar, int event) {
                super.onDismissed(snackbar, event);
                Toast.makeText(MainActivity.this,"SnackBar Dismiss",Toast.LENGTH_SHORT).show();
            }

            @Override
            public void onShown(Snackbar snackbar) {
                super.onShown(snackbar);
                Toast.makeText(MainActivity.this,"SnackBar Show",Toast.LENGTH_SHORT).show();
            }
        });

//       1、Resources resource = (Resources) getBaseContext().getResources();
//       ColorStateList csl = (ColorStateList) resource.getColorStateList(R.color.PeachPuff);
//       mSnackBar.setActionTextColor(csl);
//       2、mSnackBar.setActionTextColor(Color.rgb(232,44,123))
//       3、mSnackBar.setActionTextColor(Color.BLUE)
        mSnackBar.setActionTextColor(Color.parseColor("#FFDAB9"));

        //自定义SnackBar样式
        mCustomSnackBar = Snackbar.make(mCoor, R.string.custom, Snackbar.LENGTH_SHORT)
                .setAction(R.string.UNDO, new View.OnClickListener() {
                    @Override
                    public void onClick(View v) {

                    }
                });
        mCustomSnackBar.setActionTextColor(Color.parseColor("#FFDAB9"));
        //获得SnackBar这个View
        view = mCustomSnackBar.getView();
        if (view != null) {
            view.setBackgroundColor(Color.parseColor("#7B68EE"));
            //获取Snackbar的message控件，修改字体颜色
            ((TextView) view.findViewById(R.id.snackbar_text)).setTextColor(Color.parseColor("#FFDAB9"));
            //添加图标
            Snackbar.SnackbarLayout snackbarLayout = (Snackbar.SnackbarLayout) view;
            //custom_layout是你自定义的布局
            View add_view = LayoutInflater.from(view.getContext()).inflate(R.layout.custom_layout, null);
            LinearLayout.LayoutParams p = new LinearLayout.LayoutParams(LinearLayout.LayoutParams.WRAP_CONTENT, LinearLayout.LayoutParams.WRAP_CONTENT);
            p.gravity = Gravity.CENTER_VERTICAL;
            //数字表示新加的布局在SnackBar中的位置，从0开始,取决于你SnackBar里面有多少个子View
            snackbarLayout.addView(add_view, 0, p);
        }
    }


    private void initWidget() {
        btn_normal = (Button) findViewById(R.id.btn_normal);
        btn_normal.setOnClickListener(this);
        btn_callback = (Button) findViewById(R.id.btn_callback);
        btn_callback.setOnClickListener(this);
        btn_custom = (Button) findViewById(R.id.btn_custom);
        btn_custom.setOnClickListener(this);
        mCoor = (CoordinatorLayout) findViewById(R.id.coordinatorLayout);
    }


    @Override
    public void onClick(View v) {
        switch (v.getId()) {
            case R.id.btn_normal:
                Snackbar.make(mCoor, R.string.normal, Snackbar.LENGTH_SHORT).show();
                break;
            case R.id.btn_callback:

//                Snackbar.make(mCoor,R.string.callback,Snackbar.LENGTH_SHORT)
//                        .setAction(R.string.UNDO, new View.OnClickListener() {
//                            @Override
//                            public void onClick(View v) {
//
//                            }
//                        }).show();
                mSnackBar.show();
                break;
            case R.id.btn_custom:
                mCustomSnackBar.show();
//                if (mSnackBar.isShown()) {
//                    mSnackBar.dismiss();
//                }
                break;
        }
    }
}

```

**activity_main.xml**

```java
<?xml version="1.0" encoding="utf-8"?>
<android.support.design.widget.CoordinatorLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/coordinatorLayout"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <RelativeLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent">

        <Button
            android:id="@+id/btn_normal"
            android:layout_width="150dp"
            android:layout_height="wrap_content"
            android:layout_alignParentTop="true"
            android:layout_centerHorizontal="true"
            android:layout_marginTop="80dp"
            android:background="@drawable/btn_bg"
            android:text="@string/normal"
            android:textSize="15sp" />

        <Button
            android:id="@+id/btn_callback"
            android:layout_width="150dp"
            android:layout_height="wrap_content"
            android:layout_alignStart="@+id/btn_normal"
            android:layout_below="@+id/btn_normal"
            android:layout_marginTop="38dp"
            android:background="@drawable/btn_bg"
            android:text="@string/callback"
            android:textSize="15sp" />

        <Button
            android:id="@+id/btn_custom"
            android:layout_width="150dp"
            android:layout_height="wrap_content"
            android:layout_alignStart="@+id/btn_callback"
            android:layout_below="@+id/btn_callback"
            android:layout_marginTop="45dp"
            android:background="@drawable/btn_bg"
            android:text="@string/custom"
            android:textSize="15sp" />
    </RelativeLayout>

</android.support.design.widget.CoordinatorLayout>

```

**custom_layout.xml**

```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:layout_gravity="center_vertical"
    >
    <ImageView
        android:layout_width="35dp"
        android:layout_height="35dp"
        android:layout_gravity="center_vertical"
        android:src="@drawable/header"/>
</LinearLayout>
```
#### 彩蛋

[SnackBar源码](https://github.com/android/platform_frameworks_support/blob/62eb3105e51335cf9074a5506d8d2b220aeb95dc/design/src/android/support/design/widget/Snackbar.java)

[项目源码](http://download.csdn.net/detail/wei_smile/9512799)
[项目GitHub地址](https://github.com/GiitSmile/SnackBarDemo)

----------


**参考**
[没时间解释了，快使用Snackbar!](http://www.jianshu.com/p/cd1e80e64311)
[Android Material Design Snackbar Example](http://www.androidhive.info/2015/09/android-material-design-snackbar-example/)

喜欢Android Design Support Library系列的朋友欢迎关注我的微信公众号未央进化论，第一时间通知博客更新，荆轲刺秦王(＾－＾)