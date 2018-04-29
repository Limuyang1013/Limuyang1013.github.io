---
title: 使用ViewPager动画来做出不一样的引导页
date: 2016-04-16 20:29:26
categories: Android
tags: [引导页,ViewPager,动画]
---

就算Google从很早开始就自带了设置引导页动画的接口，但是就我目前看来市面上使用引导页动画的还是很少的，也不知道是为什么，一想到Material Design的使用率也这么少表示很心塞。
首先来看看市面上千篇一律的引导页效果，诺：

![这里写图片描述](http://img.my.csdn.net/uploads/201604/16/1460774446_5006.gif)

很单调对不对，你们没看吐我都看吐了，再看一份加了引导页动画效果的：
![这里写图片描述](http://img.my.csdn.net/uploads/201604/16/1460774609_1329.gif)

有没有瞬间耳目一新的感觉，下面就谈谈如何做出这样的引导页动画。

<!-- more -->
其实从Android 3.0也就是API 11开始Android就自带了一个PageTransformer接口用来实现ViewPager动画效果并为之加入了setPageTransformer方法来自定义我们自己的动画效果，用的时候很简单：

```java
viewpager.setPageTransformer(false, new ViewPager.PageTransformer() {
    @Override
    public void transformPage(View page, float position) {
        // do transformation here
        }
});
```
这里setPageTransformer传入了两个参数，第一个布尔型参数表示的意思就是在两个页面切换产生动画效果时候是否要反转一下让下一个页面在上一个页面底下，因为ViewPager默认下一个页面是绘制在上一个页面的上面，这里一般传入true，第二次参数才是重点，这里实现了PageTransformer接口，然后我们所有需要的动画效果都在transformPage这个接口方法里面实现，现在我们来看看这个方法。

我们发现transformPage方法也有两个参数，第一个参数就表示当前显示在屏幕上的Activity或者Fragment，这个不管，后一个参数是重点，这个position并不是我们引导页页面的position，下面借用一张图来说明这个position参数：
![这里写图片描述](http://img.blog.csdn.net/20160416110043276)

> [图片出处](http://chiemy.com/android/great-animation-with-pagetranformer)

谷歌官方对这个参数的解释是，这个参数表明了一个给定的页面相对于屏幕中心的位置，并且这个参数随着我们的滑动会动态的变化，最重要的一点是这个position参数是相对于我们的屏幕左边缘来说的，如果当前的页面刚好占满了整个屏幕，就如上图所示的1界面，那么这个页面的position参数就是0，如果一个页面刚从屏幕右边缘划出来，可以理解为上图中页面1和页面2的交界线刚冒头的情况，那么这个页面的position值就是1，如果某个时刻我们把页面1向左滑动一半，导致屏幕中央既有一半页面1显示又有一半页面2显示的时候，这时候页面1的position值就是-0.5，页面2的position值就是0.5，都是相对于左边缘来说的，而且左边缘的值固定是0，就是下面这种情况：
![这里写图片描述](http://img.my.csdn.net/uploads/201604/16/1460776252_2170.gif)

那么知道这个值有什么用呢，其实多亏了这个值我们才能做出更好的动画效果，要知道这个值是动态变化的，有了一个动态变化的值就可以做出动态变化的效果，我们可以看看谷歌是怎么用它的：

```java
public class ZoomOutPageTransformer implements ViewPager.PageTransformer {
    private static final float MIN_SCALE = 0.85f;
    private static final float MIN_ALPHA = 0.5f;

    public void transformPage(View view, float position) {
        int pageWidth = view.getWidth();
        int pageHeight = view.getHeight();

        if (position < -1) { // [-Infinity,-1)
            // This page is way off-screen to the left.
            view.setAlpha(0);

        } else if (position <= 1) { // [-1,1]
            // Modify the default slide transition to shrink the page as well
            float scaleFactor = Math.max(MIN_SCALE, 1 - Math.abs(position));
            float vertMargin = pageHeight * (1 - scaleFactor) / 2;
            float horzMargin = pageWidth * (1 - scaleFactor) / 2;
            if (position < 0) {
                view.setTranslationX(horzMargin - vertMargin / 2);
            } else {
                view.setTranslationX(-horzMargin + vertMargin / 2);
            }

            // Scale the page down (between MIN_SCALE and 1)
            view.setScaleX(scaleFactor);
            view.setScaleY(scaleFactor);

            // Fade the page relative to its size.
            view.setAlpha(MIN_ALPHA +
                    (scaleFactor - MIN_SCALE) /
                    (1 - MIN_SCALE) * (1 - MIN_ALPHA));

        } else { // (1,+Infinity]
            // This page is way off-screen to the right.
            view.setAlpha(0);
        }
    }
}
```

这份代码的实际效果如下：
![这里写图片描述](http://img.my.csdn.net/uploads/201604/16/1460776900_4978.gif)
可以看到很明显的缩放以及透明度的变换，我们看看代码是怎么利用这个position参数的：

```java
 if (position < -1) { // [-Infinity,-1)
            // This page is way off-screen to the left.
            view.setAlpha(0);
```
首先这里判断如果position参数是负无穷到-1的时候，也就是此时这个page划出了屏幕左边缘之外不可见得情况，直接设置为透明的

```java
} else if (position <= 1) { // [-1,1]
            // Modify the default slide transition to shrink the page as well
            float scaleFactor = Math.max(MIN_SCALE, 1 - Math.abs(position));
            float vertMargin = pageHeight * (1 - scaleFactor) / 2;
            float horzMargin = pageWidth * (1 - scaleFactor) / 2;
            if (position < 0) {
                view.setTranslationX(horzMargin - vertMargin / 2);
```

接下里判断如果这个页面的position处于-1到1之间，也就是最极端的情况是这个页面即将向左边滑出屏幕之外，或者说这个页面还在屏幕右边缘即将滑入我们的视线，这时就用到了我们的position参数，这里谷歌把它当成了一个缩放系数来用;

```java
float scaleFactor = Math.max(MIN_SCALE, 1 - Math.abs(position));
```
这里`1 - Math.abs(position))` 的值在0~1之间，并且代码判断了MIN_SCALE和`1 - Math.abs(position))` 的大小并取其中的最大值，MIN_SCALE是最上面定义的最小缩放系数，可以看到这一行代码得出来的缩放系数是处于MIN_SCALE和1之间的，然后下面的代码就是根据position位置参数的变化来进行page的水平移动效果，其实ViewPager提供了相当多的方法共给我们去操作，诸如:setScale设置缩放啦，setRotation设置旋转效果，还有这里的setTranslationX设置水平移动效果等等，结合position这个动态变化的参数可以做出很多意想不到的动画，比如下面这个旋转动画：
![这里写图片描述](http://img.my.csdn.net/uploads/201604/16/1460777751_3362.gif)
代码如下：

```java
public class  RotateDownPageTransformer implements ViewPager.PageTransformer
{

    private static final float ROT_MAX = 20.0f;
    private float mRot;



    public void transformPage(View view, float position)
    {

        Log.e("TAG", view + " , " + position + "");

        if (position < -1)
        { // [-Infinity,-1)
            // This page is way off-screen to the left.
            ViewHelper.setRotation(view, 0);

        } else if (position <= 1) // a页滑动至b页 ； a页从 0.0 ~ -1 ；b页从1 ~ 0.0
        { // [-1,1]
            // Modify the default slide transition to shrink the page as well
            if (position < 0)
            {

                mRot = (ROT_MAX * position);
                ViewHelper.setPivotX(view, view.getMeasuredWidth() * 0.5f);
                ViewHelper.setPivotY(view, view.getMeasuredHeight());
                ViewHelper.setRotation(view, mRot);
            } else
            {

                mRot = (ROT_MAX * position);
                ViewHelper.setPivotX(view, view.getMeasuredWidth() * 0.5f);
                ViewHelper.setPivotY(view, view.getMeasuredHeight());
                ViewHelper.setRotation(view, mRot);
            }

            // Scale the page down (between MIN_SCALE and 1)

            // Fade the page relative to its size.

        } else
        { // (1,+Infinity]
            // This page is way off-screen to the right.
            ViewHelper.setRotation(view, 0);
        }
    }
}
```
注意：这里的ViewHelper类需要引用nineoldandroids第三方动画库来实现，具体的东西我都打包好了在最下面有下载链接，需要的自行取用。

#### 更酷炫的效果
其实除了对position这个参数进行处理，我们还可以对view进行动画处理，前面提到transformPage这个接口方法第一个参数view就是当前可见的Activity或Fragment，那么我们还可以对这个Activity或Fragment内部的元素进行处理来达到视察动画的效果，一个很好地例子：
<div><img src="https://d262ilb51hltx0.cloudfront.net/max/600/1*zD4p2a5gBqt63PQH9ZLNdQ.gif" /></div>

----------


```java
public void transformPage(View view, float position) {
    int pageWidth = view.getWidth();
        
    if (position < -1) { // [-Infinity,-1)
        // This page is way off-screen to the left.
        view.setAlpha(0);

    } else if (position <= 1) { // [-1,1]
          
		  
        mBlur.setTranslationX((float) (-(1 - position) * 0.5 * pageWidth));
		mBlurLabel.setTranslationX((float) (-(1 - position) * 0.5 * pageWidth));

		mDim.setTranslationX((float) (-(1 - position) * pageWidth));
		mDimLabel.setTranslationX((float) (-(1 - position) * pageWidth));

		mCheck.setTranslationX((float) (-(1 - position) * 1.5 * pageWidth));
		mDoneButton.setTranslationX((float) (-(1 - position) * 1.7 * pageWidth)); 
		// The 0.5, 1.5, 1.7 values you see here are what makes the view move in a different speed.
		// The bigger the number, the faster the view will translate.
		// The result float is preceded by a minus because the views travel in the opposite direction of the movement.

		mFirstColor.setTranslationX((position) * (pageWidth / 4));

		mSecondColor.setTranslationX((position) * (pageWidth / 1));

		mTint.setTranslationX((position) * (pageWidth / 2));

		mDesaturate.setTranslationX((position) * (pageWidth / 1));
		// This is another way to do it
		  
		  
    } else { // (1,+Infinity]
        // This page is way off-screen to the right.
        view.setAlpha(0);
    }
}
```
[工程地址](https://github.com/chiemy/PageTransformerDemo)

最后，附上整个项目的地址：[项目地址](http://download.csdn.net/detail/wei_smile/9492930)
有任何问题可以留言探讨