---
layout: post
title: "Android动画 —— Activity过渡"
date: "2017-10-12 00:20:06 +0800"
description:
img:
tags: [Android, 动画]
mathjax: true
---

学如逆水行舟，不进则退。

接触Android开发虽已经颇有时日，但感觉相关知识总停留于一知半解，或者又缺乏系统关联导致顾此而失彼。是时候总结一下了。

那就从常常开发过程中经常遇到的Android动画开始吧。

Material设计为Android增加了一种动画称为Activity过渡（*Activity Transitions*），旨在提供不同状态下共有元素的视觉连接。官方文档的描述，听起来拗口，读起来也不够顺畅。Activity过渡在我接触的项目中没有遇到过，今天就以它初试牛刀，既是学习，也是巩固。

# 类别

Activity的过渡动画总共可以分为三类：

- **进入过渡**（*Enter transition*）：定义Activity中的view如何进入界面
- **退出过渡**（*Exit transition*）：定义Activity中的view如何退出界面
- **共享元素过渡**（*Shared elements transition*）： 如果两个Activity包含了若干“共享元素”（可以理解为相同的元素控件，至少，看起来是如此），那么**共享元素过渡**动画，就是将这些共享元素从一个Activity优雅地过渡到另一个Activity

任何继承了***Visibility***类的过渡，都可以作为进入或者退出过渡。目前，Android默认支持的进入/退出过渡如下表。

|过渡|说明|
|:-:|:-:|
|*explode*|移动view进入/退出界面中央|
|*slide*|从界面的某一条边移入/移出view|
|*fade*|通过改变透明度来展现view的添加和移出|

Android默认支持的共享元素过渡如下表。

|过渡|说明|
|:-:|:-:|
|*changeBounds*|过渡view的layout bound变化|
|*changeClipBounds*|过渡view的clip bound变化|
|*changeTransform*|过渡view的scale和rotation的变化|
|*changeImageTransform*|过渡image的尺寸和放缩比例变化|

# 过渡动画
## 进入/退出过渡

首先，开启过渡动画需要在style中将windowActivityTransitions属性设置为true
```xml
	<item name="android:windowActivityTransitions">true</item>
```
或者，也可以在代码中调用方法来设置

```java
	getWindow().requestFeature(Window.FEATURE_CONTENT_TRANSITIONS);
```

前面已经说过，进入、退出过渡，是指Activity中的view元素，在Activity界面显示时的动画过渡。下面，我们使用默认的***Explode***过渡来举个例子。

### (1) SecondActivity

定义SecondActivity，用于显示进入和退出过渡，界面为一个TextView和四个定位View（用处后面会说），代码如下

```xml
	<?xml version="1.0" encoding="utf-8"?>
	<FrameLayout xmlns:android="http://schemas.android.com/apk/res/android"
	    xmlns:tools="http://schemas.android.com/tools"
	    android:layout_width="match_parent"
	    android:layout_height="match_parent"
	    tools:context="com.djx.activitytransition.SecondActivity">

	    <View
	        android:layout_width="16dp"
	        android:layout_height="16dp"
	        android:layout_margin="64dp"
	        android:layout_gravity="top"
	        android:background="#888" />
	
	    <View
	        android:layout_width="16dp"
	        android:layout_height="16dp"
	        android:layout_margin="64dp"
	        android:layout_gravity="top|right"
	        android:background="#888" />
	
	    <View
	        android:layout_width="16dp"
	        android:layout_height="16dp"
	        android:layout_margin="64dp"
	        android:layout_gravity="bottom"
	        android:background="#888" />
	
	    <View
	        android:layout_width="16dp"
	        android:layout_height="16dp"
	        android:layout_margin="64dp"
	        android:layout_gravity="bottom|right"
	        android:background="#888" />
	
	    <TextView
	        android:id="@+id/hello"
	        android:layout_width="wrap_content"
	        android:layout_height="wrap_content"
	        android:layout_gravity="center"
	        android:transitionName="test_text"
	        android:textSize="45sp"
	        android:text="Hello"/>
	
	</FrameLayout>
```

同时，在onCreate中设置并开启进入过渡，并使用内置的Explode效果。

```java
	getWindow().setEnterTransition(new Explode());
```

### (2) MainActivity

主界面Activity，用于启动SecondActivity，并添加transition的选项bundle

```java
	Intent intent = new Intent(this, SecondActivity.class);
	ActivityOptions options = ActivityOptions.makeSceneTransitionAnimation(this);
	startActivity(intent, options != null ? options.toBundle() : null);
```

### (3) 过渡效果

启动SecondActivity，待界面显示出来后，点击返回键，退出。效果如下图

![Enter and exit.gif](/assets/post_resources/01.gif)

正如Explode的描述，进入时，界面中的所有View都是从外部往中间飞入（注意方向是中心）；而退出时，所有View都是以中心为源而向外飞散。上面定义的文本及四个不同位置的定位View，就是为了让这个现象更清楚。

例子中都是使用代码方式控制了过渡，其实，更加简单地，可以直接在style里面设置windowEnterTransition和windowExitTransition两个属性来实现。这里就不赘述了。

值得注意的是，**启动一个Activity，或者是退出覆盖其上的其他Activity，使其切回前台，都是一个“进入”过程，都会执行进入过渡；同样地，退出一个Activity，或者在其界面启动其他Activity导致其退到后台，都是一个“退出”过程，也都会执行退出过渡**。而且，这两组进入与退出，是相互独立互不干扰的。

