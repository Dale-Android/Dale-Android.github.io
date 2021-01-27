---
layout: post
title: "WorkManager (6) —— Constraints、Operation、Result"
date: "2021-01-27 12:04:59 +0800"
description: 
img: 
tags: [Android, WorkManager]
mathjax: true
---

今天的主角是我们还没关注到的几个`WorkManager`的功能和细节。

## Constraints

WorkManager里面有一个`Constraints` —— 姑且在这儿称它为**限制**，它的用处就是：**保证一定的限制条件满足后，才执行任务请求**。

在任务请求的builder里，有一个方法就用于设置限制条件，`WorkRequest.Builder.setConstraints(Constraints)`，即`WorkRequest`运行的前提条件。


```kotlin
// A specification of the requirements that need to be met before a {@link WorkRequest} can run.
public final class Constraints {

    /**
     * Represents a Constraints object with no requirements.
     */
    public static final Constraints NONE = new Constraints.Builder().build();

    // NOTE: this is effectively a @NonNull, but changing the annotation would result in a really
    // annoying database migration that we can deal with later.
    @ColumnInfo(name = "required_network_type")
    private NetworkType mRequiredNetworkType = NOT_REQUIRED;

    @ColumnInfo(name = "requires_charging")
    private boolean mRequiresCharging;

    @ColumnInfo(name = "requires_device_idle")
    private boolean mRequiresDeviceIdle;

    @ColumnInfo(name = "requires_battery_not_low")
    private boolean mRequiresBatteryNotLow;

    @ColumnInfo(name = "requires_storage_not_low")
    private boolean mRequiresStorageNotLow;

    @ColumnInfo(name = "trigger_content_update_delay")
    private long mTriggerContentUpdateDelay = -1;

    @ColumnInfo(name = "trigger_max_content_delay")
    private long  mTriggerMaxContentDelay = -1;

    // NOTE: this is effectively a @NonNull, but changing the annotation would result in a really
    // annoying database migration that we can deal with later.
    @ColumnInfo(name = "content_uri_triggers")
    private ContentUriTriggers mContentUriTriggers = new ContentUriTriggers();

    // .....
}
```

从`Constraints`类的定义中看出，包括***网络情况、充电状态、存储容量***等，都可以作为限制条件。

### 实验

接下来，用网络作为一个限制条件，来实操下。

给DelayWorker添加toast，提醒任务执行：

```kotlin
class DelayWorker(context: Context, workerParams: WorkerParameters) : Worker(context, workerParams) {

    override fun doWork(): Result {
        val name = inputData.getString(ARG_NAME)
        if (inputData.getBoolean(ARG_TOAST_START, false) && !name.isNullOrEmpty()) {
            val mainHandler = Handler(Looper.getMainLooper())
            mainHandler.post {
                Toast.makeText(applicationContext, "$name started!", Toast.LENGTH_SHORT).show()
            }
        }

  // ......

  companion object {
        // .....
        const val ARG_TOAST_START = "start"
    }
}
```

网络限制：
```kotlin
val builder = OneTimeWorkRequestBuilder<DelayWorker>()
        .setInputData(
            Data.Builder()
                .putString(DelayWorker.ARG_NAME, "constraint one")
                .putBoolean(DelayWorker.ARG_TOAST_START, true) // 为更直观，真正运行时，弹出toast提醒
                .build()
        )

// 网络连接作为限制条件
val constraint = Constraints.Builder()
    .setRequiredNetworkType(NetworkType.CONNECTED)
    .build()

Log.d(TAG, "enqueue with constraint")
WorkManager.getInstance(applicationContext).enqueue(builder.setConstraints(constraint).build())
```


(1) 断网启动应用，执行结果：

        2021-01-14 16:37:28.197 25062-25062/com.jacee.examples.workmanager D/JTest: enqueue with constraint

虽然添加了任务（enqueued），但任务并未执行。


(2) 联网后：

        2021-01-14 16:37:36.497 25062-25153/com.jacee.examples.workmanager D/JTest: [constraint one] doWork on 7516 started - 1610613456497  -> 8ec4e62d-ad2a-4160-b9e7-6efb556ff879
        2021-01-14 16:37:39.499 25062-25153/com.jacee.examples.workmanager D/JTest: [constraint one] doWork on 7516 ended - 1610613459499

此时，日志打印显示任务执行，toast也弹出了。


## Operation

到目前为止，我们enqueue的所有任务，都像是“嫁出去的女儿，泼出去的水”，并没有关注到底有没有“嫁”成功 —— 其实enqueue和cancel，都有返回值的啊！

```java
@NonNull
public abstract Operation enqueue(@NonNull List<? extends WorkRequest> requests);

public abstract @NonNull Operation cancelWorkById(@NonNull UUID id);
```

这个返回值`Operation`，就是用来监听WorkManager的执行状态的，包括三种：`SUCCESS`（执行成功）、`IN_PROGRESS`（执行中）和`FAILURE`（执行失败）

```java
public interface Operation {
    /**
     * The lifecycle state of an {@link Operation}.
     */
    abstract class State {
        public static final class SUCCESS extends Operation.State { // ...
        public static final class IN_PROGRESS extends Operation.State { // ...
        public static final class FAILURE extends Operation.State { // ...
    }
}
```

### 实验

下面给单次任务添加一个Operation监听：

```kotlin
WorkManager.getInstance(applicationContext).enqueue(request.also {
    WorkManager.getInstance(applicationContext).getWorkInfoByIdLiveData(it.id).observe({lifecycle}) { info ->
        Log.d(TAG, "onetime: ${info.id}: ${info.state}")
    }
}).also {
    it.state.observe({lifecycle}) { op ->
        Log.d(TAG, "operation: $op")
    }
}
```

