---
layout: post
title: "WorkManager (3) —— 取消和监听任务"
date: "2021-01-21 10:20:36 +0800"
description: 
img: 
tags: [Android, WorkManager]
mathjax: true
---

上一篇说到，周期性延时任务，实际被非延时的周期任务给干扰了。这是因为，**任务一直是添加到系统的，应用未启动的时候，不会有，但是当应用重新启动过后，如果条件满足，之前添加的周期性任务就会执行**。

既然如此，是时候来学习下如何取消任务了。

## 取消任务

`WorkManager`里共有四个取消任务接口：

```java
/**
 * Cancels work with the given id if it isn't finished.  Note that cancellation is a best-effort
 * policy and work that is already executing may continue to run.  Upon cancellation,
 * {@link ListenableWorker#onStopped()} will be invoked for any affected workers.
 *
 * @param id The id of the work
 * @return An {@link Operation} that can be used to determine when the cancelWorkById has
 * completed
 */
public abstract @NonNull Operation cancelWorkById(@NonNull UUID id);

/**
 * Cancels all unfinished work with the given tag.  Note that cancellation is a best-effort
 * policy and work that is already executing may continue to run.  Upon cancellation,
 * {@link ListenableWorker#onStopped()} will be invoked for any affected workers.
 *
 * @param tag The tag used to identify the work
 * @return An {@link Operation} that can be used to determine when the cancelAllWorkByTag has
 * completed
 */
public abstract @NonNull Operation cancelAllWorkByTag(@NonNull String tag);

/**
 * Cancels all unfinished work in the work chain with the given name.  Note that cancellation is
 * a best-effort policy and work that is already executing may continue to run.  Upon
 * cancellation, {@link ListenableWorker#onStopped()} will be invoked for any affected workers.
 *
 * @param uniqueWorkName The unique name used to identify the chain of work
 * @return An {@link Operation} that can be used to determine when the cancelUniqueWork has
 * completed
 */
public abstract @NonNull Operation cancelUniqueWork(@NonNull String uniqueWorkName);

/**
 * Cancels all unfinished work.  <b>Use this method with extreme caution!</b>  By invoking it,
 * you will potentially affect other modules or libraries in your codebase.  It is strongly
 * recommended that you use one of the other cancellation methods at your disposal.
 * <p>
 * Upon cancellation, {@link ListenableWorker#onStopped()} will be invoked for any affected
 * workers.
 *
 * @return An {@link Operation} that can be used to determine when the cancelAllWork has
 * completed
 */
public abstract @NonNull Operation cancelAllWork();
```

1. 第一个方法，需要Worker的id作为参数
2. 第二个方法，需要Worker的tag标记作为参数
3. （Work chain？？还不懂，先不管）
4. 第四个方法，不用参数，取消所有任务


### 取消所有

先来个简单的，取消所有任务（点击启动后，迅速点击取消所有）：

```kotlin
Log.d(TAG, "cancel all")
WorkManager.getInstance(applicationContext).cancelAllWork()
```

执行结果：

    2021-01-13 15:15:25.865 32686-32686/com.jacee.examples.workmanager D/JTest: enqueue periodic on 2 1610522125865
    2021-01-13 15:15:25.938 32686-32723/com.jacee.examples.workmanager D/JTest: doWork on 6774 started - 1610522125938
    2021-01-13 15:15:26.900 32686-32686/com.jacee.examples.workmanager D/JTest: cancel all
    2021-01-13 15:15:28.941 32686-32723/com.jacee.examples.workmanager D/JTest: doWork on 6774 ended - 1610522128941

日志中看到，虽然调用了cancel，但是当前任务的“ended”还是打出来了。这是因为，**已经执行的任务将继续执行，不受cancel影响，cancel的目标是后续的周期任务**。

同时，因为调用了`cancelAllWork`，之前测试添加的任务项也一并消除了。

### 通过id

Worker的id从哪儿来呢？构造的时候任务时候，只生成了一个WorkerRequest对象，果然它有一个获取id的方法：

```java
/**
 * Gets the unique identifier associated with this unit of work.
 *
 * @return The identifier for this unit of work
 */
public @NonNull UUID getId() {
    return mId;
}
```

同样点击启动后，迅速取消：

```kotlin
binding.repeatStop.tag = request
WorkManager.getInstance(applicationContext).enqueue(request)
// ......
(binding.repeatStop.tag as? PeriodicWorkRequest)?.let { request ->
    Log.d(TAG, "cancel periodic: ${request.id}")
    WorkManager.getInstance(applicationContext).cancelWorkById(request.id)
}
```

