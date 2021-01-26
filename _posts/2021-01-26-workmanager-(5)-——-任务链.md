---
layout: post
title: "WorkManager (5) —— 任务链"
date: "2021-01-26 09:56:23 +0800"
description: 
img: 
tags: [Android, WorkManager]
mathjax: true
---

之前讨论过的任务，无论是单次任务，还是周期性任务，都是单一的任务项执行。如果我们要多个任务项同时进行，或者按一定顺序执行，就需要用到链式任务。

## 任务链

任务链的启动，需要一个核心类：`WorkContinuation`

### WorkContinuation

```java
public abstract class WorkContinuation {
    // 添加单个任务
    public final @NonNull WorkContinuation then(@NonNull OneTimeWorkRequest work) {
        return then(Collections.singletonList(work));
    }

    // 添加任务表
    public abstract @NonNull WorkContinuation then(@NonNull List<OneTimeWorkRequest> work);

    // 执行已添加的所有任务
    public abstract @NonNull Operation enqueue();

    // 连接多个continuation，并作为另外的continuation的前置条件
    public static @NonNull WorkContinuation combine(@NonNull List<WorkContinuation> continuations) {
        return continuations.get(0).combineInternal(continuations);
    }
}
```

上述四个方法，前三个都好理解，第四个到底是什么意思呢？

    官方的例子   A       C
               |       |
               B       D
               |       |
               +-------+
                   |
                   E
      如果有执行顺序 A->B，C->D，且须这两个都执行完了，再执行E，那应该这样写：

      WorkContinuation left = workManager.beginWith(A).then(B);
      WorkContinuation right = workManager.beginWith(C).then(D);
      WorkContinuation final = WorkContinuation.combine(Arrays.asList(left, right)).then(E);
      final.enqueue();


所以，才会有“前置条件"(prerequisites)的说法。

那么，如何获到一个WorkContinuation呢？

### 构造与执行

获取一个WorkContinuation很简单，WorkManager里面的的"`begin*`"方法族就是干这个的：

```java
public final @NonNull WorkContinuation beginWith(@NonNull OneTimeWorkRequest work) {
    return beginWith(Collections.singletonList(work));
}

public abstract @NonNull WorkContinuation beginWith(@NonNull List<OneTimeWorkRequest> work);
```

上面是其中两个简单的方法，直接传入`OneTimeWorkRequest`任务（或列表）就行了。

来测试一下，目的是顺序执行多个任务。**DelayWorker**增加时间参数，用以控制不同的任务执行时间。


```kotlin
class DelayWorker(context: Context, workerParams: WorkerParameters) : Worker(context, workerParams) {

    override fun doWork(): Result {
        val name = inputData.getString(ARG_NAME)
        Log.d(TAG, "[$name] doWork on ${Thread.currentThread().id} started - ${System.currentTimeMillis()}  -> $id")
        // emulated work
        Thread.sleep(inputData.getLong(ARG_TEST_DURATION, 3_000L))
        Log.d(TAG, "[$name] doWork on ${Thread.currentThread().id} ended - ${System.currentTimeMillis()}")
        return Result.success()
    }

    companion object {
        const val ARG_NAME = "name"
        const val ARG_TEST_DURATION = "duration" // 执行时间参数
    }
}

class FailedWorker(context: Context, workerParams: WorkerParameters) : Worker(context, workerParams) {

    override fun doWork(): Result {
        Log.d(TAG, "FailedWorker on ${Thread.currentThread().id} started - ${System.currentTimeMillis()}")
        // emulated work
        Thread.sleep(1_000)
        Log.d(TAG, "FailedWorker on ${Thread.currentThread().id} ended - ${System.currentTimeMillis()}")
        return Result.failure()
    }
}

// 执行任务
val request = OneTimeWorkRequestBuilder<DelayWorker>()
    .setInputData(
        Data.Builder()
            .putString(DelayWorker.ARG_NAME, "1")
            .build()
    )
    .build()
val request2 = OneTimeWorkRequestBuilder<DelayWorker>()
    .setInputData(
        Data.Builder()
            .putString(DelayWorker.ARG_NAME, "2")
            .putLong(DelayWorker.ARG_TEST_DURATION, 5_000L)
            .build()
    )
    .build()
val request3 = OneTimeWorkRequestBuilder<DelayWorker>()
    .setInputData(
        Data.Builder()
            .putString(DelayWorker.ARG_NAME, "3")
            .putLong(DelayWorker.ARG_TEST_DURATION, 1_500L)
            .build()
    )
    .build()
val request4 = OneTimeWorkRequestBuilder<FailedWorker>().build()
Log.d(TAG, "chain enqueue on ${Thread.currentThread().id} ${System.currentTimeMillis()}")

WorkManager.getInstance(applicationContext)
    .beginWith(request)
    .then(request2)
    .then(request3)
    .then(request4)
    .enqueue()
```

看看结果：

    2021-01-14 10:40:48.800 14655-14655/com.jacee.examples.workmanager D/JTest: chain enqueue on 2 1610592048800
    2021-01-14 10:40:48.851 14655-14688/com.jacee.examples.workmanager D/JTest: [1] doWork on 7255 started - 1610592048851  -> 9c350943-a964-4ae2-a085-f59a7acad577
    2021-01-14 10:40:51.853 14655-14688/com.jacee.examples.workmanager D/JTest: [1] doWork on 7255 ended - 1610592051853
    2021-01-14 10:40:51.926 14655-14698/com.jacee.examples.workmanager D/JTest: [2] doWork on 7257 started - 1610592051926  -> 2ca49f36-a84c-4bd2-9bca-1db84698e789
    2021-01-14 10:40:56.928 14655-14698/com.jacee.examples.workmanager D/JTest: [2] doWork on 7257 ended - 1610592056928
    2021-01-14 10:40:57.000 14655-14705/com.jacee.examples.workmanager D/JTest: [3] doWork on 7258 started - 1610592057000  -> 64dadb7e-d11c-4362-b177-8f66b87e05b6
    2021-01-14 10:40:58.501 14655-14705/com.jacee.examples.workmanager D/JTest: [3] doWork on 7258 ended - 1610592058500
    2021-01-14 10:40:58.569 14655-14711/com.jacee.examples.workmanager D/JTest: FailedWorker on 7259 started - 1610592058569
    2021-01-14 10:40:59.570 14655-14711/com.jacee.examples.workmanager D/JTest: FailedWorker on 7259 ended - 1610592059569

四个任务如愿顺序执行了，执行时间依次为：3秒（默认），5秒，1.5秒，1秒。


### fail任务处理

前面的第四个任务，用了一个`FailedWorker`，它执行后返回是`Result.failure()`，即执行失败。前文的结果，好像并没有什么影响。下面，把它提到第二位置，再添加状态监听：

```kotlin
WorkManager.getInstance(applicationContext)
    .beginWith(request)
    .then(request4)
    .then(request2)
    .then(request3)
    .apply {
        workInfosLiveData.observe({lifecycle}) { l ->
            l.forEach {
                Log.d(TAG, "chain live: ${it.id}: ${it.state}")
            }
        }
    }
    .enqueue()
```