结果：

    2021-01-14 17:08:30.256 25920-25920/com.jacee.examples.workmanager D/JTest: enqueue on 2 1610615310256
    2021-01-14 17:08:30.291 25920-25920/com.jacee.examples.workmanager D/JTest: operation: IN_PROGRESS
    2021-01-14 17:08:30.325 25920-25920/com.jacee.examples.workmanager D/JTest: operation: SUCCESS
    2021-01-14 17:08:30.344 25920-26018/com.jacee.examples.workmanager D/JTest: [ONE-TIME] doWork on 7531 started - 1610615310344  -> 6db20040-7212-4bb5-83d4-fa67fdc507ee
    2021-01-14 17:08:30.348 25920-25920/com.jacee.examples.workmanager D/JTest: onetime: 6db20040-7212-4bb5-83d4-fa67fdc507ee: RUNNING
    2021-01-14 17:08:33.346 25920-26018/com.jacee.examples.workmanager D/JTest: [ONE-TIME] doWork on 7531 ended - 1610615313346
    2021-01-14 17:08:33.401 25920-25920/com.jacee.examples.workmanager D/JTest: onetime: 6db20040-7212-4bb5-83d4-fa67fdc507ee: SUCCEEDED


添加任务后，Operation就马在经历了 `IN_PROGRESS -> SUCCESS`，也就是任务添加成功。接下来就是任务本身的执行及其执行状态的变化。

再来看看取消周期任务

```kotlin
(binding.repeatStop.tag as? PeriodicWorkRequest)?.let { request ->
    Log.d(TAG, "cancel periodic: ${request.id}")
    WorkManager.getInstance(applicationContext).cancelWorkById(request.id).also {
        it.state.observe({ lifecycle }) { op ->
            Log.d(TAG, "cancel repeat operation: $op")
        }
    }
}
```

结果：

    2021-01-14 17:34:24.333 27066-27066/com.jacee.examples.workmanager D/JTest: enqueue periodic on 2 1610616864333
    2021-01-14 17:34:24.380 27066-27152/com.jacee.examples.workmanager D/JTest: [null] doWork on 7592 started - 1610616864380  -> c6d87881-cb00-45a9-9102-f07192f62166
    2021-01-14 17:34:24.380 27066-27066/com.jacee.examples.workmanager D/JTest: periodic: c6d87881-cb00-45a9-9102-f07192f62166: ENQUEUED
    2021-01-14 17:34:24.383 27066-27066/com.jacee.examples.workmanager D/JTest: periodic: c6d87881-cb00-45a9-9102-f07192f62166: RUNNING
    2021-01-14 17:34:27.381 27066-27152/com.jacee.examples.workmanager D/JTest: [null] doWork on 7592 ended - 1610616867381
    2021-01-14 17:34:27.445 27066-27066/com.jacee.examples.workmanager D/JTest: periodic: c6d87881-cb00-45a9-9102-f07192f62166: ENQUEUED
    2021-01-14 17:34:32.366 27066-27066/com.jacee.examples.workmanager D/JTest: cancel periodic: c6d87881-cb00-45a9-9102-f07192f62166
    2021-01-14 17:34:32.389 27066-27066/com.jacee.examples.workmanager D/JTest: cancel repeat operation: IN_PROGRESS
    2021-01-14 17:34:32.401 27066-27066/com.jacee.examples.workmanager D/JTest: cancel repeat operation: SUCCESS
    2021-01-14 17:34:32.407 27066-27066/com.jacee.examples.workmanager D/JTest: periodic: c6d87881-cb00-45a9-9102-f07192f62166: CANCELLED


同样地，取消动作 `IN_PROGRESS -> SUCCESS`

## Result

相信，`Result`已经不是一个陌生的东西了，**DelayWorker**任务结束就调用了它。顾名思义，它就是用于标志任务的执行结果。

```java
public abstract class ListenableWorker {

    // ......

    public abstract static class Result {

        // 成功
        @NonNull
        public static Result success() {
            return new Success();
        }

        // 带数据的成功
        @NonNull
        public static Result success(@NonNull Data outputData) {
            return new Success(outputData);
        }

        // 暂时性失败而需要重试
        @NonNull
        public static Result retry() {
            return new Retry();
        }

        // 永久性失败
        @NonNull
        public static Result failure() {
            return new Failure();
        }

        // 带数据的永久性失败
        @NonNull
        public static Result failure(@NonNull Data outputData) {
            return new Failure(outputData);
        }

        public static final class Success extends Result {
            // ......
        }

        public static final class Failure extends Result {
            // ......
        }

        public static final class Retry extends Result {
            // ......
        }
    }
}
```

其中，`Success`、`Failure`和`Retry`都是隐藏的，需要构造时，调用Result的静态方法就行。

在之前讨论链式任务的时候，已经有用到`Failure`了，以此返回的任务，状态为失败FAILED，并丢掉后续的任务。

那么，`Retry`怎么用呢？

### 实验

定义一个可以返回Retry结果类型的Worker：

```kotlin
class RetriedWorker(context: Context, workerParams: WorkerParameters) : Worker(context, workerParams) {

    override fun doWork(): Result {
        Log.d(TAG, "RetriedWorker on ${Thread.currentThread().id} started - ${System.currentTimeMillis()}")
        // emulated work
        Thread.sleep(1_000)
        Log.d(TAG, "RetriedWorker on ${Thread.currentThread().id} ended - ${System.currentTimeMillis()}")
        val first = inputData.getBoolean(ARG_IS_FIRST, false)
        // 根据参数决定是否成功，如果是第一次，就retry；否则success
        return if (first) Result.retry() else Result.success()
    }

    companion object {
        const val ARG_IS_FIRST = "is_first_done"
    }
}
```

启动上述任务：

```kotlin
val request = OneTimeWorkRequestBuilder<RetriedWorker>()
    .setInputData(
        Data.Builder()
            .putBoolean(RetriedWorker.ARG_IS_FIRST, true)
            .build()
    )
    .build()
Log.d(TAG, "enqueue on ${Thread.currentThread().id} ${System.currentTimeMillis()}")
WorkManager.getInstance(applicationContext).enqueue(request.also {
    WorkManager.getInstance(applicationContext).getWorkInfoByIdLiveData(it.id).observe({ lifecycle }) { info ->
        Log.d(TAG, "retry: ${info.id}: ${info.state}")
    }
}).also {
    it.state.observe({ lifecycle }) { op ->
        Log.d(TAG, "retry operation: $op")
    }
}
```