执行结果:

    2021-01-13 15:36:33.962 32686-32686/com.jacee.examples.workmanager D/JTest: enqueue periodic on 2 1610523393962
    2021-01-13 15:36:34.021 32686-1457/com.jacee.examples.workmanager D/JTest: doWork on 6777 started - 1610523394021
    2021-01-13 15:36:34.929 32686-32686/com.jacee.examples.workmanager D/JTest: cancel periodic: 893c86c1-40e7-4850-9d9d-689a45601467
    2021-01-13 15:36:37.024 32686-1457/com.jacee.examples.workmanager D/JTest: doWork on 6777 ended - 1610523397024


### 通过tag

通过tag取消任务，与通过id类似，不同之处在于：**id是reqeuest构造的时候，自动生成的；但tag则需要自行添加**

```kotlin
// 添加tag
val request = if (android.os.Build.VERSION.SDK_INT >= android.os.Build.VERSION_CODES.O) {
    PeriodicWorkRequestBuilder<DelayWorker>(Duration.ofMillis(PeriodicWorkRequest.MIN_PERIODIC_INTERVAL_MILLIS))
        .addTag("test_periodic")
        .build()
} else {
    PeriodicWorkRequestBuilder<DelayWorker>(PeriodicWorkRequest.MIN_PERIODIC_INTERVAL_MILLIS, TimeUnit.MILLISECONDS)
        .addTag("test_periodic")
        .build()
}
```

取消任务：

```kotlin
Log.d(TAG, "cancel periodic by tag")
WorkManager.getInstance(applicationContext).cancelAllWorkByTag("test_periodic")
```

执行结果：

    2021-01-13 15:42:31.044 1790-1790/com.jacee.examples.workmanager D/JTest: enqueue periodic on 2 1610523751044
    2021-01-13 15:42:31.095 1790-1890/com.jacee.examples.workmanager D/JTest: doWork on 6790 started - 1610523751095
    2021-01-13 15:42:32.107 1790-1790/com.jacee.examples.workmanager D/JTest: cancel periodic by tag
    2021-01-13 15:42:34.096 1790-1890/com.jacee.examples.workmanager D/JTest: doWork on 6790 ended - 1610523754096



## 任务信息监听

有没有发现，前面取消任务的各种操作，都只是日志显示调用了cancel，并不能证明真的成功cancel任务了。除非真的坐等15分钟，看有没有执行下一次任务……（hmmmm...当然，本人写这些话的时候，确实等了15分钟+，也证明确实cancel成功了）。

所以问题来了，有没有办法监听任务的实际状态呢？当然有啊！秘密就在`WorkManager`里：

```java
/**
 * Gets a {@link LiveData} of the {@link WorkInfo} for a given work id.
 *
 * @param id The id of the work
 * @return A {@link LiveData} of the {@link WorkInfo} associated with {@code id}; note that
 *         this {@link WorkInfo} may be {@code null} if {@code id} is not known to
 *         WorkManager.
 */
public abstract @NonNull LiveData<WorkInfo> getWorkInfoByIdLiveData(@NonNull UUID id);
```

通过id，可以获取一个LiveData，数据类型是`WorkInfo`，来看看它是个什么货

```java
/**
 * Information about a particular {@link WorkRequest} containing the id of the WorkRequest, its
 * current {@link State}, output, tags, and run attempt count.  Note that output is only available
 * for the terminal states ({@link State#SUCCEEDED} and {@link State#FAILED}).
 */

public final class WorkInfo {

    private @NonNull UUID mId;
    private @NonNull State mState;
    private @NonNull Data mOutputData;
    private @NonNull Set<String> mTags;
    private @NonNull Data mProgress;
    private int mRunAttemptCount;

    // .....

    /**
     * The current lifecycle state of a {@link WorkRequest}.
     */
    public enum State {

        /**
         * Used to indicate that the {@link WorkRequest} is enqueued and eligible to run when its
         * {@link Constraints} are met and resources are available.
         */
        ENQUEUED,

        /**
         * Used to indicate that the {@link WorkRequest} is currently being executed.
         */
        RUNNING,

        /**
         * Used to indicate that the {@link WorkRequest} has completed in a successful state.  Note
         * that {@link PeriodicWorkRequest}s will never enter this state (they will simply go back
         * to {@link #ENQUEUED} and be eligible to run again).
         */
        SUCCEEDED,

        /**
         * Used to indicate that the {@link WorkRequest} has completed in a failure state.  All
         * dependent work will also be marked as {@code #FAILED} and will never run.
         */
        FAILED,

        /**
         * Used to indicate that the {@link WorkRequest} is currently blocked because its
         * prerequisites haven't finished successfully.
         */
        BLOCKED,

        /**
         * Used to indicate that the {@link WorkRequest} has been cancelled and will not execute.
         * All dependent work will also be marked as {@code #CANCELLED} and will not run.
         */
        CANCELLED;

        /**
         * Returns {@code true} if this State is considered finished.
         *
         * @return {@code true} for {@link #SUCCEEDED}, {@link #FAILED}, and * {@link #CANCELLED}
         *         states
         */
        public boolean isFinished() {
            return (this == SUCCEEDED || this == FAILED || this == CANCELLED);
        }
    }

```