执行结果：

    2021-01-14 10:44:47.925 14934-14934/com.jacee.examples.workmanager D/JTest: chain enqueue on 2 1610592287925
    2021-01-14 10:44:47.978 14934-14934/com.jacee.examples.workmanager D/JTest: ----------------
    2021-01-14 10:44:47.992 14934-14986/com.jacee.examples.workmanager D/JTest: [1] doWork on 7259 started - 1610592287992  -> 04c4e398-2427-448d-9e6a-e2893064ddca
    2021-01-14 10:44:47.994 14934-14934/com.jacee.examples.workmanager D/JTest: chain live: 04c4e398-2427-448d-9e6a-e2893064ddca: ENQUEUED
    2021-01-14 10:44:47.994 14934-14934/com.jacee.examples.workmanager D/JTest: chain live: 1e36ea88-f60d-446e-b765-cf6b0e17be71: BLOCKED
    2021-01-14 10:44:47.995 14934-14934/com.jacee.examples.workmanager D/JTest: chain live: 2c5a1314-abfd-4d68-b540-28dd7038984f: BLOCKED
    2021-01-14 10:44:47.995 14934-14934/com.jacee.examples.workmanager D/JTest: chain live: 49eb7dd2-f826-4c6a-a419-60090ef3166d: BLOCKED
    2021-01-14 10:44:47.995 14934-14934/com.jacee.examples.workmanager D/JTest: ----------------
    2021-01-14 10:44:47.997 14934-14934/com.jacee.examples.workmanager D/JTest: chain live: 04c4e398-2427-448d-9e6a-e2893064ddca: RUNNING
    2021-01-14 10:44:47.997 14934-14934/com.jacee.examples.workmanager D/JTest: chain live: 1e36ea88-f60d-446e-b765-cf6b0e17be71: BLOCKED
    2021-01-14 10:44:47.997 14934-14934/com.jacee.examples.workmanager D/JTest: chain live: 2c5a1314-abfd-4d68-b540-28dd7038984f: BLOCKED
    2021-01-14 10:44:47.997 14934-14934/com.jacee.examples.workmanager D/JTest: chain live: 49eb7dd2-f826-4c6a-a419-60090ef3166d: BLOCKED
    2021-01-14 10:44:47.997 14934-14934/com.jacee.examples.workmanager D/JTest: ----------------
    2021-01-14 10:44:50.996 14934-14986/com.jacee.examples.workmanager D/JTest: [1] doWork on 7259 ended - 1610592290996
    2021-01-14 10:44:51.102 14934-14988/com.jacee.examples.workmanager D/JTest: FailedWorker on 7261 started - 1610592291102
    2021-01-14 10:44:51.103 14934-14934/com.jacee.examples.workmanager D/JTest: chain live: 04c4e398-2427-448d-9e6a-e2893064ddca: SUCCEEDED
    2021-01-14 10:44:51.103 14934-14934/com.jacee.examples.workmanager D/JTest: chain live: 1e36ea88-f60d-446e-b765-cf6b0e17be71: ENQUEUED
    2021-01-14 10:44:51.103 14934-14934/com.jacee.examples.workmanager D/JTest: chain live: 2c5a1314-abfd-4d68-b540-28dd7038984f: BLOCKED
    2021-01-14 10:44:51.103 14934-14934/com.jacee.examples.workmanager D/JTest: chain live: 49eb7dd2-f826-4c6a-a419-60090ef3166d: BLOCKED
    2021-01-14 10:44:51.103 14934-14934/com.jacee.examples.workmanager D/JTest: ----------------
    2021-01-14 10:44:51.117 14934-14934/com.jacee.examples.workmanager D/JTest: chain live: 04c4e398-2427-448d-9e6a-e2893064ddca: SUCCEEDED
    2021-01-14 10:44:51.117 14934-14934/com.jacee.examples.workmanager D/JTest: chain live: 1e36ea88-f60d-446e-b765-cf6b0e17be71: RUNNING
    2021-01-14 10:44:51.117 14934-14934/com.jacee.examples.workmanager D/JTest: chain live: 2c5a1314-abfd-4d68-b540-28dd7038984f: BLOCKED
    2021-01-14 10:44:51.118 14934-14934/com.jacee.examples.workmanager D/JTest: chain live: 49eb7dd2-f826-4c6a-a419-60090ef3166d: BLOCKED
    2021-01-14 10:44:51.118 14934-14934/com.jacee.examples.workmanager D/JTest: ----------------
    2021-01-14 10:44:52.105 14934-14988/com.jacee.examples.workmanager D/JTest: FailedWorker on 7261 ended - 1610592292105
    2021-01-14 10:44:52.162 14934-14934/com.jacee.examples.workmanager D/JTest: chain live: 04c4e398-2427-448d-9e6a-e2893064ddca: SUCCEEDED
    2021-01-14 10:44:52.163 14934-14934/com.jacee.examples.workmanager D/JTest: chain live: 1e36ea88-f60d-446e-b765-cf6b0e17be71: FAILED
    2021-01-14 10:44:52.163 14934-14934/com.jacee.examples.workmanager D/JTest: chain live: 2c5a1314-abfd-4d68-b540-28dd7038984f: FAILED
    2021-01-14 10:44:52.163 14934-14934/com.jacee.examples.workmanager D/JTest: chain live: 49eb7dd2-f826-4c6a-a419-60090ef3166d: FAILED
    2021-01-14 10:44:52.163 14934-14934/com.jacee.examples.workmanager D/JTest: ----------------


从日志可以清楚地看到，最开始的时候，除第一个任务为**ENQUEUED**，其他都是**BLOCKED**阻塞状态。第一个任务成功执行状态变化：**ENQUEUED -> RUNNING -> SUCCEEDED**。

第一个任务**SUCCEEDED**的同时，第二个任务变为**ENQUEUED**准备执行，接下来，第二个任务经历 **RUNNING -> FAILED**，这导致剩下两个任务也同时从 **BLOCKED -> FAILED**。

**一旦链式任务有执行失败的，后续任务将全部不执行而以失败处理**，这一点，官方API文档说得很明白：

> New items depend on the successful completion of all previously added {@link OneTimeWorkRequest}s.


### combine的使用

前面介绍`combine`方法的时候，已经简单说明了它的用途，下面就来实操一下。

#### 案例

在已有的3个正常request的基础上，再增加request5和request6，需要执行以下调用顺序：

1. request（3秒） -> request2（5秒）：**序列一**
2. request3（1.5秒） -> request5（2秒）：**序列二**
3. 1和2两个执行链同时执行
4. 3中所有执行完毕后，执行request6（1秒）

这其实是完全和API的example一致的

```kotlin
val request5 = OneTimeWorkRequestBuilder<DelayWorker>()
    .setInputData(
        Data.Builder()
            .putString(DelayWorker.ARG_NAME, "5")
            .putLong(DelayWorker.ARG_TEST_DURATION, 2_000L)
            .build()
    )
    .build()
val request6 = OneTimeWorkRequestBuilder<DelayWorker>()
    .setInputData(
        Data.Builder()
            .putString(DelayWorker.ARG_NAME, "6")
            .putLong(DelayWorker.ARG_TEST_DURATION, 1_000L)
            .build()
    )
    .build()

WorkContinuation.combine(arrayListOf(
    WorkManager.getInstance(applicationContext)
        .beginWith(request)
        .then(request2),
    WorkManager.getInstance(applicationContext)
        .beginWith(request3)
        .then(request5),
))
    .then(request6)
    .apply {
        workInfosLiveData.observe({lifecycle}) { l ->
            l.forEach {
                Log.d(TAG, "chain live combine: ${it.id}: ${it.state}")
            }
            Log.d(TAG, "----------------")
        }
    }
    .enqueue()

```