来看看日志打印结果：

    2021-01-22 15:03:16.995 8860-8860/com.jacee.examples.workmanager D/JTest: enqueue on 2 1611298996995
    2021-01-22 15:03:17.008 8860-8860/com.jacee.examples.workmanager D/JTest: retry operation: IN_PROGRESS
    2021-01-22 15:03:17.041 8860-8860/com.jacee.examples.workmanager D/JTest: retry operation: SUCCESS
    2021-01-22 15:03:17.057 8860-9000/com.jacee.examples.workmanager D/JTest: RetriedWorker on 3783 started - 1611298997057
    2021-01-22 15:03:17.067 8860-8860/com.jacee.examples.workmanager D/JTest: retry: d21a350e-aa24-46c0-bc85-2ede5333cf3f: RUNNING
    2021-01-22 15:03:18.061 8860-9000/com.jacee.examples.workmanager D/JTest: RetriedWorker on 3783 ended - 1611298998061
    2021-01-22 15:03:18.141 8860-8860/com.jacee.examples.workmanager D/JTest: retry: d21a350e-aa24-46c0-bc85-2ede5333cf3f: ENQUEUED
    2021-01-22 15:03:48.146 8860-9017/com.jacee.examples.workmanager D/JTest: RetriedWorker on 3785 started - 1611299028146
    2021-01-22 15:03:48.163 8860-8860/com.jacee.examples.workmanager D/JTest: retry: d21a350e-aa24-46c0-bc85-2ede5333cf3f: RUNNING
    2021-01-22 15:03:49.151 8860-9017/com.jacee.examples.workmanager D/JTest: RetriedWorker on 3785 ended - 1611299029150
    2021-01-22 15:03:49.232 8860-8860/com.jacee.examples.workmanager D/JTest: retry: d21a350e-aa24-46c0-bc85-2ede5333cf3f: ENQUEUED
    2021-01-22 15:04:49.255 8860-9616/com.jacee.examples.workmanager D/JTest: RetriedWorker on 3786 started - 1611299089255
    2021-01-22 15:04:49.280 8860-8860/com.jacee.examples.workmanager D/JTest: retry: d21a350e-aa24-46c0-bc85-2ede5333cf3f: RUNNING
    2021-01-22 15:04:50.258 8860-9616/com.jacee.examples.workmanager D/JTest: RetriedWorker on 3786 ended - 1611299090258
    2021-01-22 15:04:50.329 8860-8860/com.jacee.examples.workmanager D/JTest: retry: d21a350e-aa24-46c0-bc85-2ede5333cf3f: ENQUEUED
    2021-01-22 15:06:50.421 8860-9635/com.jacee.examples.workmanager D/JTest: RetriedWorker on 3787 started - 1611299210421
    2021-01-22 15:06:50.434 8860-8860/com.jacee.examples.workmanager D/JTest: retry: d21a350e-aa24-46c0-bc85-2ede5333cf3f: RUNNING
    2021-01-22 15:06:51.425 8860-9635/com.jacee.examples.workmanager D/JTest: RetriedWorker on 3787 ended - 1611299211424
    2021-01-22 15:06:51.511 8860-8860/com.jacee.examples.workmanager D/JTest: retry: d21a350e-aa24-46c0-bc85-2ede5333cf3f: ENQUEUED
    2021-01-22 15:10:51.601 8860-8860/com.jacee.examples.workmanager D/JTest: retry: d21a350e-aa24-46c0-bc85-2ede5333cf3f: RUNNING
    2021-01-22 15:10:52.591 8860-9000/com.jacee.examples.workmanager D/JTest: RetriedWorker on 3783 ended - 1611299452591
    2021-01-22 15:10:52.667 8860-8860/com.jacee.examples.workmanager D/JTest: retry: d21a350e-aa24-46c0-bc85-2ede5333cf3f: ENQUEUED
    2021-01-22 15:18:52.685 8860-9017/com.jacee.examples.workmanager D/JTest: RetriedWorker on 3785 started - 1611299932685
    2021-01-22 15:18:52.701 8860-8860/com.jacee.examples.workmanager D/JTest: retry: d21a350e-aa24-46c0-bc85-2ede5333cf3f: RUNNING
    2021-01-22 15:18:53.687 8860-9017/com.jacee.examples.workmanager D/JTest: RetriedWorker on 3785 ended - 1611299933687
    2021-01-22 15:18:53.764 8860-8860/com.jacee.examples.workmanager D/JTest: retry: d21a350e-aa24-46c0-bc85-2ede5333cf3f: ENQUEUED



看起来，这个任务是没完没了的执行下去了…… 明明是一个onetime任务，结果像搞成了periodic！

