---
title: WorkManager
date: 2018-11-15 10:31:13
tags:
    - Android
    - WorkManager
---

#作用
WorkManager API可以轻松指定可延迟的异步任务以及何时运行它们。这些API允许您创建任务并将其交给WorkManager立即运行或在适当的时间运行。

#引入

```gradle
dependencies {
    def work_version = "1.0.0-alpha11"

    implementation "android.arch.work:work-runtime:$work_version" // use -ktx for Kotlin

    // optional - Firebase JobDispatcher support
    implementation "android.arch.work:work-firebase:$work_version"

    // optional - Test helpers
    androidTestImplementation "android.arch.work:work-testing:$work_version"
}
```

#基本用法

##1、Work类
Worker 是一个抽象类，用来指定需要执行的具体任务。我们需要继承 Worker 类，并实现它的 doWork 方法：

```kotlin
class MyWorker:Worker() {

    val tag = javaClass.simpleName

   override fun onStopped(cancelled: Boolean) {
       super.onStopped(cancelled)
       //当任务结束时会回调这里
       ...
   }

    override fun doWork(): Result {

        Log.d(tag,"任务执行完毕！")
        return Worker.Result.SUCCESS
    }
}
```

##2、WorkRequest类
也是一个抽象类，可以对 Work 进行包装，同时装裱上一系列的约束（Constraints），这些 Constraints 用来向系统指明什么条件下，或者什么时候开始执行任务。

WorkManager 向我们提供了 WorkRequest 的两个子类：

- OneTimeWorkRequest：单次任务

- PeriodicWorkRequest：周期任务

```kotlin
val request1 = PeriodicWorkRequestBuilder<MyWorker>(60,TimeUnit.SECONDS).build()

val request2 = OneTimeWorkRequestBuilder<MyWorker>().build()

```

##3、Constraints类
指定对任务运行时间的限制（例如，“仅在连接到网络时”）

```kotlin
val myConstraints = Constraints.Builder()
        .setRequiresDeviceIdle(true)//指定{@link WorkRequest}运行时设备是否为空闲
        .setRequiresCharging(true)//指定要运行的{@link WorkRequest}是否应该插入设备
        .setRequiredNetworkType(NetworkType.NOT_ROAMING)
        .setRequiresBatteryNotLow(true)//指定设备电池是否不应低于临界阈值
        .setRequiresCharging(true)//网络状态
        .setRequiresDeviceIdle(true)//指定{@link WorkRequest}运行时设备是否为空闲
        .setRequiresStorageNotLow(true)//指定设备可用存储是否不应低于临界阈值
        .build()
val request = PeriodicWorkRequestBuilder<MyWorker>(24,TimeUnit.SECONDS)
        .setConstraints(myConstraints)//注意看这里！！！
        .build()
        
```
##4、添加标签
我们可以给两个相同任务的request都加上了标签，使他们成为了一个组：A组。这样的好处是以后可以直接控制整个组就行了，组内的每个成员都会受到影响。也可以为一个request添加多个标签。

```kotlin
val request1 = OneTimeWorkRequestBuilder<MyWorker>()
                .addTag("A")//标签
                .build()
val request2 = OneTimeWorkRequestBuilder<MyWorker>()
                .addTag("A")//标签
                .addTag("B")//标签
                .build()
                
```

##5、向任务添加参数

在Request中传参

```kotlin
val data=Data.Builder()
        .putInt("A",1)
        .putString("B","2")
        .build()
val request2 = PeriodicWorkRequestBuilder<MyWorker>(24,TimeUnit.SECONDS)
        .setInputData(data)
        .build()
```
在 Worker 中使用

```kotlin
class MyWorker:Worker() {

    val tag = javaClass.simpleName

    override fun doWork(): Result {

        val A = inputData.getInt("A",0)
        val B = inputData.getString("B")
        return Worker.Result.SUCCESS
    }
}
```
##6、任务的返回值
很类似很类似的，任务的返回值也很简单：

```kotlin
override fun doWork(): Result {

    val A = inputData.getInt("A",0)
    val B = inputData.getString("B")

    val data = Data.Builder()
            .putBoolean("C",true)
            .putFloat("D",0f)
            .build()
    outputData = data//返回值
    return Worker.Result.SUCCESS
}

```
doWork 要求最后返回一个 Result，这个 Result 是一个枚举，它有几个固定的值：