执行结果很长，我们直接在每一步后面加上分析说明：

    2021-01-14 10:51:51.282 15177-15177/com.jacee.examples.workmanager D/JTest: chain enqueue on 2 1610592711282
    2021-01-14 10:51:51.371 15177-15177/com.jacee.examples.workmanager D/JTest: ----------------
    2021-01-14 10:51:51.379 15177-15282/com.jacee.examples.workmanager D/JTest: [1] doWork on 7263 started - 1610592711379  -> 17400ade-6d69-492b-b324-4ce9f4475639
    2021-01-14 10:51:51.384 15177-15283/com.jacee.examples.workmanager D/JTest: [3] doWork on 7264 started - 1610592711384  -> 8002d485-e326-4fdf-9422-80a76af58bfd
    2021-01-14 10:51:51.391 15177-15177/com.jacee.examples.workmanager D/JTest: chain live combine: 17400ade-6d69-492b-b324-4ce9f4475639: RUNNING
    2021-01-14 10:51:51.391 15177-15177/com.jacee.examples.workmanager D/JTest: chain live combine: 6d66d32a-f416-447a-8b9b-8953631e1896: BLOCKED
    2021-01-14 10:51:51.391 15177-15177/com.jacee.examples.workmanager D/JTest: chain live combine: 8002d485-e326-4fdf-9422-80a76af58bfd: RUNNING
    2021-01-14 10:51:51.391 15177-15177/com.jacee.examples.workmanager D/JTest: chain live combine: 85f8d9c2-1bed-4665-8e79-94aade347dcc: BLOCKED
    2021-01-14 10:51:51.391 15177-15177/com.jacee.examples.workmanager D/JTest: chain live combine: a46d7945-ae68-4d87-b358-0c8e27489f51: BLOCKED
    2021-01-14 10:51:51.391 15177-15177/com.jacee.examples.workmanager D/JTest: chain live combine: ec165bdc-e301-45ea-9226-3f54877deda2: BLOCKED
    > 第一轮状态回调：1和3同时开始执行，RUNNING
    2021-01-14 10:51:51.391 15177-15177/com.jacee.examples.workmanager D/JTest: ----------------
    2021-01-14 10:51:52.885 15177-15283/com.jacee.examples.workmanager D/JTest: [3] doWork on 7264 ended - 1610592712885
    2021-01-14 10:51:52.993 15177-15177/com.jacee.examples.workmanager D/JTest: chain live combine: 17400ade-6d69-492b-b324-4ce9f4475639: RUNNING
    2021-01-14 10:51:52.994 15177-15177/com.jacee.examples.workmanager D/JTest: chain live combine: 6d66d32a-f416-447a-8b9b-8953631e1896: BLOCKED
    2021-01-14 10:51:52.994 15177-15177/com.jacee.examples.workmanager D/JTest: chain live combine: 8002d485-e326-4fdf-9422-80a76af58bfd: SUCCEEDED
    2021-01-14 10:51:52.994 15177-15177/com.jacee.examples.workmanager D/JTest: chain live combine: 85f8d9c2-1bed-4665-8e79-94aade347dcc: BLOCKED
    2021-01-14 10:51:52.994 15177-15177/com.jacee.examples.workmanager D/JTest: chain live combine: a46d7945-ae68-4d87-b358-0c8e27489f51: BLOCKED
    2021-01-14 10:51:52.994 15177-15177/com.jacee.examples.workmanager D/JTest: chain live combine: ec165bdc-e301-45ea-9226-3f54877deda2: ENQUEUED
    2021-01-14 10:51:52.994 15177-15177/com.jacee.examples.workmanager D/JTest: ----------------
    > 第二轮状态回调：时间较短的3执行完毕，变为SUCCEEDED；同时，后续5变为ENQUEUED，准备执行
    2021-01-14 10:51:52.994 15177-15418/com.jacee.examples.workmanager D/JTest: [5] doWork on 7265 started - 1610592712994  -> ec165bdc-e301-45ea-9226-3f54877deda2
    2021-01-14 10:51:53.002 15177-15177/com.jacee.examples.workmanager D/JTest: chain live combine: 17400ade-6d69-492b-b324-4ce9f4475639: RUNNING
    2021-01-14 10:51:53.003 15177-15177/com.jacee.examples.workmanager D/JTest: chain live combine: 6d66d32a-f416-447a-8b9b-8953631e1896: BLOCKED
    2021-01-14 10:51:53.003 15177-15177/com.jacee.examples.workmanager D/JTest: chain live combine: 8002d485-e326-4fdf-9422-80a76af58bfd: SUCCEEDED
    2021-01-14 10:51:53.003 15177-15177/com.jacee.examples.workmanager D/JTest: chain live combine: 85f8d9c2-1bed-4665-8e79-94aade347dcc: BLOCKED
    2021-01-14 10:51:53.003 15177-15177/com.jacee.examples.workmanager D/JTest: chain live combine: a46d7945-ae68-4d87-b358-0c8e27489f51: BLOCKED
    2021-01-14 10:51:53.003 15177-15177/com.jacee.examples.workmanager D/JTest: chain live combine: ec165bdc-e301-45ea-9226-3f54877deda2: RUNNING
    2021-01-14 10:51:53.003 15177-15177/com.jacee.examples.workmanager D/JTest: ----------------
    > 第三轮状态回调：5开始执行，RUNNING，这时，1仍然是RUNNING
    2021-01-14 10:51:54.380 15177-15282/com.jacee.examples.workmanager D/JTest: [1] doWork on 7263 ended - 1610592714380
    2021-01-14 10:51:54.441 15177-15177/com.jacee.examples.workmanager D/JTest: chain live combine: 17400ade-6d69-492b-b324-4ce9f4475639: SUCCEEDED
    2021-01-14 10:51:54.442 15177-15177/com.jacee.examples.workmanager D/JTest: chain live combine: 6d66d32a-f416-447a-8b9b-8953631e1896: BLOCKED
    2021-01-14 10:51:54.442 15177-15177/com.jacee.examples.workmanager D/JTest: chain live combine: 8002d485-e326-4fdf-9422-80a76af58bfd: SUCCEEDED
    2021-01-14 10:51:54.442 15177-15177/com.jacee.examples.workmanager D/JTest: chain live combine: 85f8d9c2-1bed-4665-8e79-94aade347dcc: BLOCKED
    2021-01-14 10:51:54.442 15177-15177/com.jacee.examples.workmanager D/JTest: chain live combine: a46d7945-ae68-4d87-b358-0c8e27489f51: ENQUEUED
    2021-01-14 10:51:54.442 15177-15177/com.jacee.examples.workmanager D/JTest: chain live combine: ec165bdc-e301-45ea-9226-3f54877deda2: RUNNING
    2021-01-14 10:51:54.442 15177-15177/com.jacee.examples.workmanager D/JTest: ----------------
    > 第三轮状态回调：1执行完毕，SUCCEEDED，后续2变为ENQUEUED准备执行
    2021-01-14 10:51:54.443 15177-15421/com.jacee.examples.workmanager D/JTest: [2] doWork on 7266 started - 1610592714443  -> a46d7945-ae68-4d87-b358-0c8e27489f51
    2021-01-14 10:51:54.481 15177-15177/com.jacee.examples.workmanager D/JTest: chain live combine: 17400ade-6d69-492b-b324-4ce9f4475639: SUCCEEDED
    2021-01-14 10:51:54.481 15177-15177/com.jacee.examples.workmanager D/JTest: chain live combine: 6d66d32a-f416-447a-8b9b-8953631e1896: BLOCKED
    2021-01-14 10:51:54.481 15177-15177/com.jacee.examples.workmanager D/JTest: chain live combine: 8002d485-e326-4fdf-9422-80a76af58bfd: SUCCEEDED
    2021-01-14 10:51:54.481 15177-15177/com.jacee.examples.workmanager D/JTest: chain live combine: 85f8d9c2-1bed-4665-8e79-94aade347dcc: BLOCKED
    2021-01-14 10:51:54.482 15177-15177/com.jacee.examples.workmanager D/JTest: chain live combine: a46d7945-ae68-4d87-b358-0c8e27489f51: RUNNING
    2021-01-14 10:51:54.482 15177-15177/com.jacee.examples.workmanager D/JTest: chain live combine: ec165bdc-e301-45ea-9226-3f54877deda2: RUNNING
    2021-01-14 10:51:54.482 15177-15177/com.jacee.examples.workmanager D/JTest: ----------------
    > 第四轮状态回调：2开始执行，RUNNING
    2021-01-14 10:51:54.995 15177-15418/com.jacee.examples.workmanager D/JTest: [5] doWork on 7265 ended - 1610592714995
    2021-01-14 10:51:55.024 15177-15177/com.jacee.examples.workmanager D/JTest: chain live combine: 17400ade-6d69-492b-b324-4ce9f4475639: SUCCEEDED
    2021-01-14 10:51:55.024 15177-15177/com.jacee.examples.workmanager D/JTest: chain live combine: 6d66d32a-f416-447a-8b9b-8953631e1896: BLOCKED
    2021-01-14 10:51:55.024 15177-15177/com.jacee.examples.workmanager D/JTest: chain live combine: 8002d485-e326-4fdf-9422-80a76af58bfd: SUCCEEDED
    2021-01-14 10:51:55.025 15177-15177/com.jacee.examples.workmanager D/JTest: chain live combine: 85f8d9c2-1bed-4665-8e79-94aade347dcc: BLOCKED
    2021-01-14 10:51:55.025 15177-15177/com.jacee.examples.workmanager D/JTest: chain live combine: a46d7945-ae68-4d87-b358-0c8e27489f51: RUNNING
    2021-01-14 10:51:55.025 15177-15177/com.jacee.examples.workmanager D/JTest: chain live combine: ec165bdc-e301-45ea-9226-3f54877deda2: SUCCEEDED
    2021-01-14 10:51:55.025 15177-15177/com.jacee.examples.workmanager D/JTest: ----------------
    > 第五轮状态回调：5执行完毕，SUCCEEDED。至此，序列二执行完毕
    2021-01-14 10:51:59.444 15177-15421/com.jacee.examples.workmanager D/JTest: [2] doWork on 7266 ended - 1610592719444
    2021-01-14 10:51:59.524 15177-15177/com.jacee.examples.workmanager D/JTest: chain live combine: 17400ade-6d69-492b-b324-4ce9f4475639: SUCCEEDED
    2021-01-14 10:51:59.524 15177-15177/com.jacee.examples.workmanager D/JTest: chain live combine: 6d66d32a-f416-447a-8b9b-8953631e1896: ENQUEUED
    2021-01-14 10:51:59.524 15177-15177/com.jacee.examples.workmanager D/JTest: chain live combine: 8002d485-e326-4fdf-9422-80a76af58bfd: SUCCEEDED
    2021-01-14 10:51:59.524 15177-15177/com.jacee.examples.workmanager D/JTest: chain live combine: 85f8d9c2-1bed-4665-8e79-94aade347dcc: BLOCKED
    2021-01-14 10:51:59.524 15177-15177/com.jacee.examples.workmanager D/JTest: chain live combine: a46d7945-ae68-4d87-b358-0c8e27489f51: SUCCEEDED
    2021-01-14 10:51:59.524 15177-15177/com.jacee.examples.workmanager D/JTest: chain live combine: ec165bdc-e301-45ea-9226-3f54877deda2: SUCCEEDED
    2021-01-14 10:51:59.524 15177-15177/com.jacee.examples.workmanager D/JTest: ----------------
    > 第六轮状态回调：2执行完毕，SUCCEEDED。至此，序列一执行完毕。同时6d66d32a-f416-447a-8b9b-8953631e1896进入ENQUEUED状态
    2021-01-14 10:51:59.551 15177-15177/com.jacee.examples.workmanager D/JTest: chain live combine: 17400ade-6d69-492b-b324-4ce9f4475639: SUCCEEDED
    2021-01-14 10:51:59.551 15177-15177/com.jacee.examples.workmanager D/JTest: chain live combine: 6d66d32a-f416-447a-8b9b-8953631e1896: SUCCEEDED
    2021-01-14 10:51:59.551 15177-15177/com.jacee.examples.workmanager D/JTest: chain live combine: 8002d485-e326-4fdf-9422-80a76af58bfd: SUCCEEDED
    2021-01-14 10:51:59.551 15177-15177/com.jacee.examples.workmanager D/JTest: chain live combine: 85f8d9c2-1bed-4665-8e79-94aade347dcc: ENQUEUED
    2021-01-14 10:51:59.551 15177-15177/com.jacee.examples.workmanager D/JTest: chain live combine: a46d7945-ae68-4d87-b358-0c8e27489f51: SUCCEEDED
    2021-01-14 10:51:59.551 15177-15177/com.jacee.examples.workmanager D/JTest: chain live combine: ec165bdc-e301-45ea-9226-3f54877deda2: SUCCEEDED
    2021-01-14 10:51:59.551 15177-15177/com.jacee.examples.workmanager D/JTest: ----------------
    > 第七轮状态回调：6d66d32a-f416-447a-8b9b-8953631e1896，SUCCEEDED；同时85f8d9c2-1bed-4665-8e79-94aade347dcc进入ENQUEUED状态
    2021-01-14 10:51:59.559 15177-15282/com.jacee.examples.workmanager D/JTest: [6] doWork on 7263 started - 1610592719559  -> 85f8d9c2-1bed-4665-8e79-94aade347dcc
    2021-01-14 10:51:59.565 15177-15177/com.jacee.examples.workmanager D/JTest: chain live combine: 17400ade-6d69-492b-b324-4ce9f4475639: SUCCEEDED
    2021-01-14 10:51:59.565 15177-15177/com.jacee.examples.workmanager D/JTest: chain live combine: 6d66d32a-f416-447a-8b9b-8953631e1896: SUCCEEDED
    2021-01-14 10:51:59.565 15177-15177/com.jacee.examples.workmanager D/JTest: chain live combine: 8002d485-e326-4fdf-9422-80a76af58bfd: SUCCEEDED
    2021-01-14 10:51:59.565 15177-15177/com.jacee.examples.workmanager D/JTest: chain live combine: 85f8d9c2-1bed-4665-8e79-94aade347dcc: RUNNING
    2021-01-14 10:51:59.565 15177-15177/com.jacee.examples.workmanager D/JTest: chain live combine: a46d7945-ae68-4d87-b358-0c8e27489f51: SUCCEEDED
    2021-01-14 10:51:59.565 15177-15177/com.jacee.examples.workmanager D/JTest: chain live combine: ec165bdc-e301-45ea-9226-3f54877deda2: SUCCEEDED
    2021-01-14 10:51:59.565 15177-15177/com.jacee.examples.workmanager D/JTest: ----------------
    > 第八轮状态回调：上一轮提到的85f8d9c2-1bed-4665-8e79-94aade347dcc（其实就是request6）进入RUNNING状态开始执行
    2021-01-14 10:52:00.564 15177-15282/com.jacee.examples.workmanager D/JTest: [6] doWork on 7263 ended - 1610592720564
    2021-01-14 10:52:00.624 15177-15177/com.jacee.examples.workmanager D/JTest: chain live combine: 17400ade-6d69-492b-b324-4ce9f4475639: SUCCEEDED
    2021-01-14 10:52:00.624 15177-15177/com.jacee.examples.workmanager D/JTest: chain live combine: 6d66d32a-f416-447a-8b9b-8953631e1896: SUCCEEDED
    2021-01-14 10:52:00.624 15177-15177/com.jacee.examples.workmanager D/JTest: chain live combine: 8002d485-e326-4fdf-9422-80a76af58bfd: SUCCEEDED
    2021-01-14 10:52:00.624 15177-15177/com.jacee.examples.workmanager D/JTest: chain live combine: 85f8d9c2-1bed-4665-8e79-94aade347dcc: SUCCEEDED
    2021-01-14 10:52:00.624 15177-15177/com.jacee.examples.workmanager D/JTest: chain live combine: a46d7945-ae68-4d87-b358-0c8e27489f51: SUCCEEDED
    2021-01-14 10:52:00.624 15177-15177/com.jacee.examples.workmanager D/JTest: chain live combine: ec165bdc-e301-45ea-9226-3f54877deda2: SUCCEEDED
    2021-01-14 10:52:00.624 15177-15177/com.jacee.examples.workmanager D/JTest: ----------------
    > 第九轮状态回调：request6执行完毕，SUCCEEDED。至此，全部完成。


