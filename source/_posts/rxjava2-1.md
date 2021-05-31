---
title: Rxjava2原理解析
date: 2020-02-10 20:57:25
catalog:    true
header-img: "post-bg-unix-linux.jpg"
tags:
    - Java
    - Android
---


## 如何使用Rxjava2

这里我们先看一下大部分情况下我们是如何使用Rxjava2的

1.创建被观察者Observable
```js
//1、创建被观察者Observable
Observable<String> MyObservable
= Observable.create(new ObservableOnSubscribe<String>() {
    @Override
    public void subscribe(ObservableEmitter<String> e) throws Exception {
        e.onNext(" RxJava：e.onNext== 第一次");
        e.onComplete();
        Log.d(TAG, "subscribe() 线程id==>" + Thread.currentThread().getId() + " 线程名：" + Thread.currentThread().getName());
    }
}).subscribeOn(Schedulers.io())
  .observeOn(AndroidSchedulers.mainThread());
```

2.创建观察者Observer
```js
//2、创建观察者Observer
Observer<String> MyObserver
= new Observer<String>() {
    @Override
    public void onSubscribe(Disposable d) {
        Log.e(TAG, "onSubscribe == 订阅");
    }

    @Override
    public void onNext(String s) {
        Log.e(TAG, "onNext ==》 " + s);
        Log.d(TAG, "onNext() 线程id==>" + Thread.currentThread().getId() + " 线程名：" + Thread.currentThread().getName());
    }

    @Override
    public void onError(Throwable e) {
        Log.e(TAG, "onError == " + e.getMessage());
    }

    @Override
    public void onComplete() {
        Log.e(TAG, "onComplete == ");
    }
};
```

3.订阅

```js
//3、订阅(观察者观察被观察者)
MyObservable.subscribe(MyObserver);
```

## 开始正文

上述第一步创建被观察者的过程中总共创建了3个Observable对象，分别是ObservableCreate、 ObservableSubscribeOn、ObservableObserveOn。

* Observable.create()返回ObservableCreate
* subscribeOn()返回ObservableSubscribeOn
* observeOn()返回ObservableObserveOn

再看一下这3个Observable对象的创建
**1.ObservableCreate**
```js
public static <T> Observable<T> create(ObservableOnSubscribe<T> source) {
    ObjectHelper.requireNonNull(source, "source is null");
    return RxJavaPlugins.onAssembly(new ObservableCreate<T>(source));
}
```

**2.ObservableSubscribeOn**
```js
public final Observable<T> subscribeOn(Scheduler scheduler) {
    ObjectHelper.requireNonNull(scheduler, "scheduler is null");
    return RxJavaPlugins.onAssembly(new ObservableSubscribeOn<T>(this, scheduler));
}
```
new ObservableSubscribeOn<T>(this, scheduler)，其中this对象是上一级创建的ObservableCreate对象，所以ObservableSubscribeOn对象中嵌套着上一级的ObservableCreate对象

**3.ObservableObserveOn**
```js
public final Observable<T> observeOn(Scheduler scheduler, boolean delayError, int bufferSize) {
    ObjectHelper.requireNonNull(scheduler, "scheduler is null");
    ObjectHelper.verifyPositive(bufferSize, "bufferSize");
    return RxJavaPlugins.onAssembly(new ObservableObserveOn<T>(this, scheduler, delayError, bufferSize));
}
```
new ObservableObserveOn<T>(this, scheduler, delayError, bufferSize)，其中this对象是上一级创建的ObservableSubscribeOn，所以ObservableObserveOn对象中嵌套着上一级的ObservableSubscribeOn对象，**形成如下图的层级结构：**
![java-javascript](observables.png)
<small class="img-hint">observable的层级结构</small>

## 如何订阅

> 下面梳理流程有以下几个关键点： MyObservable.subscribe(MyObserver)、subscribeOn(Schedulers.io())、observeOn(AndroidSchedulers.mainThread())。**订阅、线程切换、数据分发**。

