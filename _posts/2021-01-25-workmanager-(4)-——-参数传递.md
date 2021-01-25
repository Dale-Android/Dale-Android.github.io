---
layout: post
title: "WorkManager (4) —— 参数传递"
date: "2021-01-25 17:02:40 +0800"
description: 
img: 
tags: [Android, WorkManager]
mathjax: true
---

单次任务和周期任务的创建和执行，现在已经清楚了。但是有问题，之前创建的Worker就是一个单纯的Worker，和任务添加者是没有关系的，如果需要传递参数，应该怎么做？

`WorkRequest.Builder.setInputData(Data)`就是我们要找的东西。

## Data

参数传递使用了一个叫`Data`的类

```java
/**
 * A persistable set of key/value pairs which are used as inputs and outputs for
 * {@link ListenableWorker}s.  Keys are Strings, and values can be Strings, primitive types, or
 * their array variants.
 * <p>
 * This is a lightweight container, and should not be considered your data store.  As such, there is
 * an enforced {@link #MAX_DATA_BYTES} limit on the serialized (byte array) size of the payloads.
 * This class will throw {@link IllegalStateException}s if you try to serialize or deserialize past
 * this limit.
 */

public final class Data {
    // .....
    public static final class Builder {
    // .....
    }
}
```

Data是一个**轻量级的键值对数据类**，键为String，值可以是*String*、*原始类型*以及*它们的数组*。如果是其他类型，就会有异常。

从下面代码就能看出：

```java
public @NonNull Builder put(@NonNull String key, @Nullable Object value) {
    if (value == null) {
        mValues.put(key, null);
    } else {
        Class<?> valueType = value.getClass();
        if (valueType == Boolean.class
                || valueType == Byte.class
                || valueType == Integer.class
                || valueType == Long.class
                || valueType == Float.class
                || valueType == Double.class
                || valueType == String.class
                || valueType == Boolean[].class
                || valueType == Byte[].class
                || valueType == Integer[].class
                || valueType == Long[].class
                || valueType == Float[].class
                || valueType == Double[].class
                || valueType == String[].class) {
            mValues.put(key, value);
        } else if (valueType == boolean[].class) {
            mValues.put(key, convertPrimitiveBooleanArray((boolean[]) value));
        } else if (valueType == byte[].class) {
            mValues.put(key, convertPrimitiveByteArray((byte[]) value));
        } else if (valueType == int[].class) {
            mValues.put(key, convertPrimitiveIntArray((int[]) value));
        } else if (valueType == long[].class) {
            mValues.put(key, convertPrimitiveLongArray((long[]) value));
        } else if (valueType == float[].class) {
            mValues.put(key, convertPrimitiveFloatArray((float[]) value));
        } else if (valueType == double[].class) {
            mValues.put(key, convertPrimitiveDoubleArray((double[]) value));
        } else {
            throw new IllegalArgumentException(
                    String.format("Key %s has invalid type %s", key, valueType));
        }
    }
    return this;
}
```

而且传递的数据大小也有限制，最大为`MAX_DATA_BYTES（10K）`


## 参数传递

有了Data的知识，现在就传递一个名称给DelayWorker，作为一个描述字符串。

### 读取

**DelayWorker**新增一个名称参数，并在任务里面读取并打印出来

```kotlin
class DelayWorker(context: Context, workerParams: WorkerParameters) : Worker(context, workerParams) {

    override fun doWork(): Result {
        val name = inputData.getString(ARG_NAME) // 获取名称，并在日志里打印出来
        Log.d(TAG, "[$name] doWork on ${Thread.currentThread().id} started - ${System.currentTimeMillis()}  -> $id")
        // emulated work
        Thread.sleep(3000)
        Log.d(TAG, "[$name] doWork on ${Thread.currentThread().id} ended - ${System.currentTimeMillis()}")
        return Result.success()
    }

    companion object {
        const val ARG_NAME = "name" // 名称参数
    }
}
```

### 写入

如前文所说，写入参数当然使用`Worker.Builder`的方法：

```kotlin
val request = OneTimeWorkRequestBuilder<DelayWorker>()
    .setInputData(
        Data.Builder()
            .putString(DelayWorker.ARG_NAME, "ONE-TIME")
            .build()
    )
    .build()
```

执行结果：

    2021-01-14 10:01:34.920 12531-12531/com.jacee.examples.workmanager D/JTest: enqueue on 2 1610589694920
    2021-01-14 10:01:34.961 12531-12570/com.jacee.examples.workmanager D/JTest: [ONE-TIME] doWork on 7172 started - 1610589694961  -> 8aa7a2ee-af7f-47ad-9f17-b898268dd487
    2021-01-14 10:01:34.962 12531-12531/com.jacee.examples.workmanager D/JTest: onetime: 8aa7a2ee-af7f-47ad-9f17-b898268dd487: ENQUEUED
    2021-01-14 10:01:34.966 12531-12531/com.jacee.examples.workmanager D/JTest: onetime: 8aa7a2ee-af7f-47ad-9f17-b898268dd487: RUNNING
    2021-01-14 10:01:37.965 12531-12570/com.jacee.examples.workmanager D/JTest: [ONE-TIME] doWork on 7172 ended - 1610589697965
    2021-01-14 10:01:38.024 12531-12531/com.jacee.examples.workmanager D/JTest: onetime: 8aa7a2ee-af7f-47ad-9f17-b898268dd487: SUCCEEDED

成功实现名称传递。


## 小结

今天的内容很简单，但又很重要 —— 带参的任务，一定不会是个低概率需求。