`WorkInfo`包含了request的id信息，还有一个`WorkInfo.State`枚举，用于表示request的生命周期。

既然是LiveData，自然就可以observe监听了：

```kotlin
// 在enqueue前，添加observe以监听状态
WorkManager.getInstance(applicationContext).getWorkInfoByIdLiveData(request.id).observe({ lifecycle }) {
    Log.d(TAG, "periodic: ${it.id}: ${it.state}")
}
WorkManager.getInstance(applicationContext).enqueue(request)
```


**启动周期任务 -> 等一次执行完成 -> 取消任务**，来看看执行结果：

    2021-01-13 16:03:28.595 2735-2735/com.jacee.examples.workmanager D/JTest: enqueue periodic on 2 1610525008595
    2021-01-13 16:03:28.640 2735-2834/com.jacee.examples.workmanager D/JTest: doWork on 6808 started - 1610525008640
    2021-01-13 16:03:28.647 2735-2735/com.jacee.examples.workmanager D/JTest: periodic: 31f121d5-2bfc-4370-b1a7-8a1cb4b98329: RUNNING
    2021-01-13 16:03:31.645 2735-2834/com.jacee.examples.workmanager D/JTest: doWork on 6808 ended - 1610525011644
    2021-01-13 16:03:31.720 2735-2735/com.jacee.examples.workmanager D/JTest: periodic: 31f121d5-2bfc-4370-b1a7-8a1cb4b98329: ENQUEUED
    2021-01-13 16:03:34.986 2735-2735/com.jacee.examples.workmanager D/JTest: cancel periodic: 31f121d5-2bfc-4370-b1a7-8a1cb4b98329
    2021-01-13 16:03:35.025 2735-2735/com.jacee.examples.workmanager D/JTest: periodic: 31f121d5-2bfc-4370-b1a7-8a1cb4b98329: CANCELLED


可以看到，相应的WorkRequest状态的变化为：**RUNNING -> ENQUEUED -> CANCELLED**

如果在执行完就cancel呢：

    2021-01-13 16:12:55.207 2735-2735/com.jacee.examples.workmanager D/JTest: enqueue periodic on 2 1610525575207
    2021-01-13 16:12:55.249 2735-2983/com.jacee.examples.workmanager D/JTest: doWork on 6809 started - 1610525575249
    2021-01-13 16:12:55.262 2735-2735/com.jacee.examples.workmanager D/JTest: periodic: 6b25f204-99dd-485a-8bcf-dc76d2a51d69: RUNNING
    2021-01-13 16:12:56.101 2735-2735/com.jacee.examples.workmanager D/JTest: cancel periodic: 6b25f204-99dd-485a-8bcf-dc76d2a51d69
    2021-01-13 16:12:56.174 2735-2735/com.jacee.examples.workmanager D/JTest: periodic: 6b25f204-99dd-485a-8bcf-dc76d2a51d69: CANCELLED
    2021-01-13 16:12:58.253 2735-2983/com.jacee.examples.workmanager D/JTest: doWork on 6809 ended - 1610525578253

状态的变化为：**RUNNING -> CANCELLED**

因为任务取消，当然不用再次添加到下一个周期了，所以没有**ENQUEUED**。即便当次任务执行完成，也并没有出现**SUCCEEDED**状态，因为**周期任务是不可能达到SUCCEEDED状态的**。顺便来对比下OneTimeRequest的执行状态变化。

```kotlin
WorkManager.getInstance(applicationContext).enqueue(request.also {
    WorkManager.getInstance(applicationContext).getWorkInfoByIdLiveData(it.id).observe({lifecycle}) { info ->
        Log.d(TAG, "periodic: ${info.id}: ${info.state}")
    }
})
```