MyObservable.subscribe(MyObserver)
```js
public final void subscribe(Observer<? super T> observer) {
    ObjectHelper.requireNonNull(observer, "observer is null");
    try {
        observer = RxJavaPlugins.onSubscribe(this, observer);

        ObjectHelper.requireNonNull(observer, "The RxJavaPlugins.onSubscribe hook returned a null Observer. Please change the handler provided to RxJavaPlugins.setOnObservableSubscribe for invalid null returns. Further reading: https://github.com/ReactiveX/RxJava/wiki/Plugins");

        subscribeActual(observer);
    } catch (NullPointerException e) { // NOPMD
        throw e;
    } catch (Throwable e) {
        Exceptions.throwIfFatal(e);
        RxJavaPlugins.onError(e);
        NullPointerException npe = new NullPointerException("Actually not, but can't throw other exceptions due to RS");
        npe.initCause(e);
        throw npe;
    }
}
```
这是subscribe的代码，主要看这部分代码subscribeActual(observer)，subscribeActual(observer)是Observable的一个抽象方法。根据前面的形成的Observable层级结构可知，MyObservable.subscribe(MyObserver)是先调用`ObservableObserveOn`.subscribeActuall(observer)。接下来看看这部分代码：
```js
@Override
protected void subscribeActual(Observer<? super T> observer) {
    if (scheduler instanceof TrampolineScheduler) {
        source.subscribe(observer);
    } else {
        Scheduler.Worker w = scheduler.createWorker();
        source.subscribe(new ObserveOnObserver<T>(observer, w, delayError, bufferSize));
    }
}
```
其中source是ObservableSubscribeOn对象，scheduler是AndroidSchedulers.mainThread()，scheduler.createWorker()返回一个HandlerWorker对象，HandlerWorker对象中包含着handler。可以猜想到这个对象的作用是实现异步线程到主线程的切换工作，这步验证待会再看，先看source.subscribe(new ObserveOnObserver<T>(observer, w, delayError, bufferSize));此处传入的Observer对象是ObserveOnObserver。所以接下来看一下ObservableSubscribeOn中subscribeActual(observer)的实现。
```js
public void subscribeActual(final Observer<? super T> observer) {
    final SubscribeOnObserver<T> parent = new SubscribeOnObserver<T>(observer);

    observer.onSubscribe(parent);

    parent.setDisposable(scheduler.scheduleDirect(new SubscribeTask(parent)));
}
```
说明一下这段代码中的几个关键的对象，observer是第三级ObservableObserveOn中的内部类ObserveOnObserver，parent是内部类Observer的实现SubscribeOnObserver，scheduler是IoScheduler，SubscribeTask则是Runnable的实现，代码如下:
```js
final class SubscribeTask implements Runnable {
    private final SubscribeOnObserver<T> parent;

    SubscribeTask(SubscribeOnObserver<T> parent) {
        this.parent = parent;
    }

    @Override
    public void run() {
        source.subscribe(parent);
    }
}
```
接下来看一下scheduler.scheduleDirect(new SubscribeTask(parent))的代码
```js
public Disposable scheduleDirect(@NonNull Runnable run, long delay, @NonNull TimeUnit unit) {
    final Worker w = createWorker();

    final Runnable decoratedRun = RxJavaPlugins.onSchedule(run);

    DisposeTask task = new DisposeTask(decoratedRun, w);

    w.schedule(task, delay, unit);

    return task;
}
```
createWorker返回EventLoopWorker对象
```js
public Worker createWorker() {
    return new EventLoopWorker(pool.get());
}
```
再看一下EventLoopWorker
```js
static final class EventLoopWorker extends Scheduler.Worker {
    private final CompositeDisposable tasks;
    private final CachedWorkerPool pool;
    private final ThreadWorker threadWorker;

    final AtomicBoolean once = new AtomicBoolean();

    EventLoopWorker(CachedWorkerPool pool) {
        this.pool = pool;
        this.tasks = new CompositeDisposable();
        this.threadWorker = pool.get();
    }
```
其中CachedWorkerPool内部维护着一个ScheduledExecutorService线程池
```js
static final class CachedWorkerPool implements Runnable {
    private final long keepAliveTime;
    private final ConcurrentLinkedQueue<ThreadWorker> expiringWorkerQueue;
    final CompositeDisposable allWorkers;
    private final ScheduledExecutorService evictorService;
    private final Future<?> evictorTask;
    private final ThreadFactory threadFactory;
```
所以scheduler.scheduleDirect(new SubscribeTask(parent))代码实际上是使用线程池中的线程执行到SubscribeTask了run()方法中，看一下run()中的代码：
```js
final class SubscribeTask implements Runnable {
    private final SubscribeOnObserver<T> parent;

    SubscribeTask(SubscribeOnObserver<T> parent) {
        this.parent = parent;
    }

    @Override
    public void run() {
        source.subscribe(parent);
    }
}
```
此时的source.subscribe(parent)是执行在了异步线程中，上述的步骤就完成了一次线程切换，其中source是ObservableCreate，parent是ObservableSubscribeOn的内部类SubscribeOnObserver，该类是Observer的实现类。所以source.subscribe(parent);调用到ObservableCreate的subscribeActuall(observer)中，看一下该部分代码:
```js
protected void subscribeActual(Observer<? super T> observer) {
    CreateEmitter<T> parent = new CreateEmitter<T>(observer);
    observer.onSubscribe(parent);

    try {
        source.subscribe(parent);
    } catch (Throwable ex) {
        Exceptions.throwIfFatal(ex);
        parent.onError(ex);
    }
}
```
因为上面的source.subscribe(parent)已经完成了线程切换并在异步线程中执行，所以ObservableCreate.subscribeActuall(observer)也是在异步线程中执行。所以下面的代码执行都是在异步线程中，其中observer是第二级ObservableSubscribeOn的内部类SubscribeOnObserver，source是最外层的ObservableOnSubscribe，所以source.subscribe(parent)会调用到ObservableOnSubscribe.subscribe()代码如下：
```js
Observable<String> MyObservable = Observable.create(new ObservableOnSubscribe<String>() {
    @Override
    public void subscribe(ObservableEmitter<String> e) throws Exception {
        e.onNext(" RxJava：e.onNext== 第一次");
        e.onComplete();
        Log.d(TAG, "subscribe() 线程id==>" + Thread.currentThread().getId() + " 线程名：" + Thread.currentThread().getName());
    }
})
```
**执行到这一步就完成了整个订阅流程也知道了订阅流程的线程切换是发生在ObservableSubscribeOn的subscribeActual(observer)中的。**