下面来详细分析下日志：

    2021-01-22 15:03:16.995 8860-8860/com.jacee.examples.workmanager D/JTest: enqueue on 2 1611298996995
    2021-01-22 15:03:17.008 8860-8860/com.jacee.examples.workmanager D/JTest: retry operation: IN_PROGRESS
    2021-01-22 15:03:17.041 8860-8860/com.jacee.examples.workmanager D/JTest: retry operation: SUCCESS
    2021-01-22 15:03:17.057 8860-9000/com.jacee.examples.workmanager D/JTest: RetriedWorker on 3783 started - 1611298997057
    2021-01-22 15:03:17.067 8860-8860/com.jacee.examples.workmanager D/JTest: retry: d21a350e-aa24-46c0-bc85-2ede5333cf3f: RUNNING
    2021-01-22 15:03:18.061 8860-9000/com.jacee.examples.workmanager D/JTest: RetriedWorker on 3783 ended - 1611298998061
    // 至此，任务结束，返回了retry
    2021-01-22 15:03:18.141 8860-8860/com.jacee.examples.workmanager D/JTest: retry: d21a350e-aa24-46c0-bc85-2ede5333cf3f: ENQUEUED
    // 因为第一次肯定是retry，所以任务又成了等待状态：ENQUEUED
    2021-01-22 15:03:48.146 8860-9017/com.jacee.examples.workmanager D/JTest: RetriedWorker on 3785 started - 1611299028146
    // ENQUEUED状态30秒后，又一次启动任务
    2021-01-22 15:03:48.163 8860-8860/com.jacee.examples.workmanager D/JTest: retry: d21a350e-aa24-46c0-bc85-2ede5333cf3f: RUNNING
    2021-01-22 15:03:49.151 8860-9017/com.jacee.examples.workmanager D/JTest: RetriedWorker on 3785 ended - 1611299029150
    // 第二次任务结束
    2021-01-22 15:03:49.232 8860-8860/com.jacee.examples.workmanager D/JTest: retry: d21a350e-aa24-46c0-bc85-2ede5333cf3f: ENQUEUED
    // 再一次ENQUUEUED
    2021-01-22 15:04:49.255 8860-9616/com.jacee.examples.workmanager D/JTest: RetriedWorker on 3786 started - 1611299089255
    // ENQUEUED状态60秒后，又一次启动任务
    2021-01-22 15:04:49.280 8860-8860/com.jacee.examples.workmanager D/JTest: retry: d21a350e-aa24-46c0-bc85-2ede5333cf3f: RUNNING
    2021-01-22 15:04:50.258 8860-9616/com.jacee.examples.workmanager D/JTest: RetriedWorker on 3786 ended - 1611299090258
    2021-01-22 15:04:50.329 8860-8860/com.jacee.examples.workmanager D/JTest: retry: d21a350e-aa24-46c0-bc85-2ede5333cf3f: ENQUEUED
    2021-01-22 15:06:50.421 8860-9635/com.jacee.examples.workmanager D/JTest: RetriedWorker on 3787 started - 1611299210421
    // ENQUEUED状态120秒后，又一次启动任务
    2021-01-22 15:06:50.434 8860-8860/com.jacee.examples.workmanager D/JTest: retry: d21a350e-aa24-46c0-bc85-2ede5333cf3f: RUNNING
    2021-01-22 15:06:51.425 8860-9635/com.jacee.examples.workmanager D/JTest: RetriedWorker on 3787 ended - 1611299211424
    2021-01-22 15:06:51.511 8860-8860/com.jacee.examples.workmanager D/JTest: retry: d21a350e-aa24-46c0-bc85-2ede5333cf3f: ENQUEUED
    2021-01-22 15:10:51.586 8860-9000/com.jacee.examples.workmanager D/JTest: RetriedWorker on 3783 started - 1611299451586
    // ENQUEUED状态240秒后，又一次启动任务
    2021-01-22 15:10:51.601 8860-8860/com.jacee.examples.workmanager D/JTest: retry: d21a350e-aa24-46c0-bc85-2ede5333cf3f: RUNNING
    2021-01-22 15:10:52.591 8860-9000/com.jacee.examples.workmanager D/JTest: RetriedWorker on 3783 ended - 1611299452591
    2021-01-22 15:10:52.667 8860-8860/com.jacee.examples.workmanager D/JTest: retry: d21a350e-aa24-46c0-bc85-2ede5333cf3f: ENQUEUED
    2021-01-22 15:18:52.685 8860-9017/com.jacee.examples.workmanager D/JTest: RetriedWorker on 3785 started - 1611299932685
    // ENQUEUED状态480秒后，又一次启动任务
    2021-01-22 15:18:52.701 8860-8860/com.jacee.examples.workmanager D/JTest: retry: d21a350e-aa24-46c0-bc85-2ede5333cf3f: RUNNING
    2021-01-22 15:18:53.687 8860-9017/com.jacee.examples.workmanager D/JTest: RetriedWorker on 3785 ended - 1611299933687
    2021-01-22 15:18:53.764 8860-8860/com.jacee.examples.workmanager D/JTest: retry: d21a350e-aa24-46c0-bc85-2ede5333cf3f: ENQUEUED


由上面的分析日志可以得出结论：

1. 预想的第二次任务就是成功Result，没有实现，因为**即使重试执行任务，输入参数都将保留**。毕竟，既然是retry，那环境肯定需要一样。
2. 失败后的retry冷静期等待时间序列变化：`30s -> 60s -> 120s -> 240s -> 480s ...`，**第二次再retry后，后续都是按2倍时间延长**

retry的冷静期间隔时间是怎么来的？来看看源码吧。

### Retry重试执行的源码分析

```java
/**
 * Returns an instance of {@link Result} that can be used to indicate that the work
 * encountered a transient failure and should be retried with backoff specified in
 * {@link WorkRequest.Builder#setBackoffCriteria(BackoffPolicy, long, TimeUnit)}.
 *
 * @return An instance of {@link Result} indicating that the work needs to be retried
 */
@NonNull
public static Result retry() {
    return new Retry();
}
```

从上面注释可以知道，retry的等待时间，受`WorkRequest.Builder.setBackoffCriteria`影响。几个常量值得关注：`DEFAULT_BACKOFF_DELAY_MILLIS`, `MAX_BACKOFF_MILLIS`, `MIN_BACKOFF_MILLIS`