上面日志的流程说明，已经很清楚了。但有疑问，为什么总共5个request，`WorkInfo`却有6个？可以看到，在第六轮的时候，任务**6d66d32a-f416-447a-8b9b-8953631e1896**进入**ENQUEUED**，再到第七轮，没有经过**RUNNING**，该任务直接进入了**SUCCEEDED**。它并不是5个任务之一，那它是什么？其实可以大胆猜测，它就是用于保证request6等待序列一、序列二完全执行完毕才执行的“**包裹任务**”，毕竟，combine的返回结果不也是一个`WorkContinuation`吗？

#### combine源码分析

前面提到的“包裹任务”，真如猜测吗？还是来看源码比较放心。

`combine`方法最终会调到`WorkContinuationImpl`的`combineInternal`方法：

```java
@Override
protected @NonNull WorkContinuation combineInternal(
        @NonNull List<WorkContinuation> continuations) {
    // 构造了一个CombineContinuationsWorker
    OneTimeWorkRequest combinedWork =
            new OneTimeWorkRequest.Builder(CombineContinuationsWorker.class)
                    .setInputMerger(ArrayCreatingInputMerger.class)
                    .build();

    // 所有的需要combine的任务
    List<WorkContinuationImpl> parents = new ArrayList<>(continuations.size());
    for (WorkContinuation continuation : continuations) {
        parents.add((WorkContinuationImpl) continuation);
    }

    return new WorkContinuationImpl(mWorkManagerImpl,
            null,
            ExistingWorkPolicy.KEEP,
            Collections.singletonList(combinedWork),
            parents);
}
```

构造`WorkContinuationImpl`对象的时候，用到了`combinedWork`任务请求、全部任务列表。

```java
WorkContinuationImpl(@NonNull WorkManagerImpl workManagerImpl,
        String name,
        ExistingWorkPolicy existingWorkPolicy,
        @NonNull List<? extends WorkRequest> work,
        @Nullable List<WorkContinuationImpl> parents) {
    mWorkManagerImpl = workManagerImpl;
    mName = name;
    mExistingWorkPolicy = existingWorkPolicy;
    mWork = work;
    mParents = parents;
    mIds = new ArrayList<>(mWork.size());
    mAllIds = new ArrayList<>();
    if (parents != null) {
        for (WorkContinuationImpl parent : parents) {
            mAllIds.addAll(parent.mAllIds); // mAllIds添加所有任务id
        }
    }
    for (int i = 0; i < work.size(); i++) {
        String id = work.get(i).getStringId();
        mIds.add(id);       // mIds添加所有work的id
        mAllIds.add(id);    // 同时，id也添加到了mAllIds里
    }
}
```

其实上述传入的`work`参数列表，只有`combineWork`一个，而该request指向的CombineContinuationsWorker，就是用于combine任务列表的。

再来看看监听的调用也到了`WorkContinuationImpl`里：

```java
@Override
public @NonNull LiveData<List<WorkInfo>> getWorkInfosLiveData() {
    return mWorkManagerImpl.getWorkInfosById(mAllIds);
}
```

这个`mAllIds`，不就是前面提到的吗，**它除了包括所有的任务id外，确实还包括了combine需要用到的一个（或以上）的combine任务（包裹任务）id**