## 数据如何通过onNext分发

ObservableOnSubscribe.subscribe()做了两件事，执行了e.onNext()，e.onComplete(),其中e是ObservableCreate中的内部类CreateEmitter，这里只说明onNext，onComplete同理。看一下CreateEmitter.onNext()中的代码：
```js
public void onNext(T t) {
    if (t == null) {
        onError(new NullPointerException("onNext called with null. Null values are generally not allowed in 2.x operators and sources."));
        return;
    }
    if (!isDisposed()) {
        observer.onNext(t);
    }
}
```
此时onNext还是在异步线程中执行，这段代码主要执行了observer.onNext(t)，所以只是做了onNext()的转发，其中observer是第二级ObservableSubscribeOn的内部类SubscribeOnObserver，就调回了SubscribeOnObserver.onNext方法，看一下这部分的代码：
```js
public void onNext(T t) {
    downstream.onNext(t);
}
```
主要是对onNext进行转发，其中downstream是ObservableObserveOn的内部类ObserveOnObserver，看一下ObserveOnObserver.onNext()方法部分的代码：
```js
public void onNext(T t) {
    Log.d("scj", "ObservableObserveOn onNext 线程id==>" + Thread.currentThread().getId() + " 线程名：" + Thread.currentThread().getName());
    if (done) {
        return;
    }

    if (sourceMode != QueueDisposable.ASYNC) {
        queue.offer(t);
    }
    schedule();
}
```
其中主要是schedule()方法
```js
void schedule() {
    if (getAndIncrement() == 0) {
        worker.schedule(this);
    }
}
```
其中worker就是scheduler.createWorker()返回一个HandlerWorker对象，看一下schedule(this)的代码：
```js
public Disposable schedule(Runnable run, long delay, TimeUnit unit) {
    if (run == null) throw new NullPointerException("run == null");
    if (unit == null) throw new NullPointerException("unit == null");

    if (disposed) {
        return Disposables.disposed();
    }

    run = RxJavaPlugins.onSchedule(run);

    ScheduledRunnable scheduled = new ScheduledRunnable(handler, run);

    Message message = Message.obtain(handler, scheduled);
    message.obj = this; // Used as token for batch disposal of this worker's runnables.

    if (async) {
        message.setAsynchronous(true);
    }

    handler.sendMessageDelayed(message, unit.toMillis(delay));

    // Re-check disposed state for removing in case we were racing a call to dispose().
    if (disposed) {
        handler.removeCallbacks(scheduled);
        return Disposables.disposed();
    }

    return scheduled;
}
```
这部分的代码就是将线程切换到主线程上，并调用到ObserveOnObserver.run()中，看一下run()的代码：
```js
public void run() {
    if (outputFused) {
        drainFused();
    } else {
        drainNormal();
    }
}
```
此时run()已经处于主线程
```js
void drainNormal() {
    int missed = 1;

    final SimpleQueue<T> q = queue;
    final Observer<? super T> a = downstream;

    for (;;) {
        if (checkTerminated(done, q.isEmpty(), a)) {
            return;
        }

        for (;;) {
            boolean d = done;
            T v;

            try {
                v = q.poll();
            } catch (Throwable ex) {
                Exceptions.throwIfFatal(ex);
                disposed = true;
                upstream.dispose();
                q.clear();
                a.onError(ex);
                worker.dispose();
                return;
            }
            boolean empty = v == null;

            if (checkTerminated(d, empty, a)) {
                return;
            }

            if (empty) {
                break;
            }

            a.onNext(v);
        }

        missed = addAndGet(-missed);
        if (missed == 0) {
            break;
        }
    }
}
```
其中downstream就是最外层的MyObserver对象，上面代码的功能也是对onNext方法进行了转发，所以此时的onNext是在主线程上被回调转发。
```js
//2、创建观察者Observer
Observer<String> MyObserver = new Observer<String>() {
    @Override
    public void onSubscribe(Disposable d) {
        Log.e(TAG, "onSubscribe == 订阅");
    }

    @Override
    public void onNext(String s) {
        Log.e(TAG, "onNext ==》 " + s);
        Log.d(TAG, "onNext() 线程id==>" + Thread.currentThread().getId() + " 线程名：" + Thread.currentThread().getName());
    }

    @Override
    public void onError(Throwable e) {
        Log.e(TAG, "onError == " + e.getMessage());
    }

    @Override
    public void onComplete() {
        Log.e(TAG, "onComplete == ");
    }
};
```
**最后onNext就是调用到上述的代码中，并且已经完成了主线程的切换。**

> 好了，上述完成了**订阅、线程切换、数据分发**所有流程的解读。下面我再总结一下这整个步骤

## 总结

![java-javascript](observable_2.png)
<small class="img-hint">observable的层级结构</small>

1.创建Observable的过程形成如上图的层级嵌套关系
2.订阅过程由外面的层级向内部层级调用，而线程切换发生在ObservableSubscribeOn这一层的scheduler.scheduleDirect(new SubscribeTask(parent))
3.数据发送由内部层级向外部层级调用，主线程切换在ObservableObserveOn的内部类ObserveOnObserver回调到onNext后调用schedule()时发生的

> 问:subscribeOn()为什么多次调用时有效的只有第一次？