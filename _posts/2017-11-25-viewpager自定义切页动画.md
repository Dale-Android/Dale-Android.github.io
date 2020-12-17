---
layout: post
title: "ViewPager自定义切页动画"
date: "2017-11-25 11:41:00 +0800"
description: ""
img: 
tags: [Android, 动画]
mathjax: true
---

ViewPager，算是Android开发中的一个十分常用的组件了。我们今天来讨论下ViewPager的切面动画定制。

# 自定义切页动画

默认情况下，系统已经给ViewPager切页添加了一个滑动动画，简单低调，效果也不赖。

不过，如果对这个切页动画有特殊需求，我们也可以自定义一个动画效果，只需要实现系统接口*ViewPager.PageTransformer*，然后调用方法*setPageTransformer*()设置给ViewPager即可。

## PageTransformer简介

***PageTransformer***，译成“页面变形金刚”是不是略微高调了？还是叫原名吧……其实，这个接口名自带变形金刚属性（Transformer），自然就告诉我们，通过它，我们可以实现出各种各样的切页动画。

```java
	/**
     * A PageTransformer is invoked whenever a visible/attached page is scrolled.
     * This offers an opportunity for the application to apply a custom transformation
     * to the page views using animation properties.
     *
     * <p>As property animation is only supported as of Android 3.0 and forward,
     * setting a PageTransformer on a ViewPager on earlier platform versions will
     * be ignored.</p>
     */
    public interface PageTransformer {
        /**
         * Apply a property transformation to the given page.
         *
         * @param page Apply the transformation to this page
         * @param position Position of page relative to the current front-and-center
         *                 position of the pager. 0 is front and center. 1 is one full
         *                 page position to the right, and -1 is one page position to the left.
         */
        void transformPage(View page, float position);
    }
```

每当有可见的ViewPage页滚动时，都将触发PageTransformer回调。借助属性动画系统，在方法*transformPage*回调的时候，就可以通过改变属性值来实现定制动画了。

对于页面切换过程中的每一个状态点，回调*transformPage*方法的页面包括：

- 当前可见页
- 当前页相邻前（左）页
- 当前页相邻后（右）页

*transformPage*的第一个参数就是回调页的引用。第二个参数position是指回调页相对于屏幕中央的定位值，将会随着滚动页变化。position的值的意义，如下表说明一下。

|position值|说明|备注|
|:-:|:-|:-|
|0|页占满屏幕，即完全显示状态的定位值||
|-1|当前页占满屏幕时的前一页定位值||
|1|当前页占满屏幕时的后一页定位值||
|(-1, 0)|前一页渐入/渐出的定位值|渐入时，终点定位为0，即将成为“当前页”|
|(0, 1)|后一页渐入/渐出的定位值|渐入时，终点定位为0，即将成为“当前页”|

如果前页、当前页、后页的定位值分别用P、C、N表示，那它们满足以下等式：

	C - P = N - C = 1

其实，上面等式的原因就是：**“前页+当前页”，或“当前页+后页”，总是构成一个整屏**。


## DepthPageTransformer

说了这么多，先用官方的例子来具体看看PageTransformer如何实现吧。

```java
public class DepthPageTransformer implements ViewPager.PageTransformer {
    private static final float MIN_SCALE = 0.75f;

    public void transformPage(View view, float position) {
        int pageWidth = view.getWidth();

        if (position < -1) { // [-Infinity,-1)
            // This page is way off-screen to the left.
            view.setAlpha(0);

        } else if (position <= 0) { // [-1,0]
            // Use the default slide transition when moving to the left page
            view.setAlpha(1);
            view.setTranslationX(0);
            view.setScaleX(1);
            view.setScaleY(1);

        } else if (position <= 1) { // (0,1]
            // Fade the page out.
            view.setAlpha(1 - position);

            // Counteract the default slide transition
            view.setTranslationX(pageWidth * -position);

            // Scale the page down (between MIN_SCALE and 1)
            float scaleFactor = MIN_SCALE
                    + (1 - MIN_SCALE) * (1 - Math.abs(position));
            view.setScaleX(scaleFactor);
            view.setScaleY(scaleFactor);

        } else { // (1,+Infinity]
            // This page is way off-screen to the right.
            view.setAlpha(0);
        }
    }
}
```

从代码上来讲，动画包括了alpha、translation和scale三个属性变化，最终实现“深度式切页”动画。根据回调的position值，确定了当前回调页的类型（如前文表格），然后设置不同的状态属性值。简单明了。

效果如下图

![1.gif.gif](/assets/post_resources/0401.gif)


## DiagonalZoomPageTransformer（对角线缩放切页）

官方例子，很简单，照猫画狐，也不是难事。下面我们来实现一个延对角线渐显进入、渐隐退出，并且伴随尺寸放缩的切页动画。

直接上代码。

```java
public class DiagonalZoomPageTransformer implements ViewPager.PageTransformer {
    private static final float MIN_SCALE = 0.15f;

    public void transformPage(View view, float position) {

        if (position < -1) { // [-Infinity,-1)
            // This page is way off-screen to the left.
            view.setAlpha(0);

        } else if (position <= 0) { // [-1,0]
            update(view, Math.abs(position), true);
        } else if (position <= 1) { // (0,1]
            update(view, Math.abs(position), false);
        } else { // (1,+Infinity]
            // This page is way off-screen to the right.
            view.setAlpha(0);
        }
    }

    private void update(View v, float absPosition, boolean left) {
        int pageWidth = v.getWidth();
        int pageHeight = v.getHeight();

        float position = left ? -absPosition : absPosition;
        v.setAlpha(1 + position);

        v.setTranslationX(pageWidth * -position + pageWidth * (1 - MIN_SCALE) * position / 2);
        v.setTranslationY(pageHeight * (1 - MIN_SCALE) * -position / 2);

        float scale = MIN_SCALE + (1 - MIN_SCALE) * (1 - Math.abs(position));
        v.setScaleX(scale);
        v.setScaleY(scale);
    }
}
```
重点在于两个else if分支，它们都调用方法update方法进行更新，和控制属性动画一样，很简单。

效果如下。
![2.gif.gif](/assets/post_resources/0402.gif)

# 小结
一般情况下，对于ViewPager的切页效果，都不会有特殊需求。当然，有特殊