结果：

    2021-01-13 16:17:51.979 3342-3342/com.jacee.examples.workmanager D/JTest: enqueue on 2 1610525871979
    2021-01-13 16:17:52.073 3342-3410/com.jacee.examples.workmanager D/JTest: doWork on 6814 started - 1610525872073
    2021-01-13 16:17:52.079 3342-3342/com.jacee.examples.workmanager D/JTest: periodic: 0e698e3b-6e95-41c7-bc22-aa7866b086ac: RUNNING
    2021-01-13 16:17:55.074 3342-3410/com.jacee.examples.workmanager D/JTest: doWork on 6814 ended - 1610525875074
    2021-01-13 16:17:55.121 3342-3342/com.jacee.examples.workmanager D/JTest: periodic: 0e698e3b-6e95-41c7-bc22-aa7866b086ac: SUCCEEDED

如前所料，状态变化为：**RUNNING -> SUCCEEDED**


### 实验

前面有说到：应用启动时，如有历史添加的Worker，将重新添加回来，满足条件就会继续执行。下面验证一下。

首先，在应用启动时候就添加好监听，同时任务项里也打印出id。

```kotlin

// onCreate里添加
override fun onCreate(savedInstanceState: Bundle?) {
    // ....
    with(getId()) {
        if (isNotEmpty()) {
            WorkManager.getInstance(applicationContext)
                .getWorkInfoByIdLiveData(UUID.fromString(this)).observe({ lifecycle }) {
                    Log.d(TAG, "init periodic: ${it.id}: ${it.state}")
                }
        }
    }
}

private fun saveId(id: String) {
    getSharedPreferences("data", MODE_PRIVATE).edit().putString("id", id).apply()
}

private fun getId(): String {
    return getSharedPreferences("data", MODE_PRIVATE).getString("id", "") ?: ""
}
```

任务启动时，存好id

```kotlin
saveId(request.id)
WorkManager.getInstance(applicationContext).getWorkInfoByIdLiveData(request.id).observe({ lifecycle }) {
    Log.d(TAG, "periodic: ${it.id}: ${it.state}")
}
WorkManager.getInstance(applicationContext).enqueue(request)
```


1. 启动定时任务：


        2021-01-13 17:20:07.081 5586-5586/com.jacee.examples.workmanager D/JTest: enqueue periodic on 2 1610529607081
        2021-01-13 17:20:07.119 5586-5630/com.jacee.examples.workmanager D/JTest: doWork on 6872 started - 1610529607119  -> 5c52487f-e0fd-4cad-ad8d-7cd2be6b2995
        2021-01-13 17:20:07.128 5586-5586/com.jacee.examples.workmanager D/JTest: periodic: 5c52487f-e0fd-4cad-ad8d-7cd2be6b2995: RUNNING
        2021-01-13 17:20:10.124 5586-5630/com.jacee.examples.workmanager D/JTest: doWork on 6872 ended - 1610529610124
        2021-01-13 17:20:10.203 5586-5586/com.jacee.examples.workmanager D/JTest: periodic: 5c52487f-e0fd-4cad-ad8d-7cd2be6b2995: ENQUEUED


2. 杀掉应用，等待15分钟以上（15分是该任务的周期）
3. 再次启动应用，观察是否有历史任务继续执行

结果：

    2021-01-13 17:41:27.624 7289-7325/com.jacee.examples.workmanager D/JTest: doWork on 6935 started - 1610530887624  -> **5c52487f-e0fd-4cad-ad8d-7cd2be6b2995**
    2021-01-13 17:41:27.644 7289-7289/com.jacee.examples.workmanager D/JTest: init periodic: 5c52487f-e0fd-4cad-ad8d-7cd2be6b2995: RUNNING
    2021-01-13 17:41:30.630 7289-7325/com.jacee.examples.workmanager D/JTest: doWork on 6935 ended - 1610530890630
    2021-01-13 17:41:30.708 7289-7289/com.jacee.examples.workmanager D/JTest: init periodic: 5c52487f-e0fd-4cad-ad8d-7cd2be6b2995: ENQUEUED

可以看到，确实是id为**5c52487f-e0fd-4cad-ad8d-7cd2be6b2995**的任务，又执行了。因为已经隔了20分钟，所以任务是立即执行的，而且周期任务将`ENQUEUED`，继续进行。

## 小结

至此，WorkManager的任务添加、取消以及任务状态的监听，算是基本介绍完了，当然有些细节是没有深入的。

前面谈取消任务的时候，有一个方法`cancelUniqueWork`有涉及到**chain work**的概念，是什么呢？下回再来学习吧