#### failure处理

如果combine的任务列表里，有执行失败的任务，将怎么影响整个任务序列呢？

```kotlin
WorkContinuation.combine(arrayListOf(
    WorkManager.getInstance(applicationContext)
        .beginWith(request2)
        .then(request),
    WorkManager.getInstance(applicationContext)
        .beginWith(request4)
        .then(request3),
))
    .then(request6)
    .apply {
        workInfosLiveData.observe({lifecycle}) { l ->
            l.forEach {
                Log.d(TAG, "chain live combine: ${it.id}: ${it.state}")
            }
            Log.d(TAG, "----------------")
        }
    }
    .enqueue()
```

上述任务描述为：

1. request2（5秒） -> request（3秒）：序列一
2. request4（3秒） -> request3（1.5秒）:序列二
3. 1和2完毕后，request6


结果：

    2021-01-14 14:17:25.652 19503-19503/com.jacee.examples.workmanager D/JTest: chain enqueue on 2 1610605045652
    2021-01-14 14:17:25.730 19503-19503/com.jacee.examples.workmanager D/JTest: ----------------
    2021-01-14 14:17:25.740 19503-19664/com.jacee.examples.workmanager D/JTest: [2] doWork on 7385 started - 1610605045740  -> 302f1da0-99ba-486e-b471-0d74fbc23394
    2021-01-14 14:17:25.748 19503-19665/com.jacee.examples.workmanager D/JTest: FailedWorker on 7386 started - 1610605045748
    2021-01-14 14:17:25.750 19503-19503/com.jacee.examples.workmanager D/JTest: chain live combine: 0e1eb5a8-c3ed-4c35-8981-f63439169cf3: BLOCKED
    2021-01-14 14:17:25.750 19503-19503/com.jacee.examples.workmanager D/JTest: chain live combine: 302f1da0-99ba-486e-b471-0d74fbc23394: RUNNING
    2021-01-14 14:17:25.750 19503-19503/com.jacee.examples.workmanager D/JTest: chain live combine: 3e7b86fe-e3a8-4d74-92c0-5611fed39919: BLOCKED
    2021-01-14 14:17:25.750 19503-19503/com.jacee.examples.workmanager D/JTest: chain live combine: adc5049b-e4b2-4c35-a7a6-a5daca7a46f2: BLOCKED
    2021-01-14 14:17:25.750 19503-19503/com.jacee.examples.workmanager D/JTest: chain live combine: d7364e62-640c-4f70-b544-5c344725718c: ENQUEUED
    2021-01-14 14:17:25.750 19503-19503/com.jacee.examples.workmanager D/JTest: chain live combine: ed4624ab-2feb-48d6-982c-0676687b8480: BLOCKED
    2021-01-14 14:17:25.750 19503-19503/com.jacee.examples.workmanager D/JTest: ----------------
    2021-01-14 14:17:25.755 19503-19503/com.jacee.examples.workmanager D/JTest: chain live combine: 0e1eb5a8-c3ed-4c35-8981-f63439169cf3: BLOCKED
    2021-01-14 14:17:25.755 19503-19503/com.jacee.examples.workmanager D/JTest: chain live combine: 302f1da0-99ba-486e-b471-0d74fbc23394: RUNNING
    2021-01-14 14:17:25.755 19503-19503/com.jacee.examples.workmanager D/JTest: chain live combine: 3e7b86fe-e3a8-4d74-92c0-5611fed39919: BLOCKED
    2021-01-14 14:17:25.755 19503-19503/com.jacee.examples.workmanager D/JTest: chain live combine: adc5049b-e4b2-4c35-a7a6-a5daca7a46f2: BLOCKED
    2021-01-14 14:17:25.755 19503-19503/com.jacee.examples.workmanager D/JTest: chain live combine: d7364e62-640c-4f70-b544-5c344725718c: RUNNING
    2021-01-14 14:17:25.755 19503-19503/com.jacee.examples.workmanager D/JTest: chain live combine: ed4624ab-2feb-48d6-982c-0676687b8480: BLOCKED
    2021-01-14 14:17:25.755 19503-19503/com.jacee.examples.workmanager D/JTest: ----------------
    > 2和FailedWorker都到RUNNING状态
    2021-01-14 14:17:26.752 19503-19665/com.jacee.examples.workmanager D/JTest: FailedWorker on 7386 ended - 1610605046752
    2021-01-14 14:17:26.836 19503-19503/com.jacee.examples.workmanager D/JTest: chain live combine: 0e1eb5a8-c3ed-4c35-8981-f63439169cf3: BLOCKED
    2021-01-14 14:17:26.836 19503-19503/com.jacee.examples.workmanager D/JTest: chain live combine: 302f1da0-99ba-486e-b471-0d74fbc23394: RUNNING
    2021-01-14 14:17:26.836 19503-19503/com.jacee.examples.workmanager D/JTest: chain live combine: 3e7b86fe-e3a8-4d74-92c0-5611fed39919: FAILED
    2021-01-14 14:17:26.837 19503-19503/com.jacee.examples.workmanager D/JTest: chain live combine: adc5049b-e4b2-4c35-a7a6-a5daca7a46f2: FAILED
    2021-01-14 14:17:26.837 19503-19503/com.jacee.examples.workmanager D/JTest: chain live combine: d7364e62-640c-4f70-b544-5c344725718c: FAILED
    2021-01-14 14:17:26.837 19503-19503/com.jacee.examples.workmanager D/JTest: chain live combine: ed4624ab-2feb-48d6-982c-0676687b8480: FAILED
    2021-01-14 14:17:26.837 19503-19503/com.jacee.examples.workmanager D/JTest: ----------------
    > 时间短的FailedWorker执行完毕，除2为RUNNING，0e1eb5a8-c3ed-4c35-8981-f63439169cf3为BLOCKED外，其余全部FAILED
    2021-01-14 14:17:30.744 19503-19664/com.jacee.examples.workmanager D/JTest: [2] doWork on 7385 ended - 1610605050743
    2021-01-14 14:17:30.843 19503-19503/com.jacee.examples.workmanager D/JTest: chain live combine: 0e1eb5a8-c3ed-4c35-8981-f63439169cf3: ENQUEUED
    2021-01-14 14:17:30.843 19503-19503/com.jacee.examples.workmanager D/JTest: chain live combine: 302f1da0-99ba-486e-b471-0d74fbc23394: SUCCEEDED
    2021-01-14 14:17:30.844 19503-19503/com.jacee.examples.workmanager D/JTest: chain live combine: 3e7b86fe-e3a8-4d74-92c0-5611fed39919: FAILED
    2021-01-14 14:17:30.844 19503-19503/com.jacee.examples.workmanager D/JTest: chain live combine: adc5049b-e4b2-4c35-a7a6-a5daca7a46f2: FAILED
    2021-01-14 14:17:30.844 19503-19503/com.jacee.examples.workmanager D/JTest: chain live combine: d7364e62-640c-4f70-b544-5c344725718c: FAILED
    2021-01-14 14:17:30.844 19503-19503/com.jacee.examples.workmanager D/JTest: chain live combine: ed4624ab-2feb-48d6-982c-0676687b8480: FAILED
    2021-01-14 14:17:30.844 19503-19503/com.jacee.examples.workmanager D/JTest: ----------------
    > 2完毕，SUCCEEDED，0e1eb5a8-c3ed-4c35-8981-f63439169cf3变为ENQUEUED，准备执行
    2021-01-14 14:17:30.844 19503-19667/com.jacee.examples.workmanager D/JTest: [1] doWork on 7387 started - 1610605050844  -> 0e1eb5a8-c3ed-4c35-8981-f63439169cf3
    2021-01-14 14:17:30.852 19503-19503/com.jacee.examples.workmanager D/JTest: chain live combine: 0e1eb5a8-c3ed-4c35-8981-f63439169cf3: RUNNING
    2021-01-14 14:17:30.852 19503-19503/com.jacee.examples.workmanager D/JTest: chain live combine: 302f1da0-99ba-486e-b471-0d74fbc23394: SUCCEEDED
    2021-01-14 14:17:30.852 19503-19503/com.jacee.examples.workmanager D/JTest: chain live combine: 3e7b86fe-e3a8-4d74-92c0-5611fed39919: FAILED
    2021-01-14 14:17:30.852 19503-19503/com.jacee.examples.workmanager D/JTest: chain live combine: adc5049b-e4b2-4c35-a7a6-a5daca7a46f2: FAILED
    2021-01-14 14:17:30.852 19503-19503/com.jacee.examples.workmanager D/JTest: chain live combine: d7364e62-640c-4f70-b544-5c344725718c: FAILED
    2021-01-14 14:17:30.852 19503-19503/com.jacee.examples.workmanager D/JTest: chain live combine: ed4624ab-2feb-48d6-982c-0676687b8480: FAILED
    2021-01-14 14:17:30.852 19503-19503/com.jacee.examples.workmanager D/JTest: ----------------
    > 原来0e1eb5a8-c3ed-4c35-8981-f63439169cf3即为1，开始执行，RUNNING
    2021-01-14 14:17:33.849 19503-19667/com.jacee.examples.workmanager D/JTest: [1] doWork on 7387 ended - 1610605053848
    2021-01-14 14:17:33.922 19503-19503/com.jacee.examples.workmanager D/JTest: chain live combine: 0e1eb5a8-c3ed-4c35-8981-f63439169cf3: SUCCEEDED
    2021-01-14 14:17:33.922 19503-19503/com.jacee.examples.workmanager D/JTest: chain live combine: 302f1da0-99ba-486e-b471-0d74fbc23394: SUCCEEDED
    2021-01-14 14:17:33.922 19503-19503/com.jacee.examples.workmanager D/JTest: chain live combine: 3e7b86fe-e3a8-4d74-92c0-5611fed39919: FAILED
    2021-01-14 14:17:33.922 19503-19503/com.jacee.examples.workmanager D/JTest: chain live combine: adc5049b-e4b2-4c35-a7a6-a5daca7a46f2: FAILED
    2021-01-14 14:17:33.923 19503-19503/com.jacee.examples.workmanager D/JTest: chain live combine: d7364e62-640c-4f70-b544-5c344725718c: FAILED
    2021-01-14 14:17:33.923 19503-19503/com.jacee.examples.workmanager D/JTest: chain live combine: ed4624ab-2feb-48d6-982c-0676687b8480: FAILED
    2021-01-14 14:17:33.923 19503-19503/com.jacee.examples.workmanager D/JTest: ----------------
    > 1执行完毕，SUCCEEDED。序列一完毕

