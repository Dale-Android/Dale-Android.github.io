---
layout: post
title: "初识WorkManager"
date: "2021-01-09 15:48:58 +0800"
description: 
img: 
tags: [Android, WorkManager]
mathjax: true
---


好早之前，项目中有个功能项需要创建一个下载任务，考虑到和界面的无依赖性，所以我选用了`WorkManager`。在当时来看，WorkManager还算是Android的一个新技术。而那段项目代码，也是简单的使用而已。

今天，我们从头再来重新认识并学习一下WorkManager。


## 简介

> WorkManager is an API that makes it easy to schedule deferrable, asynchronous tasks that are expected to run even if the app exits or the device restarts. 

上面是官方关于`WorkManager`的一句简要说明，从中可以提出几个要点：

- WorkManager可以轻松地生成定时、异步的后台任务
- 即使应用退出，甚至系统重启后，后台任务仍可以执行

WorkManger最小可兼容到API14，是以替代像JobScheduler、AlarmManger等定时任务发布类而存在的，也是官方推荐使用的

## 基本使用

### 依赖

首先，在相应的module的build里，添加必要的库依赖

```gradle

dependencies {

    def work_version = "2.4.0"
    // ....
    implementation "androidx.work:work-runtime-ktx:$work_version"
    // ....
}
```

因为是kotlin工程，所以这里添加的是`work-runtime-ktx`，Java工程则应该用`work-runtime`


### 实现Worker

> `androidx.work.Worker`
>
> A class that performs work synchronously on a background thread provided by WorkManager

Worker就是我们要实现的用于添加任务代码的抽象类

```java
public abstract class Worker extends ListenableWorker {

    // Package-private to avoid synthetic accessor.
    SettableFuture<Result> mFuture;

    @Keep
    @SuppressLint("BanKeepAnnotation")
    public Worker(@NonNull Context context, @NonNull WorkerParameters workerParams) {
        super(context, workerParams);
    }

    /**
     * Override this method to do your actual background processing.  This method is called on a
     * background thread - you are required to <b>synchronously</b> do your work and return the
     * {@link androidx.work.ListenableWorker.Result} from this method.  Once you return from this
     * method, the Worker is considered to have finished what its doing and will be destroyed.  If
     * you need to do your work asynchronously on a thread of your own choice, see
     * {@link ListenableWorker}.
     * <p>
     * A Worker is given a maximum of ten minutes to finish its execution and return a
     * {@link androidx.work.ListenableWorker.Result}.  After this time has expired, the Worker will
     * be signalled to stop.
     *
     * @return The {@link androidx.work.ListenableWorker.Result} of the computation; note that
     *         dependent work will not execute if you use
     *         {@link androidx.work.ListenableWorker.Result#failure()} or
     *         {@link androidx.work.ListenableWorker.Result#failure(Data)}
     */
    @WorkerThread
    public abstract @NonNull Result doWork();

    @Override
    public final @NonNull ListenableFuture<Result> startWork() {
        mFuture = SettableFuture.create();
        getBackgroundExecutor().execute(new Runnable() {
            @Override
            public void run() {
                try {
                    Result result = doWork();
                    mFuture.set(result);
                } catch (Throwable throwable) {
                    mFuture.setException(throwable);
                }

            }
        });
        return mFuture;
    }
}
```

复写`doWork`方法，并在其中添加实际任务代码即可。

这里来一个假任务，作为测试：

```kotlin
class DelayWorker(context: Context, workerParams: WorkerParameters) : Worker(context, workerParams) {

    override fun doWork(): Result {
        Log.d(TAG, "doWork on ${Thread.currentThread().id} started - ${System.currentTimeMillis()}")
        // emulated work
        Thread.sleep(3000)
        Log.d(TAG, "doWork on ${Thread.currentThread().id} ended - ${System.currentTimeMillis()}")
        return Result.success()
    }
}
```

### WorkRequest

任务建好后，需要一个启动任务的“请求类”，如果需要的话，还可以添加参数 —— `WorkRequest`上场了。

```java
public abstract class WorkRequest {
    // ....

    public abstract static class Builder<B extends Builder<?, ?>, W extends WorkRequest> {
        // ...
    }

    // ....
}
```

可以看出，构造WorkRequest，使用的是Builder模型；但WorkRequest和它的builder都是抽象类啊，别慌，有抽象，就有实现。

WorkRequest有两个实现：`OneTimeWorkRequest`和`PeriodicWorkRequest`。望文生义，前者用于非重复的单个任务，而后者用于周期重复性任务。

下面来构造一个非重复任务：

```kotlin
val request = OneTimeWorkRequestBuilder<DelayWorker>().build()
```

很简单，这里用了ktx的扩展，实际上就是用了`OneTimeWorkRequest.Builder`来build的。

```kotlin
/**
 * Creates a [OneTimeWorkRequest] with the given [ListenableWorker].
 */
inline fun <reified W : ListenableWorker> OneTimeWorkRequestBuilder() =
        OneTimeWorkRequest.Builder(W::class.java)
```


### 启动任务

worker和request都有了，接下来就要启动任务了，`WorkManager`上场：

```kotlin
Log.d(TAG, "enqueue on ${Thread.currentThread().id} ${System.currentTimeMillis()}")
WorkManager.getInstance(applicationContext).enqueue(request)
```

WorkManager以单例形式存在，调用`enqueue`，添加前面创建的request即可。


### 执行结果

上述任务的执行日志：

    2021-01-09 15:46:43.437 15929-15929/com.jacee.examples.workmanager D/JTest: enqueue on 2 1610178403437
    2021-01-09 15:46:43.513 15929-15969/com.jacee.examples.workmanager D/JTest: doWork on 4754 started - 1610178403513
    2021-01-09 15:46:46.515 15929-15969/com.jacee.examples.workmanager D/JTest: doWork on 4754 ended - 1610178406514

任务成功异常地在子线程执行了，并耗时3秒（当然，这里是假任务）。


## 结语

前面的测试代码，算是帮我们基本认识了WorkManager，也大概了解了它的基本使用。有时间，咱再来学习WorkManager的其他功能。