为了证实上面结论，增加一个ThirdActivity，并在SecondActivity中添加一个按键用于启动它。同时，为了进行区分，SecondActivity启动ThirdActivity的退出过渡，我们设置为另一个默认效果Slide，方向为向左。

```java
	getWindow().setExitTransition(new Slide(Gravity.LEFT));
    startActivity(new Intent(SecondActivity.this, ThirdActivity.class),
      ActivityOptions.makeSceneTransitionAnimation(SecondActivity.this).toBundle());
```

启动和退出Activity顺序为

`1.MainActivity - 2.SecondActivity - 3.ThirdActivity - 4.SecondActivity - 5.MainActivity `

此过程的过渡效果如下

![Two enters and exits.gif](/assets/post_resources/02.gif)

可以看到，正如上述结论，SecondActivity被启动时，采用Explode效果进入，在最后退出时，也会是此效果退出（即2的进入、4的退出为一组Explode）；当SecondActivity启动ThirdActivity时，退出到后台，执行了Slide效果，退出ThirdActivity使其重新回到前台时，同样执行了Slide效果进入（即2的退出、4的进入为一组Slide）。SecondActivity在整个过程中有两组进入与退出，且相互独立。

## 共享元素过渡

在两个Activity中，如果有两个或者多个元素控件相同（至少外表看起来如此），那么，我们就称这两个或者多个控件是共享元素。还是用例子来说明比较清晰一点。

在SecondActivity中，增加一个TextView，文本内容是“World”
```xml
<TextView
        android:id="@+id/text"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:gravity="center"
        android:text="World"
        android:textSize="32sp"
        android:layout_gravity="bottom|left"
        android:layout_marginLeft="16dp"
        android:layout_marginBottom="16dp"
        android:transitionName="test_text" />
```
同样，在ThirdActivity中，也增加一个TextView，文本内容也是“World”
```xml
<TextView
        android:id="@+id/text"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:gravity="center"
        android:text="World"
        android:textSize="32sp"
        android:layout_gravity="center"
        android:transitionName="test_text" />
```
这样一来，两个分属不同Activity的文本控件，内容相同，那么我们就认为这两个控件是共享元素。现在我们要做的，就是让这两个Activity相互跳转的时候，共享元素能够神奇地从一个Activity过渡到另一个。

### (1) 文本过渡

以上面的两个TextView作过渡，十分简单
```java
startActivity(new Intent(this, ThirdActivity.class),
      ActivityOptions.makeSceneTransitionAnimation(this, mText, "test_text").toBundle());
```
与进入/退出过渡类似，也调用了*makeSceneTransitionAnimation*方法，不过此处参数略有不同：第二个参数指定共享元素（View类型）；第三个参数为过渡名称。共享元素须设置相同的名称。

无需另加效果，以上调用即可实现共享元素过渡。

![Shared text.gif](/assets/post_resources/03.gif)

是不是感觉很酷炫？

需要注意的是，这里的两个文本元素的控件属性是相同的（字体大小、颜色等），如果不同，原生的效果暂不支持，我们就会看到过渡动画过程中出现突变的问题。这个需要自定义实现，暂时留到以后再作讨论。

### (2) 图片过渡

图片的共享过渡，比起文本来说，原生实现得更好，不同大小的共享图片也可以完美过渡。

类似地，分别在SecondActivity和ThirdActivity中增加一个共享ImageView。

*SecondActivity中的图片*
```xml
<ImageView
        android:id="@+id/image"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginRight="16dp"
        android:layout_marginBottom="16dp"
        android:layout_gravity="bottom|right"
        android:src="@mipmap/ic_launcher"/>
```

*ThirdActivity中的图片*
```xml
<ImageView
        android:id="@+id/image"
        android:layout_width="120dp"
        android:layout_height="120dp"
        android:layout_gravity="bottom"
        android:layout_marginLeft="80dp"
        android:layout_marginBottom="80dp"
        android:scaleType="centerCrop"
        android:transitionName="test_image"
        android:src="@mipmap/ic_launcher"/>
```
两个界面中，图片的位置和大小，都有差别。

![shared image.gif](/assets/post_resources/04.gif)

成功实现图片过渡，并且还有图片尺寸的变化。

不知道细心的你有没有发现，前面的两个过渡图片控件，并没有同时指定过渡名称（transitionName），但是依然成功的实现了过渡效果。经实验发现，只要过渡目标控件有指定过渡名称，就没有问题了。至于为什么，暂不清楚。

### (3) 共享元素组过渡

如果我们需要将多个不同的共享元素同时进行过渡，也是允许的，只需要传入Pair<View, String>类型的数组即可，其中，View是共享元素，String是相应的过渡名称。

我们把前面的文本过渡、图片过渡放到一起，就是一个简单的共享元素组过渡。

```java
options = ActivityOptions.makeSceneTransitionAnimation(this,
                  new Pair<View, String>(mImage, "test_image"), new Pair<View, String>(mText, "test_text"));
```

![shared image and text.gif](/assets/post_resources/05.gif)

# 小结

Android动画系统包含了太多有趣且实用的功能，但鉴于目前广大产品设计者和开发们没有深入挖掘，白白浪费掉了现成的好东西。Activity过渡就是其中一个。鄙人在此只作了一个简单的学习总结，抛夸引玉。如有不当和谬误之处，烦请各位看官不吝赐教啊。

附上[源码](https://github.com/Dale-Android/ActivityTransition)，仅供参考。
