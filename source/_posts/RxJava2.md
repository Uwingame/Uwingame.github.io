---
title: RxJava2
date: 2018-01-10 10:31:13
tags:
    - Android
    - RxJava
---

原文地址：<https://www.jianshu.com/p/464fa025229e>

# 订阅关系

观察者和被观察者可以用两根水管的关系来表示

![Rxjava关系](image/rxjava_1.png)

产生事件的水管称为 **上游**，接收事件的水管称为 **下游**，对应到代码，上游为 **Observable**，下游为 **Observer**，他们之间的连接为 **subscribe()**。
创建上游的时候，我们使用 **发射器** 来发送事件，使用 **onNext()** 发送单个事件，**onComplete()** 表示发送完成，**onError()** 表示出现错误。

``` java
Observable<Integer> observable = Observable.create(new ObservableOnSubscribe<Integer>() {
    @Override
    public void subscribe(ObservableEmitter<Integer> emitter) throws Exception {
        emitter.onNext(1);
        emitter.onNext(2);
        emitter.onNext(3);
        emitter.onComplete();
    }
});

Observer<Integer> observer = new Observer<Integer>() {
    @Override
    public void onSubscribe(Disposable disposable) {
    }

    @Override
    public void onNext(Integer o) {
    }

    @Override
    public void onError(Throwable throwable) {
    }

    @Override
    public void onComplete() {
    }
};

observable.subscribe(observer);
```

需要注意以下几点：
 - 上游可以发送无限个 **onNext**
 - **onComplete** 和 **onError** 唯一并且互斥，而且这只能由我们自己去保证
 - 上游可以在 **onComplete** 和 **onError** 之后继续发送 **onNext**, 下游收到 **onComplete** 或者 **onError** 后将不再接收 **onNext** 事件
 - 可以不发送 **onComplete** 和 **onError**
 - 发送多个 **onComplete** 程序只会接收第一个，之后还能正常运行，发送多个 **onError** 会导致程序崩溃

subscribe()有多个重载方法

``` java
public final Disposable subscribe() {}
public final Disposable subscribe(Consumer<? super T> onNext) {}
public final Disposable subscribe(Consumer<? super T> onNext, Consumer<? super Throwable> onError) {} 
public final Disposable subscribe(Consumer<? super T> onNext, Consumer<? super Throwable> onError, Action onComplete) {}
public final Disposable subscribe(Consumer<? super T> onNext, Consumer<? super Throwable> onError, Action onComplete, Consumer<? super Disposable> onSubscribe) {}
public final void subscribe(Observer<? super T> observer) {}
```

# 线程切换

RxJava的线程分为两种：

 - 上游发送事件的线程
 - 下游接收事件的线程

需要注意的有：

 - 上游线程一旦指定后不会改变，**subscribe()** 只有第一次调用有小，之后无效
 - 下游线程可以切换

我们最关心的是代码在哪个线程里面运行，可以这样来判断，首先是 **Observable.create** 时 **subscribe** 的代码运行在第一次 **subscribeOn** 指定的线程。之后的代码运行在 **observeOn** 指定的线程，每执行一次 **observeOn** ，线程就切换一次。

首先是RxJava链式函数调用顺序，实际顺序和下面代码的调用顺序一样

``` kotlin
Observable.create<Int> {
    Log.d(TAG, "create")
    it.onNext(1)
    it.onComplete()
}.doOnSubscribe {}
        .doOnNext {}
        .doAfterNext {}
        .doOnTerminate {}
        .doOnError {}
        .doOnComplete {}
        .doAfterTerminate {}
        .subscribe({}, {})
```

RxJava调用流程如下

![RxJava调用流程](image/rxjava_2.png)

##　subscribe 
 
运行在距离自己最近的 subscribeOn 指定的线程，就是所有链式调用中第一个 subscribeOn 指定的线程，map 和 flatMap 和 subscribe 运行在同一个线程

## doOnSubscribe 

由之后的第一个 subscribeOn 指定线程，而且多个 doOnSubscribe 调用顺序和在链里面的顺序是反的，如果 doOnSubscribe 后面没有指定任何线程，会使用调用 subscribe 的线程

 ``` kotlin
 // 在主线程中调用
Observable.create<Int> {
    printName("subscribe")
    it.onNext(1)
    it.onComplete()
}
        .subscribeOn(AndroidSchedulers.mainThread())
        .observeOn(AndroidSchedulers.mainThread())
        .doOnSubscribe { printName("doOnSubscribe1") } // 在io线程中
        .subscribeOn(Schedulers.io())
        .doOnSubscribe { printName("doOnSubscribe2") } // 在main线程中
        .subscribeOn(AndroidSchedulers.mainThread())
        .doOnSubscribe { printName("doOnSubscribe3") } // 使用默认线程
        .subscribe()
 ```