```java
public abstract class WorkRequest {

    /**
     * The default initial backoff time (in milliseconds) for work that has to be retried.
     默认的初始重试时间间隔是30秒
     */
    public static final long DEFAULT_BACKOFF_DELAY_MILLIS = 30000L;

    /**
     * The maximum backoff time (in milliseconds) for work that has to be retried.
     最长的重试时间间隔是5个小时
     */
    @SuppressLint("MinMaxConstant")
    public static final long MAX_BACKOFF_MILLIS = 5 * 60 * 60 * 1000; // 5 hours.

    /**
     * The minimum backoff time for work (in milliseconds) that has to be retried.
     最小重试时间间隔是10秒
     */
    @SuppressLint("MinMaxConstant")
    public static final long MIN_BACKOFF_MILLIS = 10 * 1000; // 10 seconds.

    // .....
    public abstract static class Builder<B extends Builder<?, ?>, W extends WorkRequest> {

        // .....

        /**
         * Sets the backoff policy and backoff delay for the work.  The default values are
         * {@link BackoffPolicy#EXPONENTIAL} and
         * {@value WorkRequest#DEFAULT_BACKOFF_DELAY_MILLIS}, respectively.  {@code backoffDelay}
         * will be clamped between {@link WorkRequest#MIN_BACKOFF_MILLIS} and
         * {@link WorkRequest#MAX_BACKOFF_MILLIS}.
         *
         * @param backoffPolicy The {@link BackoffPolicy} to use when increasing backoff time
         * @param backoffDelay Time to wait before retrying the work in {@code timeUnit} units
         * @param timeUnit The {@link TimeUnit} for {@code backoffDelay}
         * @return The current {@link Builder}
         */
        public final @NonNull B setBackoffCriteria(
                @NonNull BackoffPolicy backoffPolicy,
                long backoffDelay,
                @NonNull TimeUnit timeUnit) {
            mBackoffCriteriaSet = true;
            mWorkSpec.backoffPolicy = backoffPolicy;
            mWorkSpec.setBackoffDelayDuration(timeUnit.toMillis(backoffDelay));
            return getThis();
        }

    // ....
    }
}

// 等待测试，一种是指数型，一种是线性
public enum BackoffPolicy {

    /**
     * Used to indicate that {@link WorkManager} should increase the backoff time exponentially
     */
    EXPONENTIAL,

    /**
     * Used to indicate that {@link WorkManager} should increase the backoff time linearly
     */
    LINEAR
}
```


在实际执行任务时，相关参数存在一个`WorkSpec`对象里：

```java
public final class WorkSpec {
    private static final String TAG = Logger.tagWithPrefix("WorkSpec");
    public static final long SCHEDULE_NOT_REQUESTED_YET = -1;

    @ColumnInfo(name = "id")
    @PrimaryKey
    @NonNull
    public String id;

    @ColumnInfo(name = "state")
    @NonNull
    public WorkInfo.State state = ENQUEUED;

    @ColumnInfo(name = "worker_class_name")
    @NonNull
    public String workerClassName;

    @ColumnInfo(name = "input_merger_class_name")
    public String inputMergerClassName;

    @ColumnInfo(name = "input")
    @NonNull
    public Data input = Data.EMPTY;

    @ColumnInfo(name = "output")
    @NonNull
    public Data output = Data.EMPTY;

    @ColumnInfo(name = "initial_delay")
    public long initialDelay;

    @ColumnInfo(name = "interval_duration")
    public long intervalDuration;

    @ColumnInfo(name = "flex_duration")
    public long flexDuration;

    @Embedded
    @NonNull
    public Constraints constraints = Constraints.NONE;

    @ColumnInfo(name = "run_attempt_count")
    @IntRange(from = 0)
    public int runAttemptCount;

    @ColumnInfo(name = "backoff_policy")
    @NonNull
    public BackoffPolicy backoffPolicy = BackoffPolicy.EXPONENTIAL;  // 策略默认是指数型

    @ColumnInfo(name = "backoff_delay_duration")
    public long backoffDelayDuration = WorkRequest.DEFAULT_BACKOFF_DELAY_MILLIS; // 时间默认是前面提到的30秒

    // ....

    // 计算任务执行时间
    public long calculateNextRunTime() {
        if (isBackedOff()) {
            boolean isLinearBackoff = (backoffPolicy == BackoffPolicy.LINEAR);
            // 这里根据实际的类型来计算delay
            long delay = isLinearBackoff ? (backoffDelayDuration * runAttemptCount)
                    : (long) Math.scalb(backoffDelayDuration, runAttemptCount - 1);
            return periodStartTime + Math.min(WorkRequest.MAX_BACKOFF_MILLIS, delay);
        } else if (isPeriodic()) {
            // .....
        } else {
            // .....
        }
    }

    // ....
}
```

指数函数:

```java
public static float scalb(float f, int scaleFactor) {
    // magnitude of a power of two so large that scaling a finite
    // nonzero value by it would be guaranteed to over or
    // underflow; due to rounding, scaling down takes takes an
    // additional power of two which is reflected here
    final int MAX_SCALE = FloatConsts.MAX_EXPONENT + -FloatConsts.MIN_EXPONENT +
                          FloatConsts.SIGNIFICAND_WIDTH + 1;

    // Make sure scaling factor is in a reasonable range
    scaleFactor = Math.max(Math.min(scaleFactor, MAX_SCALE), -MAX_SCALE);

    /*
     * Since + MAX_SCALE for float fits well within the double
     * exponent range and + float -> double conversion is exact
     * the multiplication below will be exact. Therefore, the
     * rounding that occurs when the double product is cast to
     * float will be the correctly rounded float result.  Since
     * all operations other than the final multiply will be exact,
     * it is not necessary to declare this method strictfp.
     */
    return (float)((double)f*powerOfTwoD(scaleFactor));
}
```

默认重试间隔是30秒，那第一次重试间隔：

$$30 * 2^{(1 - 1)} = 30$$

第二次重试间隔：

$$30 * 2^{(2 - 1)} = 60$$


以此类推。所以，这就是前面日志打印的时间间隔来源。


### 实验2

有了相关的重试逻辑知识后，修改前面的retry任务为线性重试，初始间隔时间为15秒：

```kotlin
val request = OneTimeWorkRequestBuilder<RetriedWorker>()
//            .setInputData(
//                Data.Builder()
//                    .putBoolean(RetriedWorker.ARG_IS_FIRST, true)
//                    .build()
//            )
    // 线性retry策略
    .setBackoffCriteria(BackoffPolicy.LINEAR, 15, TimeUnit.SECONDS)
    .build()
Log.d(TAG, "enqueue on ${Thread.currentThread().id} ${System.currentTimeMillis()}")
WorkManager.getInstance(applicationContext).enqueue(request.also {
    WorkManager.getInstance(applicationContext).getWorkInfoByIdLiveData(it.id).observe({ lifecycle }) { info ->
        Log.d(TAG, "retry: ${info.id}: ${info.state}")
    }
}).also {
    it.state.observe({ lifecycle }) { op ->
        Log.d(TAG, "retry operation: $op")
    }
}
```

