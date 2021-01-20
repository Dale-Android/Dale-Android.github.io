---
layout: post
title: "WorkManager (2) —— 周期性任务"
date: "2021-01-20 18:12:25 +0800"
description: 
img: 
tags: [Android, WorkManager]
mathjax: true
---

上一回，我们已经简单地实现了一个单次任务，即通过`OneTimeWorkRequest`构造的任务请求。今天，来试试一个周期性任务请求：`PeriodicWorkRequest`

## 周期性任务

为简单起见，延用之前的**Worker**任务就行了。

下面来构造周期性任务的request。同样地，使用kotlin扩展builder来构造：

```kotlin
val request = if (android.os.Build.VERSION.SDK_INT >= android.os.Build.VERSION_CODES.O) {
    PeriodicWorkRequestBuilder<DelayWorker>(Duration.ofMillis(1000L)).build()
} else {
    PeriodicWorkRequestBuilder<DelayWorker>(1000L, TimeUnit.MILLISECONDS).build()
}
```

可以看出，任务要求是: **每间隔1秒钟，执行一次任务**

接着，执行此任务：

```kotlin
Log.d(TAG, "enqueue periodic on ${Thread.currentThread().id} ${System.currentTimeMillis()}")
WorkManager.getInstance(applicationContext).enqueue(request)
```

和单次任务是一模一样的。

### 执行结果

直接来看看执行结果：

    2021-01-12 17:39:18.895 15044-15044/com.jacee.examples.workmanager D/JTest: enqueue periodic on 2 1610444358895
    2021-01-12 17:39:18.978 15044-15277/com.jacee.examples.workmanager D/JTest: doWork on 6222 started - 1610444358978
    2021-01-12 17:39:21.982 15044-15277/com.jacee.examples.workmanager D/JTest: doWork on 6222 ended - 1610444361982
    2021-01-12 17:54:22.047 15044-15643/com.jacee.examples.workmanager D/JTest: doWork on 6224 started - 1610445262047
    2021-01-12 17:54:25.048 15044-15643/com.jacee.examples.workmanager D/JTest: doWork on 6224 ended - 1610445265048
    2021-01-12 18:09:25.128 15044-16082/com.jacee.examples.workmanager D/JTest: doWork on 6225 started - 1610446165128
    2021-01-12 18:09:28.130 15044-16082/com.jacee.examples.workmanager D/JTest: doWork on 6225 ended - 1610446168130

任务发出后，马上就执行了一次，但是，**我们想要的一秒后再次执行，并没有成功！** 为什么呢？第二次、第三次执行都相隔了15分钟？!

### 分析

官方文档是时候出来了：

> A WorkRequest for repeating work. This work executes multiple times until it is cancelled, with the **first execution happening immediately** or as soon as the given Constraints are met. The next execution will happen during the period interval; note that execution **may be delayed because WorkManager is subject to OS battery optimizations, such as doze mode**.
You can control when the work executes in the period interval more exactly - see PeriodicWorkRequest.Builder for documentation on flexIntervals.
**Periodic work has a minimum interval of 15 minutes.**
Periodic work is intended for use cases where you want a fairly consistent delay between consecutive runs, and you are willing to accept inexactness due to battery optimizations and doze mode. Please note that if your periodic work has constraints, it will not execute until the constraints are met, even if the delay between periods has been met.

要点出炉：

- 第一次执行确实是立即的
- 执行可能被延迟，受系统耗电控制逻辑影响，以及doze模式等
- 周期任务最少是15分钟


所以，我们的1秒钟，将被系统忽略，最终使用了最少的15分钟。这其实和AlarmManager的规则是一样的，包括电池及doze的影响，都是延续AlarmManager。


## 带延时参数的周期任务

周期性任务还可以传入一个参数，以控制任务实际的执行时间。

```java
public final class PeriodicWorkRequest extends WorkRequest {

    // ......

    public static final class Builder extends WorkRequest.Builder<Builder, PeriodicWorkRequest> {

        // ......

        @RequiresApi(26)
        public Builder(
                @NonNull Class<? extends ListenableWorker> workerClass,
                @NonNull Duration repeatInterval,
                @NonNull Duration flexInterval) {
            super(workerClass);
            mWorkSpec.setPeriodic(repeatInterval.toMillis(), flexInterval.toMillis());
        }
    }
    // ......
}

```

其中，builder比普通的调用，增加了**flexInterval**参数，什么意思呢？来看看注释


```html
Creates a PeriodicWorkRequest to run periodically once within the flex period of every interval period. See diagram below. The flex period begins at repeatInterval - flexInterval to the end of the interval. The repeat interval must be greater than or equal to MIN_PERIODIC_INTERVAL_MILLIS and the flex interval must be greater than or equal to MIN_PERIODIC_FLEX_MILLIS.
           [     before flex     |     flex     ][     before flex     |     flex     ]...
           [   cannot run work   | can run work ][   cannot run work   | can run work ]...
           \____________________________________/\____________________________________/...
                          interval 1                            interval 2             ...(repeat)
```