从以上日志分析可以得出以下结论：

- 用then串联的WorkContinuation，保持“**一错毁所有**”原则，一旦执行失败，后续都失败
- combine的子链的failure会抛给整个combine，导致整个任务组合的failure
- combine的子链相互之间不会影响，即正常执行的子链，将完成全部任务

很好理解，子链的失败，和普通的链式任务一样，一旦失败，依赖它的”包裹任务“自然失败，那么`then`后的任务也必然失败。


## 唯一链

除了前面提到的两个启动链式任务的方法外，还有两个方法：

```java
public final @NonNull WorkContinuation beginUniqueWork(
        @NonNull String uniqueWorkName,
        @NonNull ExistingWorkPolicy existingWorkPolicy,
        @NonNull OneTimeWorkRequest work) {
    return beginUniqueWork(uniqueWorkName, existingWorkPolicy, Collections.singletonList(work));
}

public abstract @NonNull WorkContinuation beginUniqueWork(
        @NonNull String uniqueWorkName,
        @NonNull ExistingWorkPolicy existingWorkPolicy,
        @NonNull List<OneTimeWorkRequest> work);
```

顾名思义，`unique`就意味着，**用此方法启动的链式任务，在应用中是唯一的**。

`ExistingWorkPolicy`是一个枚举类，提供了当遭遇到workName冲突时的不同处理策略

```java
public enum ExistingWorkPolicy {
    REPLACE,            // 如果有未完成的同名work，取消并删除，添加新的
    KEEP,               // 如果有未完成的同名work，则继续它。如果没有，才添加新的
    APPEND,             // 如果有未完成的同名work，则添加新的作为它的后续。新的任务将遵行”一错毁所有“原则
    APPEND_OR_REPLACE,   // 如果有未完成的同名work，则添加新的作为它的后续。如果前置任务有失败的，那此新任务就自行独立出来作为一个新的链式头任务
}
```

现在使用REPLACE策略，连续启动两个同名唯一链。

```kotlin
WorkManager.getInstance(applicationContext)
   .beginUniqueWork(
       "a_unique_name_of_one",
       ExistingWorkPolicy.REPLACE,
       OneTimeWorkRequestBuilder<DelayWorker>()
           .setInputData(
               Data.Builder()
                   .putString(DelayWorker.ARG_NAME, "unique_one").build()
           )
           .build()
           .apply {
               WorkManager.getInstance(applicationContext).getWorkInfoByIdLiveData(id).observe({ lifecycle }) {
                   Log.d(TAG, "unique chain: ${it.id}: ${it.state}")
               }
           }
   ).enqueue()
```

执行结果：

    2021-01-14 15:23:43.165 21361-21411/com.jacee.examples.workmanager D/JTest: [unique_one] doWork on 7435 started - 1610609023165  -> d3b3a2f3-6372-43f6-a275-99e03ac055ae
    2021-01-14 15:23:43.165 21361-21361/com.jacee.examples.workmanager D/JTest: unique chain: d3b3a2f3-6372-43f6-a275-99e03ac055ae: ENQUEUED
    2021-01-14 15:23:43.168 21361-21361/com.jacee.examples.workmanager D/JTest: unique chain: d3b3a2f3-6372-43f6-a275-99e03ac055ae: RUNNING
    2021-01-14 15:23:44.012 21361-21410/com.jacee.examples.workmanager I/WM-WorkerWrapper: Work [ id=d3b3a2f3-6372-43f6-a275-99e03ac055ae, tags={ com.jacee.examples.workmanager.work.DelayWorker } ] was cancelled
        java.util.concurrent.CancellationException: Task was cancelled.
            at androidx.work.impl.utils.futures.AbstractFuture.cancellationExceptionWithCause(AbstractFuture.java:1184)
            at androidx.work.impl.utils.futures.AbstractFuture.getDoneValue(AbstractFuture.java:514)
            at androidx.work.impl.utils.futures.AbstractFuture.get(AbstractFuture.java:475)
            at androidx.work.impl.WorkerWrapper$2.run(WorkerWrapper.java:298)
            at androidx.work.impl.utils.SerialExecutor$Task.run(SerialExecutor.java:91)
            at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
            at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
            at java.lang.Thread.run(Thread.java:764)
    2021-01-14 15:23:44.029 21361-21413/com.jacee.examples.workmanager D/JTest: [unique_one] doWork on 7437 started - 1610609024029  -> cf6e62f3-cc2d-4049-9e8c-23262c097259
    2021-01-14 15:23:44.030 21361-21361/com.jacee.examples.workmanager D/AndroidRuntime: Shutting down VM


        --------- beginning of crash
    2021-01-14 15:23:44.031 21361-21361/com.jacee.examples.workmanager E/AndroidRuntime: FATAL EXCEPTION: main
        Process: com.jacee.examples.workmanager, PID: 21361
        java.lang.NullPointerException: it must not be null
            at com.jacee.examples.workmanager.WorkActivity$scheduleUniqueChain$1$2.onChanged(WorkActivity.kt:184)
            at com.jacee.examples.workmanager.WorkActivity$scheduleUniqueChain$1$2.onChanged(WorkActivity.kt:16)
            at androidx.lifecycle.LiveData.considerNotify(LiveData.java:131)
            at androidx.lifecycle.LiveData.dispatchingValue(LiveData.java:149)
            at androidx.lifecycle.LiveData.setValue(LiveData.java:307)
            at androidx.lifecycle.MutableLiveData.setValue(MutableLiveData.java:50)
            at androidx.lifecycle.LiveData$1.run(LiveData.java:91)
            at android.os.Handler.handleCallback(Handler.java:873)
            at android.os.Handler.dispatchMessage(Handler.java:99)
            at android.os.Looper.loop(Looper.java:207)
            at android.app.ActivityThread.main(ActivityThread.java:6878)
            at java.lang.reflect.Method.invoke(Native Method)
            at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:547)
            at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:876)
    2021-01-14 15:23:44.046 21361-21361/? I/Process: Sending signal. PID: 21361 SIG: 9


居然……crash了……

原因是，第二次启动任务时，因为是REPLACE，所以**第一个被cancel掉，但它不会收到一个CANCLED状态，而是直接收到一个空的WorkInfo**。所以崩溃来了。好吧，改改代码，并加更详细的标志信息

```kotlin
private var uniqueIndex = 0

// ...

WorkManager.getInstance(applicationContext)
    .beginUniqueWork(
        "a_unique_name_of_one",
        ExistingWorkPolicy.REPLACE,
        OneTimeWorkRequestBuilder<DelayWorker>()
            .setInputData(
                Data.Builder()
                    .putString(DelayWorker.ARG_NAME, "unique_${++uniqueIndex}").build()
            )
            .build()
            .apply {
                WorkManager.getInstance(applicationContext).getWorkInfoByIdLiveData(id).observe({ lifecycle }) {
                    Log.d(TAG, "unique chain: ${it?.id ?: return@observe}: ${it.state}")
                }
            }
    ).enqueue()
```