任务的retry与否，简单起见，通过时间来判断：

```kotlin
    override fun doWork(): Result {
        Log.d(TAG, "RetriedWorker on ${Thread.currentThread().id} started - ${System.currentTimeMillis()}")
        // emulated work
        Thread.sleep(1_000)
        val time = System.currentTimeMillis()
        Log.d(TAG, "RetriedWorker on ${Thread.currentThread().id} ended - $time")
//        val first = inputData.getBoolean(ARG_IS_FIRST, false)
        return if (time.rem(3) == 0L) Result.retry() else Result.success()
    }
```

来看看结果：

    2021-01-22 17:54:12.177 17307-17307/com.jacee.examples.workmanager D/JTest: enqueue on 2 1611309252177
    2021-01-22 17:54:12.201 17307-17307/com.jacee.examples.workmanager D/JTest: retry operation: IN_PROGRESS
    2021-01-22 17:54:12.209 17307-17307/com.jacee.examples.workmanager D/JTest: retry operation: SUCCESS
    2021-01-22 17:54:12.224 17307-17404/com.jacee.examples.workmanager D/JTest: RetriedWorker on 3922 started - 1611309252224
    2021-01-22 17:54:12.241 17307-17307/com.jacee.examples.workmanager D/JTest: retry: 2ce496f4-0086-441c-b87b-e2f0000f7f53: RUNNING
    2021-01-22 17:54:13.226 17307-17404/com.jacee.examples.workmanager D/JTest: RetriedWorker on 3922 ended - 1611309253226
    2021-01-22 17:54:13.306 17307-17307/com.jacee.examples.workmanager D/JTest: retry: 2ce496f4-0086-441c-b87b-e2f0000f7f53: SUCCEEDED
    // 第一次执行，直接成功了

    2021-01-22 17:54:15.960 17307-17307/com.jacee.examples.workmanager D/JTest: enqueue on 2 1611309255960
    2021-01-22 17:54:15.972 17307-17307/com.jacee.examples.workmanager D/JTest: retry operation: IN_PROGRESS
    2021-01-22 17:54:15.988 17307-17307/com.jacee.examples.workmanager D/JTest: retry operation: SUCCESS
    2021-01-22 17:54:16.006 17307-17406/com.jacee.examples.workmanager D/JTest: RetriedWorker on 3923 started - 1611309256006
    2021-01-22 17:54:16.018 17307-17307/com.jacee.examples.workmanager D/JTest: retry: 6d9cffc7-5900-42dc-b50f-dcef16287f0c: RUNNING
    2021-01-22 17:54:17.009 17307-17406/com.jacee.examples.workmanager D/JTest: RetriedWorker on 3923 ended - 1611309257009
    2021-01-22 17:54:17.089 17307-17307/com.jacee.examples.workmanager D/JTest: retry: 6d9cffc7-5900-42dc-b50f-dcef16287f0c: SUCCEEDED
    // 第二次执行，直接成功了


    2021-01-22 17:54:22.185 17307-17307/com.jacee.examples.workmanager D/JTest: enqueue on 2 1611309262185
    2021-01-22 17:54:22.200 17307-17307/com.jacee.examples.workmanager D/JTest: retry operation: IN_PROGRESS
    2021-01-22 17:54:22.222 17307-17307/com.jacee.examples.workmanager D/JTest: retry operation: SUCCESS
    2021-01-22 17:54:22.233 17307-17385/com.jacee.examples.workmanager D/JTest: RetriedWorker on 3919 started - 1611309262233
    2021-01-22 17:54:22.256 17307-17307/com.jacee.examples.workmanager D/JTest: retry: 21615cd7-aa22-42fa-a296-cf570e208fee: RUNNING
    2021-01-22 17:54:23.239 17307-17385/com.jacee.examples.workmanager D/JTest: RetriedWorker on 3919 ended - 1611309263238
    2021-01-22 17:54:23.336 17307-17307/com.jacee.examples.workmanager D/JTest: retry: 21615cd7-aa22-42fa-a296-cf570e208fee: ENQUEUED
    2021-01-22 17:54:38.292 17307-17397/com.jacee.examples.workmanager D/JTest: RetriedWorker on 3921 started - 1611309278292
    2021-01-22 17:54:38.333 17307-17307/com.jacee.examples.workmanager D/JTest: retry: 21615cd7-aa22-42fa-a296-cf570e208fee: RUNNING
    2021-01-22 17:54:39.294 17307-17397/com.jacee.examples.workmanager D/JTest: RetriedWorker on 3921 ended - 1611309279294
    2021-01-22 17:54:39.381 17307-17307/com.jacee.examples.workmanager D/JTest: retry: 21615cd7-aa22-42fa-a296-cf570e208fee: ENQUEUED
    2021-01-22 17:55:09.373 17307-17404/com.jacee.examples.workmanager D/JTest: RetriedWorker on 3922 started - 1611309309373
    2021-01-22 17:55:09.406 17307-17307/com.jacee.examples.workmanager D/JTest: retry: 21615cd7-aa22-42fa-a296-cf570e208fee: RUNNING
    2021-01-22 17:55:10.378 17307-17404/com.jacee.examples.workmanager D/JTest: RetriedWorker on 3922 ended - 1611309310378
    2021-01-22 17:55:10.461 17307-17307/com.jacee.examples.workmanager D/JTest: retry: 21615cd7-aa22-42fa-a296-cf570e208fee: SUCCEEDED
    // 第三次执行，失败两次后重试成功，时间间隔：15s -> 30s



### 带数据结果

前面有提到，Result有两个方法可以返回带数据的Result：