输出结果：

```
doOnSubscribe3 on thread main
doOnSubscribe2 on thread main
doOnSubscribe1 on thread RxCachedThreadScheduler-1
subscribe on thread main
```

## doOnNext doAfterNext doOnTerminate doAfterTerminate onNext onError...

这些函数执行的线程由 observeOn 指定， 可以这样认为，每执行一次observeOn，链条之后的这类函数的线程切换一次

``` kotlin
Observable.create<Int> {
    printName("subscribe")
    it.onNext(1)
    it.onComplete()
}
        .subscribeOn(AndroidSchedulers.mainThread())
        .observeOn(Schedulers.io())

        // 切换为io线程
        .doOnNext { printName("doOnNext1") }
        .doAfterNext { printName("doAfterNext1") }
        .doOnTerminate { printName("doOnTerminate1") }
        .doAfterTerminate { printName("doAfterTerminate1") }

        // 切换为主线程
        .observeOn(AndroidSchedulers.mainThread())
        .doOnNext { printName("doOnNext2") }
        .doAfterNext { printName("doAfterNext2") }
        .doOnTerminate { printName("doOnTerminate2") }
        .doAfterTerminate { printName("doAfterTerminate3") }

        // 切换为io线程
        .observeOn(Schedulers.io())
        .doOnNext { printName("doOnNext3") }
        .doAfterNext { printName("doAfterNext3") }
        .doOnTerminate { printName("doOnTerminate3") }
        .doAfterTerminate { printName("doAfterTerminate3") }

        .doOnNext { printName("doOnNext4") }
        .doAfterNext { printName("doAfterNext4") }
        .doOnTerminate { printName("doOnTerminate4") }
        .doAfterTerminate { printName("doAfterTerminate4") }

        // 切换为住线程
        .observeOn(AndroidSchedulers.mainThread())
        .subscribe({ printName("onNext") })
```

输出结果：

```
subscribe on thread main
doOnNext1 on thread RxCachedThreadScheduler-1
doAfterNext1 on thread RxCachedThreadScheduler-1
doOnTerminate1 on thread RxCachedThreadScheduler-1
doOnNext2 on thread main
doAfterNext2 on thread main
doOnTerminate2 on thread main
doAfterTerminate1 on thread RxCachedThreadScheduler-1
doAfterTerminate3 on thread main
doOnNext3 on thread RxCachedThreadScheduler-2
doOnNext4 on thread RxCachedThreadScheduler-2
doAfterNext4 on thread RxCachedThreadScheduler-2
doAfterNext3 on thread RxCachedThreadScheduler-2
doOnTerminate3 on thread RxCachedThreadScheduler-2
doOnTerminate4 on thread RxCachedThreadScheduler-2
doAfterTerminate4 on thread RxCachedThreadScheduler-2
doAfterTerminate3 on thread RxCachedThreadScheduler-2
onNext on thread main
```

# Backpressure

## MissingBackpressureException

这个异常产生的原因就是上游发送的事件速度超过下游处理事件的速度，Observable中没有处理背压，当下游处理不了上游的事件的时候，就会抛出异常，对于数据的处理，应该使用 FLowable，创建的时候要指定 BackpressureStrategy

``` kotlin
Flowable.create(FlowableOnSubscribe<Int> {

}, BackpressureStrategy.BUFFER)
```

BackpressureStrategy类型有

``` java
/**
 * Represents the options for applying backpressure to a source sequence.
 */
public enum BackpressureStrategy {
    /**
     * OnNext events are written without any buffering or dropping.
     * Downstream has to deal with any overflow.
     * <p>Useful when one applies one of the custom-parameter onBackpressureXXX operators.
     */
    MISSING,
    /**
     * Signals a MissingBackpressureException in case the downstream can't keep up.
     */
    ERROR,
    /**
     * Buffers <em>all</em> onNext values until the downstream consumes it.
     */
    BUFFER,
    /**
     * Drops the most recent onNext value if the downstream can't keep up.
     */
    DROP,
    /**
     * Keeps only the latest onNext value, overwriting any previous value if the
     * downstream can't keep up.
     */
    LATEST
}
```

 - MISSING 上游不做任何处理，将事件全部抛给下游
 - ERROR 当下游处理不了的时候抛出*MissingBackpressureException*异常
 - BUFFER 上游缓存事件等待下游去处理
 - DROP 下游处理不了的时候，上游抛弃最新的事件
 - LATEST 下游处理不了的时候上游只发送最新的事件