结果：

    2021-01-14 15:33:05.467 22097-22144/com.jacee.examples.workmanager D/JTest: [unique_1] doWork on 7447 started - 1610609585466  -> 73f6c63a-6358-42fd-b2bd-25c18f86e703
    2021-01-14 15:33:05.468 22097-22097/com.jacee.examples.workmanager D/JTest: unique chain: 73f6c63a-6358-42fd-b2bd-25c18f86e703: ENQUEUED
    2021-01-14 15:33:05.471 22097-22097/com.jacee.examples.workmanager D/JTest: unique chain: 73f6c63a-6358-42fd-b2bd-25c18f86e703: RUNNING
    2021-01-14 15:33:06.692 22097-22146/com.jacee.examples.workmanager D/JTest: [unique_2] doWork on 7449 started - 1610609586692  -> f455d1dd-c40c-414c-8450-3b6780466521
    2021-01-14 15:33:06.694 22097-22097/com.jacee.examples.workmanager D/JTest: unique chain: f455d1dd-c40c-414c-8450-3b6780466521: RUNNING
    2021-01-14 15:33:08.469 22097-22144/com.jacee.examples.workmanager D/JTest: [unique_1] doWork on 7447 ended - 1610609588469
    2021-01-14 15:33:09.694 22097-22146/com.jacee.examples.workmanager D/JTest: [unique_2] doWork on 7449 ended - 1610609589694
    2021-01-14 15:33:09.764 22097-22097/com.jacee.examples.workmanager D/JTest: unique chain: f455d1dd-c40c-414c-8450-3b6780466521: SUCCEEDED


虽然第一个唯一任务被取消了，但是**已经启动的运行过程还是正常执行完成了，只是其状态已经不被WorkManager所关注，所以没有状态回调**。

那么多于一个任务项的chain呢？

```kotlin
WorkManager.getInstance(applicationContext)
    .beginUniqueWork(
        "a_unique_name_of_two",
        ExistingWorkPolicy.REPLACE,
        OneTimeWorkRequestBuilder<DelayWorker>()
            .setInputData(
                Data.Builder()
                    .putString(DelayWorker.ARG_NAME, "unique_${++uniqueIndex}").build()
            )
            .build()
            .apply {
                WorkManager.getInstance(applicationContext).getWorkInfoByIdLiveData(id).observe({ lifecycle }) {
                    Log.d(TAG, "unique chain 1: ${it?.id ?: return@observe}: ${it.state}")
                }
            }
    )
    // 增加一个后续任务
    .then(OneTimeWorkRequestBuilder<DelayWorker>()
        .setInputData(
            Data.Builder()
                .putString(DelayWorker.ARG_NAME, "unique_${++uniqueIndex}").build()
        )
        .build()
        .apply {
            WorkManager.getInstance(applicationContext).getWorkInfoByIdLiveData(id).observe({ lifecycle }) {
                Log.d(TAG, "unique chain 2: ${it?.id ?: return@observe}: ${it.state}")
            }
        })
    .enqueue()
```