```java
/**
* Returns an instance of {@link Result} that can be used to indicate that the work
* completed successfully. Any work that depends on this can be executed as long as all of
* its other dependencies and constraints are met.

依赖此任务的OneTimeWorkRequest，可以获取outputData中的数据

*
* @param outputData A {@link Data} object that will be merged into the input Data of any
*                   OneTimeWorkRequest that is dependent on this work
* @return An instance of {@link Result} indicating successful execution of work
*/
@NonNull
public static Result success(@NonNull Data outputData) {
    return new Success(outputData);
}

/**
 * Returns an instance of {@link Result} that can be used to indicate that the work
 * completed with a permanent failure. Any work that depends on this will also be marked as
 * failed and will not be run. <b>If you need child workers to run, you need to use
 * {@link #success()} or {@link #success(Data)}</b>; failure indicates a permanent stoppage
 * of the chain of work.
 *
 * @return An instance of {@link Result} indicating failure when executing work
 */
@NonNull
public static Result failure(@NonNull Data outputData) {
    return new Failure(outputData);
}
```

当然，能获取成功执行的任务的数据，是针对链式任务的`OneTimeWorkRequest`来说的。


#### 成功数据

生成一个携带数据的**DataWorker**

```kotlin
class DataWorker(context: Context, workerParams: WorkerParameters) : Worker(context, workerParams) {

    override fun doWork(): Result {
        // 获取名称
        val name = inputData.getString(ARG_NAME)
        Log.d(TAG, "[$name] doWork on ${Thread.currentThread().id} started - ${System.currentTimeMillis()}  -> $id")
        // emulated work
        Thread.sleep(1_000L)
        Log.d(TAG, "[$name] doWork on ${Thread.currentThread().id} ended - ${System.currentTimeMillis()}  -> $id")
        // 传递一个新的名称给依赖任务(如果有的话)
        return Result.success(Data.Builder()
            .putString(ARG_NAME, "$name ${System.currentTimeMillis()}")
            .build())
    }

    companion object {
        const val ARG_NAME = "name"
    }
}
```

构造连续执行的三个DataWorker任务请求：

```kotlin
WorkManager.getInstance(applicationContext)
    .beginWith(
        OneTimeWorkRequestBuilder<DataWorker>()
            .setInputData(
                Data.Builder()
                    .putString(DataWorker.ARG_NAME, "one")
                    .build()
            )
            .build()
    )
    .then(
        OneTimeWorkRequestBuilder<DataWorker>().build()
    )
    .then(
        OneTimeWorkRequestBuilder<DataWorker>().build()
    )
    .enqueue()
```

结果：

    2021-01-23 10:30:39.322 28199-28305/com.jacee.examples.workmanager D/JTest: [one] doWork on 4343 started - 1611369039322  -> b0d426f1-b1d9-45c8-a660-59b36b4354bd
    2021-01-23 10:30:40.325 28199-28305/com.jacee.examples.workmanager D/JTest: [one] doWork on 4343 ended - 1611369040324  -> b0d426f1-b1d9-45c8-a660-59b36b4354bd
    2021-01-23 10:30:40.386 28199-28316/com.jacee.examples.workmanager D/JTest: [one 1611369040325] doWork on 4345 started - 1611369040386  -> 8722ed96-bf33-430b-a2d3-102259310695
    2021-01-23 10:30:41.390 28199-28316/com.jacee.examples.workmanager D/JTest: [one 1611369040325] doWork on 4345 ended - 1611369041389  -> 8722ed96-bf33-430b-a2d3-102259310695
    2021-01-23 10:30:41.560 28199-28317/com.jacee.examples.workmanager D/JTest: [one 1611369040325 1611369041390] doWork on 4346 started - 1611369041560  -> 7cea6bc1-96c6-4a56-8bdb-8abe92bf962f
    2021-01-23 10:30:42.562 28199-28317/com.jacee.examples.workmanager D/JTest: [one 1611369040325 1611369041390] doWork on 4346 ended - 1611369042562  -> 7cea6bc1-96c6-4a56-8bdb-8abe92bf962f

后续任务都成功的获取到前一个任务通过`Result.success(@NonNull Data outputData)`传入的名称。


#### 失败数据

在任务链那篇讲过，一旦有任务失败，链式任务后续的任务都将fail。所以，带数据的faile，不是在继任任务场景下使用的。那么，带数据的失败结果，如何获取呢？

还记得前面讲任务的状态时关注到的类`WorkInfo`吗？

```java
public final class WorkInfo {

    private @NonNull UUID mId;
    private @NonNull State mState;
    private @NonNull Data mOutputData;
    private @NonNull Set<String> mTags;
    private @NonNull Data mProgress;
    private int mRunAttemptCount;

    // ......
}
```

其中，`mOutputData`字段是不是就是我们要找的结果数据呢？

改装一下FailedWorker，让它可以抛出失败数据：

```kotlin
class FailedWorker(context: Context, workerParams: WorkerParameters) : Worker(context, workerParams) {

    override fun doWork(): Result {
        Log.d(TAG, "FailedWorker on ${Thread.currentThread().id} started - ${System.currentTimeMillis()}")
        // emulated work
        Thread.sleep(1_000)
        Log.d(TAG, "FailedWorker on ${Thread.currentThread().id} ended - ${System.currentTimeMillis()}")
        val withResult = inputData.getBoolean(ARG_RESULT, false)
        // 根据参数判断要不要给失败数据
        return if (withResult) Result.failure(
            Data.Builder()
                .putString("result", "I failed!")
                .build()
        ) else Result.failure()
    }

    companion object {
        const val ARG_RESULT = "with_result"
    }
}
```

构造任务链：