首先，任务的执行时机，是**每个周期的的flex时间区间**，执行一次。其次，**flex的区间排在整个周期的结尾**。**周期的最小值是15分钟，flex的最小值是5分钟**。

也就是说，带flex参数的任务发出后，如果flex小于周期，那首次任务执行时间就有机会做到延时效果（上面的“before flex”就是这个实际的延时时间）。

构造一个周期为30分钟，flex为20分钟的请求并发出任务：

```kotlin
        val request = if (android.os.Build.VERSION.SDK_INT >= android.os.Build.VERSION_CODES.O) {
            PeriodicWorkRequestBuilder<DelayWorker>(Duration.ofMinutes(30), Duration.ofMinutes(20)).build()
        } else {
            PeriodicWorkRequestBuilder<DelayWorker>(30, TimeUnit.MINUTES, 20, TimeUnit.MINUTES).build()
        }
        Log.d(TAG, "enqueue periodic with flex on ${Thread.currentThread().id} ${System.currentTimeMillis()}")
        WorkManager.getInstance(applicationContext).enqueue(request)
```

结果如下：

    2021-01-13 10:47:57.458 25041-25041/com.jacee.examples.workmanager D/JTest: enqueue periodic with flex on 2 1610506077458
    2021-01-13 10:58:09.925 25041-25446/com.jacee.examples.workmanager D/JTest: doWork on 6587 started - 1610506689925
    2021-01-13 10:58:12.930 25041-25446/com.jacee.examples.workmanager D/JTest: doWork on 6587 ended - 1610506692929
    2021-01-13 11:28:13.082 25041-25259/com.jacee.examples.workmanager D/JTest: doWork on 6586 started - 1610508493082
    2021-01-13 11:28:16.085 25041-25259/com.jacee.examples.workmanager D/JTest: doWork on 6586 ended - 1610508496085


关注下时间就可知道事件序列：`10:47发出任务 -> 10分钟(30 - 20)后，执行第一次任务 -> 30分钟（第一周期20分钟flex + 第二周期10分钟延时）后，执行第二次任务`。其实总的看来，就是**第一个任务延时（interval - flex），之后每间隔interval执行一次**。

## 小结

周期任务的介绍到此结束。但其实还有一个小问题 —— 带延时任务的输出结果实际是被干扰了的，实际如下：

    **2021-01-13 10:46:15.216 25041-25259/com.jacee.examples.workmanager D/JTest: doWork on 6586 started - 1610505975216**
    **2021-01-13 10:46:18.221 25041-25259/com.jacee.examples.workmanager D/JTest: doWork on 6586 ended - 1610505978220**
    2021-01-13 10:47:57.458 25041-25041/com.jacee.examples.workmanager D/JTest: enqueue periodic with flex on 2 1610506077458
    2021-01-13 10:58:09.925 25041-25446/com.jacee.examples.workmanager D/JTest: doWork on 6587 started - 1610506689925
    2021-01-13 10:58:12.930 25041-25446/com.jacee.examples.workmanager D/JTest: doWork on 6587 ended - 1610506692929
    **2021-01-13 11:01:18.379 25041-25485/com.jacee.examples.workmanager D/JTest: doWork on 6589 started - 1610506878379**
    **2021-01-13 11:01:21.381 25041-25485/com.jacee.examples.workmanager D/JTest: doWork on 6589 ended - 1610506881380**
    **2021-01-13 11:16:21.537 25041-25661/com.jacee.examples.workmanager D/JTest: doWork on 6590 started - 1610507781537**
    **2021-01-13 11:16:24.541 25041-25661/com.jacee.examples.workmanager D/JTest: doWork on 6590 ended - 1610507784541**
    2021-01-13 11:28:13.082 25041-25259/com.jacee.examples.workmanager D/JTest: doWork on 6586 started - 1610508493082
    2021-01-13 11:28:16.085 25041-25259/com.jacee.examples.workmanager D/JTest: doWork on 6586 ended - 1610508496085
    **2021-01-13 11:31:24.706 25041-25446/com.jacee.examples.workmanager D/JTest: doWork on 6587 started - 1610508684706**
    **2021-01-13 11:31:27.709 25041-25446/com.jacee.examples.workmanager D/JTest: doWork on 6587 ended - 1610508687709**


中间夹杂着上一次创建的周期任务的执行日志 —— 原来上一个一般性周期任务还在执行！那么，我们自然应该想到：周期性任务如何取消呢？下回分解……