结果与分析:

    2021-01-14 15:38:37.347 22488-22488/com.jacee.examples.workmanager D/JTest: unique chain 2: 78bcce40-bba7-4b1b-9f19-d7932f5c5204: BLOCKED
    2021-01-14 15:38:37.356 22488-22530/com.jacee.examples.workmanager D/JTest: [unique_1] doWork on 7463 started - 1610609917356  -> 2e00761e-54f3-4812-9110-be1ca735c489
    2021-01-14 15:38:37.358 22488-22488/com.jacee.examples.workmanager D/JTest: unique chain 1: 2e00761e-54f3-4812-9110-be1ca735c489: ENQUEUED
    2021-01-14 15:38:37.363 22488-22488/com.jacee.examples.workmanager D/JTest: unique chain 1: 2e00761e-54f3-4812-9110-be1ca735c489: RUNNING
    2021-01-14 15:38:38.398 22488-22488/com.jacee.examples.workmanager D/JTest: unique chain 2: 50012173-523b-4a13-bcbe-f3ed5617a4f2: BLOCKED
    > 1准备然后执行，2阻塞
    2021-01-14 15:38:38.407 22488-22532/com.jacee.examples.workmanager D/JTest: [unique_3] doWork on 7465 started - 1610609918407  -> 56edd1ae-40b9-4bad-8e05-51b267a0eed0
    2021-01-14 15:38:38.415 22488-22488/com.jacee.examples.workmanager D/JTest: unique chain 1: 56edd1ae-40b9-4bad-8e05-51b267a0eed0: RUNNING
    > 启动了同名chain，3开始运行，1仍在执行，但是它及它的后续(2）被取消了
    2021-01-14 15:38:40.356 22488-22530/com.jacee.examples.workmanager D/JTest: [unique_1] doWork on 7463 ended - 1610609920356
    > 1执行结束（但是无状态更新，因为被取消）
    2021-01-14 15:38:41.408 22488-22532/com.jacee.examples.workmanager D/JTest: [unique_3] doWork on 7465 ended - 1610609921408
    2021-01-14 15:38:41.500 22488-22533/com.jacee.examples.workmanager D/JTest: [unique_4] doWork on 7466 started - 1610609921500  -> 50012173-523b-4a13-bcbe-f3ed5617a4f2
    2021-01-14 15:38:41.514 22488-22488/com.jacee.examples.workmanager D/JTest: unique chain 1: 56edd1ae-40b9-4bad-8e05-51b267a0eed0: SUCCEEDED
    2021-01-14 15:38:41.518 22488-22488/com.jacee.examples.workmanager D/JTest: unique chain 2: 50012173-523b-4a13-bcbe-f3ed5617a4f2: RUNNING
    > 3执行完成，SUCCEEDED，同时它的后续4开始执行，RUNNING
    2021-01-14 15:38:44.502 22488-22533/com.jacee.examples.workmanager D/JTest: [unique_4] doWork on 7466 ended - 1610609924502
    2021-01-14 15:38:44.556 22488-22488/com.jacee.examples.workmanager D/JTest: unique chain 2: 50012173-523b-4a13-bcbe-f3ed5617a4f2: SUCCEEDED
    > 4执行完成，SUCCEEDED


如日志分析，符合REPLACE的策略行为描述。改成APPEND试试：

    2021-01-14 15:54:31.431 23061-23061/com.jacee.examples.workmanager D/JTest: unique chain 2: 1235f8e0-ed41-46a2-91ce-5a2859873135: BLOCKED
    2021-01-14 15:54:31.440 23061-23120/com.jacee.examples.workmanager D/JTest: [unique_1] doWork on 7476 started - 1610610871439  -> 9375ba9d-fd74-4db9-a61e-7fa7ed27e0df
    2021-01-14 15:54:31.440 23061-23061/com.jacee.examples.workmanager D/JTest: unique chain 1: 9375ba9d-fd74-4db9-a61e-7fa7ed27e0df: ENQUEUED
    2021-01-14 15:54:31.445 23061-23061/com.jacee.examples.workmanager D/JTest: unique chain 1: 9375ba9d-fd74-4db9-a61e-7fa7ed27e0df: RUNNING
    2021-01-14 15:54:33.015 23061-23061/com.jacee.examples.workmanager D/JTest: unique chain 2: 3e6cbab8-b7d7-42c2-a956-24022ce63451: BLOCKED
    2021-01-14 15:54:33.022 23061-23061/com.jacee.examples.workmanager D/JTest: unique chain 1: d291b741-ba8f-4cdc-8811-b7538f274f1f: BLOCKED
    2021-01-14 15:54:34.442 23061-23120/com.jacee.examples.workmanager D/JTest: [unique_1] doWork on 7476 ended - 1610610874442
    2021-01-14 15:54:34.533 23061-23128/com.jacee.examples.workmanager D/JTest: [unique_2] doWork on 7478 started - 1610610874533  -> 1235f8e0-ed41-46a2-91ce-5a2859873135
    2021-01-14 15:54:34.536 23061-23061/com.jacee.examples.workmanager D/JTest: unique chain 1: 9375ba9d-fd74-4db9-a61e-7fa7ed27e0df: SUCCEEDED
    2021-01-14 15:54:34.541 23061-23061/com.jacee.examples.workmanager D/JTest: unique chain 2: 1235f8e0-ed41-46a2-91ce-5a2859873135: RUNNING
    2021-01-14 15:54:37.534 23061-23128/com.jacee.examples.workmanager D/JTest: [unique_2] doWork on 7478 ended - 1610610877534
    2021-01-14 15:54:37.608 23061-23142/com.jacee.examples.workmanager D/JTest: [unique_3] doWork on 7479 started - 1610610877608  -> d291b741-ba8f-4cdc-8811-b7538f274f1f
    2021-01-14 15:54:37.616 23061-23061/com.jacee.examples.workmanager D/JTest: unique chain 2: 1235f8e0-ed41-46a2-91ce-5a2859873135: SUCCEEDED
    2021-01-14 15:54:37.620 23061-23061/com.jacee.examples.workmanager D/JTest: unique chain 1: d291b741-ba8f-4cdc-8811-b7538f274f1f: RUNNING
    2021-01-14 15:54:40.611 23061-23142/com.jacee.examples.workmanager D/JTest: [unique_3] doWork on 7479 ended - 1610610880611
    2021-01-14 15:54:40.704 23061-23143/com.jacee.examples.workmanager D/JTest: [unique_4] doWork on 7480 started - 1610610880704  -> 3e6cbab8-b7d7-42c2-a956-24022ce63451
    2021-01-14 15:54:40.716 23061-23061/com.jacee.examples.workmanager D/JTest: unique chain 1: d291b741-ba8f-4cdc-8811-b7538f274f1f: SUCCEEDED
    2021-01-14 15:54:40.720 23061-23061/com.jacee.examples.workmanager D/JTest: unique chain 2: 3e6cbab8-b7d7-42c2-a956-24022ce63451: RUNNING
    2021-01-14 15:54:43.709 23061-23143/com.jacee.examples.workmanager D/JTest: [unique_4] doWork on 7480 ended - 1610610883709
    2021-01-14 15:54:43.798 23061-23061/com.jacee.examples.workmanager D/JTest: unique chain 2: 3e6cbab8-b7d7-42c2-a956-24022ce63451: SUCCEEDED

可以看到，第二次的chain（3->4）按序在1->2后执行了。

### 唯一链的监听

上面的监听其实略复杂，`WorkManager`有提供针对唯一链的监听，也是之前讨论监听的时候忽略掉的，因为当时还没用上唯一链。

调用`getWorkInfosForUniqueWorkLiveData`统一监听，KEEP策略下，快速启动两次同名唯一链：

```kotlin
val name = "a_unique_name_of_three"
WorkManager.getInstance(applicationContext).getWorkInfosForUniqueWorkLiveData(name).observe({ lifecycle }) { l ->
    l.forEach {
        Log.d(TAG, "unique chain: ${it.id}: ${it.state}")
    }
}
WorkManager.getInstance(applicationContext)
    .beginUniqueWork(
        name,
        ExistingWorkPolicy.KEEP,
        OneTimeWorkRequestBuilder<DelayWorker>()
            .setInputData(
                Data.Builder()
                    .putString(DelayWorker.ARG_NAME, "unique_${++uniqueIndex}").build()
            )
            .build()
    )
    .then(
        OneTimeWorkRequestBuilder<DelayWorker>()
            .setInputData(
                Data.Builder()
                    .putString(DelayWorker.ARG_NAME, "unique_${++uniqueIndex}").build()
            )
            .build()
    )
    .enqueue()
```

结果：

    2021-01-14 16:02:32.788 23465-23515/com.jacee.examples.workmanager D/JTest: [unique_1] doWork on 7483 started - 1610611352788  -> 809b9e10-3ed6-4f62-a177-ef463582c345
    2021-01-14 16:02:32.805 23465-23465/com.jacee.examples.workmanager D/JTest: unique chain: 809b9e10-3ed6-4f62-a177-ef463582c345: RUNNING
    2021-01-14 16:02:32.805 23465-23465/com.jacee.examples.workmanager D/JTest: unique chain: 84b1e479-5c39-46a0-b23c-1aff5228364a: BLOCKED
    2021-01-14 16:02:33.738 23465-23465/com.jacee.examples.workmanager D/JTest: unique chain: 809b9e10-3ed6-4f62-a177-ef463582c345: RUNNING
    2021-01-14 16:02:33.739 23465-23465/com.jacee.examples.workmanager D/JTest: unique chain: 84b1e479-5c39-46a0-b23c-1aff5228364a: BLOCKED
    2021-01-14 16:02:35.793 23465-23515/com.jacee.examples.workmanager D/JTest: [unique_1] doWork on 7483 ended - 1610611355792
    2021-01-14 16:02:35.896 23465-23518/com.jacee.examples.workmanager D/JTest: [unique_2] doWork on 7485 started - 1610611355896  -> 84b1e479-5c39-46a0-b23c-1aff5228364a
    2021-01-14 16:02:35.899 23465-23465/com.jacee.examples.workmanager D/JTest: unique chain: 809b9e10-3ed6-4f62-a177-ef463582c345: SUCCEEDED
    2021-01-14 16:02:35.899 23465-23465/com.jacee.examples.workmanager D/JTest: unique chain: 84b1e479-5c39-46a0-b23c-1aff5228364a: ENQUEUED
    2021-01-14 16:02:35.901 23465-23465/com.jacee.examples.workmanager D/JTest: unique chain: 809b9e10-3ed6-4f62-a177-ef463582c345: SUCCEEDED
    2021-01-14 16:02:35.901 23465-23465/com.jacee.examples.workmanager D/JTest: unique chain: 84b1e479-5c39-46a0-b23c-1aff5228364a: RUNNING
    2021-01-14 16:02:35.909 23465-23465/com.jacee.examples.workmanager D/JTest: unique chain: 809b9e10-3ed6-4f62-a177-ef463582c345: SUCCEEDED
    2021-01-14 16:02:35.909 23465-23465/com.jacee.examples.workmanager D/JTest: unique chain: 84b1e479-5c39-46a0-b23c-1aff5228364a: RUNNING
    2021-01-14 16:02:38.897 23465-23518/com.jacee.examples.workmanager D/JTest: [unique_2] doWork on 7485 ended - 1610611358897
    2021-01-14 16:02:38.943 23465-23465/com.jacee.examples.workmanager D/JTest: unique chain: 809b9e10-3ed6-4f62-a177-ef463582c345: SUCCEEDED
    2021-01-14 16:02:38.943 23465-23465/com.jacee.examples.workmanager D/JTest: unique chain: 84b1e479-5c39-46a0-b23c-1aff5228364a: SUCCEEDED
    2021-01-14 16:02:38.953 23465-23465/com.jacee.examples.workmanager D/JTest: unique chain: 809b9e10-3ed6-4f62-a177-ef463582c345: SUCCEEDED
    2021-01-14 16:02:38.953 23465-23465/com.jacee.examples.workmanager D/JTest: unique chain: 84b1e479-5c39-46a0-b23c-1aff5228364a: SUCCEEDED

如预期，KEEP模式下，原来任务继续执行至完毕。

### 非链式唯一性任务

别以为只有链式任务才有unique版本，一般任务也是有的，这里就顺带说一下

```java
@NonNull
public Operation enqueueUniqueWork(
        @NonNull String uniqueWorkName,
        @NonNull ExistingWorkPolicy existingWorkPolicy,
        @NonNull OneTimeWorkRequest work) {
    return enqueueUniqueWork(
            uniqueWorkName,
            existingWorkPolicy,
            Collections.singletonList(work));
}

@NonNull
public abstract Operation enqueueUniqueWork(
        @NonNull String uniqueWorkName,
        @NonNull ExistingWorkPolicy existingWorkPolicy,
        @NonNull List<OneTimeWorkRequest> work);

public abstract Operation enqueueUniquePeriodicWork(
        @NonNull String uniqueWorkName,
        @NonNull ExistingPeriodicWorkPolicy existingPeriodicWorkPolicy,
        @NonNull PeriodicWorkRequest periodicWork);
```

单次任务、周期性任务，都可以添加唯一性任务。注意，周期性任务的策略是周期性策略`ExistingPeriodicWorkPolicy`，仅支持REPLACE和KEEP

```java
public enum ExistingPeriodicWorkPolicy {

    /**
     * If there is existing pending (uncompleted) work with the same unique name, cancel and delete
     * it.  Then, insert the newly-specified work.
     */
    REPLACE,

    /**
     * If there is existing pending (uncompleted) work with the same unique name, do nothing.
     * Otherwise, insert the newly-specified work.
     */
    KEEP
}

```

## 小结

链式任务给任务的组合提供了便捷的API，不仅可以顺序连接任务，还能同时执行任务（combine就是干这事的）。而唯一链更提供了复杂的控制策略，使得任务执行的自定义更丰富强大。