```kotlin
 WorkManager.getInstance(applicationContext)
     .beginWith(
         OneTimeWorkRequestBuilder<DataWorker>()
             .setInputData(
                 Data.Builder()
                     .putString(DataWorker.ARG_NAME, "one")
                     .build()
             )
             .build()
     )
     // 继任FailedWorker，并让其抛数据
     .then(
         OneTimeWorkRequestBuilder<FailedWorker>()
             .setInputData(
                 Data.Builder()
                     .putBoolean(FailedWorker.ARG_RESULT, true)
                     .build()
             )
             .build()
     )
     // 继任Worker，但上一任是fail，自然这个任务是肯定不会执行的
     .then(
         OneTimeWorkRequestBuilder<DataWorker>().build()
     ).apply {
         workInfosLiveData.observe({ lifecycle }) { l ->
             l.forEach {
                 // 打印outputData字段
                 Log.d(TAG, "fail output: ${it.id}, ${it.state}, ${it.outputData}")
             }
             Log.d(TAG, "----------------")
         }
     }
     .enqueue()
```

来看看结果：

    2021-01-23 11:00:21.109 28911-28911/com.jacee.examples.workmanager D/JTest: ----------------
    2021-01-23 11:00:21.123 28911-28995/com.jacee.examples.workmanager D/JTest: [one] doWork on 4355 started - 1611370821123  -> 53caeea5-b774-4913-b493-7d334ad948bf
    2021-01-23 11:00:21.124 28911-28911/com.jacee.examples.workmanager D/JTest: fail output: 0e4a0613-791d-47c3-97f3-002c00532f1f, BLOCKED, Data {}
    2021-01-23 11:00:21.124 28911-28911/com.jacee.examples.workmanager D/JTest: fail output: 53caeea5-b774-4913-b493-7d334ad948bf, ENQUEUED, Data {}
    2021-01-23 11:00:21.124 28911-28911/com.jacee.examples.workmanager D/JTest: fail output: 91b6f3ff-29e4-4397-a26f-fcef1385f108, BLOCKED, Data {}
    2021-01-23 11:00:21.124 28911-28911/com.jacee.examples.workmanager D/JTest: ----------------
    2021-01-23 11:00:21.136 28911-28911/com.jacee.examples.workmanager D/JTest: fail output: 0e4a0613-791d-47c3-97f3-002c00532f1f, BLOCKED, Data {}
    2021-01-23 11:00:21.136 28911-28911/com.jacee.examples.workmanager D/JTest: fail output: 53caeea5-b774-4913-b493-7d334ad948bf, RUNNING, Data {}
    2021-01-23 11:00:21.136 28911-28911/com.jacee.examples.workmanager D/JTest: fail output: 91b6f3ff-29e4-4397-a26f-fcef1385f108, BLOCKED, Data {}
    2021-01-23 11:00:21.136 28911-28911/com.jacee.examples.workmanager D/JTest: ----------------
    2021-01-23 11:00:22.128 28911-28995/com.jacee.examples.workmanager D/JTest: [one] doWork on 4355 ended - 1611370822127  -> 53caeea5-b774-4913-b493-7d334ad948bf
    2021-01-23 11:00:22.240 28911-28911/com.jacee.examples.workmanager D/JTest: fail output: 0e4a0613-791d-47c3-97f3-002c00532f1f, BLOCKED, Data {}
    2021-01-23 11:00:22.240 28911-28911/com.jacee.examples.workmanager D/JTest: fail output: 53caeea5-b774-4913-b493-7d334ad948bf, SUCCEEDED, Data {name : one 1611370822128, }
    2021-01-23 11:00:22.240 28911-28911/com.jacee.examples.workmanager D/JTest: fail output: 91b6f3ff-29e4-4397-a26f-fcef1385f108, ENQUEUED, Data {}
    2021-01-23 11:00:22.240 28911-28911/com.jacee.examples.workmanager D/JTest: ----------------
    // 此时，第一个任务执行成功了，SUCCEEDED，且output也有数据了。第二个任务ENQUEUED
    2021-01-23 11:00:22.241 28911-28997/com.jacee.examples.workmanager D/JTest: FailedWorker on 4357 started - 1611370822241
    2021-01-23 11:00:22.247 28911-28911/com.jacee.examples.workmanager D/JTest: fail output: 0e4a0613-791d-47c3-97f3-002c00532f1f, BLOCKED, Data {}
    2021-01-23 11:00:22.247 28911-28911/com.jacee.examples.workmanager D/JTest: fail output: 53caeea5-b774-4913-b493-7d334ad948bf, SUCCEEDED, Data {name : one 1611370822128, }
    2021-01-23 11:00:22.247 28911-28911/com.jacee.examples.workmanager D/JTest: fail output: 91b6f3ff-29e4-4397-a26f-fcef1385f108, RUNNING, Data {}
    2021-01-23 11:00:22.247 28911-28911/com.jacee.examples.workmanager D/JTest: ----------------
    2021-01-23 11:00:23.247 28911-28997/com.jacee.examples.workmanager D/JTest: FailedWorker on 4357 ended - 1611370823246
    2021-01-23 11:00:23.328 28911-28911/com.jacee.examples.workmanager D/JTest: fail output: 0e4a0613-791d-47c3-97f3-002c00532f1f, FAILED, Data {}
    2021-01-23 11:00:23.329 28911-28911/com.jacee.examples.workmanager D/JTest: fail output: 53caeea5-b774-4913-b493-7d334ad948bf, SUCCEEDED, Data {name : one 1611370822128, }
    2021-01-23 11:00:23.329 28911-28911/com.jacee.examples.workmanager D/JTest: fail output: 91b6f3ff-29e4-4397-a26f-fcef1385f108, FAILED, Data {result : I failed!, }
    2021-01-23 11:00:23.329 28911-28911/com.jacee.examples.workmanager D/JTest: ----------------
    // 第二个任务执行失败，FAILED，同时，带了数据。同时第三个任务直接FAILED


果然，失败数据存入了`WorkInfo`里面。同样地，**成功数据也可以通过这种方法获取**。


## 小结

这次讨论了三个点：`Constraints`, `Operation`, 和 `Result`，它们都是在之前的讨论中忽略掉的、但又很有用的功能。

至此，**WorkManager**比较重要的知识点及其使用，都讲完了。在写这些内容的时候，我有种体会：**写文章更能帮助加深对于知识点的理解**。所以，加油吧！

> 再给一下[文章中涉及的代码]()