- FAILURE 任务失败。

- RETRY 遇到暂时性失败，此时可使用
- WorkRequest.Builder.setBackoffCriteria(BackoffPolicy, long, TimeUnit)来重试。
- SUCCESS 任务成功。

##7、WorkManager类
经过上面的操作，相信我们已经能够成功创建 request 了，接下来我们就需要把任务放进任务队列，我们使用 WorkManager。

WorkManager 是个单例，它负责调度任务并且监听任务状态。

```kotlin
WorkManager.getInstance().enqueue(request)
```
当我们的 request 入列后，WorkManager 会给它分配一个 work ID，之后我们可以使用这个work id 来取消或者停止任务

```kotlin
WorkManager.getInstance().cancelWorkById(request.id)
```

注意：WorkManager 并不一定能结束任务，因为任务有可能已经执行完毕了。

同时，WorkManager 还提供了其他结束任务的方法：

- cancelAllWork():取消所有任务

- cancelAllWorkByTag(tag:String):取消一组带有相同标签的任务

- cancelUniqueWork(uniqueWorkName:String):取消唯一任务

##8、WorkStatus类
当 WorkManager 把任务加入队列后，会为每个WorkRequest对象提供一个 LiveData。 LiveData 持有 WorkStatus;通过观察该 LiveData, 我们可以确定任务的当前状态, 并在任务完成后获取所有返回的值。

获取request状态的方式：

- getStatusById(UUID)

```kotlin
val liveData: LiveData<WorkStatus> = WorkManager.getInstance().getStatusById(request.id)
```
- getStatusesByTag(String)

```kotlin
val liveData: LiveData<List<WorkStatus>> = WorkManager.getInstance().getStatusesByTag(tag)
```
- getStatusesForUniqueWork(String)

```kotlin
val liveData: LiveData<List<WorkStatus>> = WorkManager.getInstance().getStatusesByTag(uniqueWorkName)
```
 WorkStatus 到底都包涵什么，可以看看它的源码：
 
```kotlin
    private @NonNull UUID mId;
    private @NonNull State mState;
    private @NonNull Data mOutputData;
    private @NonNull Set<String> mTags;
```
##10、任务链
有时候我们想让应用程序按照特定的顺序运行多个任务。 WorkManager允许我们创建和排队多个任务的工作序列，以及它们应该以什么顺序运行。

例如，假如我们的应用有三个 OneTimeWorkRequest 对象：workA, workB, 和 workC，这些任务必须按照该顺序执行,要想将它们排队;

第一步：请使用WorkManager.beginWith() 方法创建一个序列，并传递第一个OneTimeWorkRequest对象，该方法返回一个WorkContinuation对象，该对象定义了一系列任务。

第二步：然后依次使用 WorkContinuation.then()添加剩余的OneTimeWorkRequest对象，最后使用 WorkContinuation.enqueue()排序整个序列：

例1：workA 、B 、C顺序执行


```kotlin
WorkManager.getInstance()
    .beginWith(workA)
        // Note: WorkManager.beginWith() returns a
        // WorkContinuation object; the following calls are
        // to WorkContinuation methods
    .then(workB)    // FYI, then() returns a new WorkContinuation instance
    .then(workC)
    .enqueue()
```

例2：workA1、A2、A3同时开始执行,然后执行 B 和 C

```kotlin
WorkManager.getInstance()
    // First, run all the A tasks (in parallel):
    .beginWith(workA1, workA2, workA3)
    // ...when all A tasks are finished, run the single B task:
    .then(workB)
    // ...then run the C tasks (in any order):
    .then(workC)
    .enqueue()
```
例3：workA 和 B 顺序链式执行，C 和 D 顺序链式执行，且他们同步进行，他们执行完之后，执行 E

```kotlin
val chain1 = WorkManager.getInstance()
    .beginWith(workA)
    .then(workB)
val chain2 = WorkManager.getInstance()
    .beginWith(workC)
    .then(workD)
val chain3 = WorkContinuation
    .combine(chain1, chain2)
    .then(workE)
chain3.enqueue()
```




















