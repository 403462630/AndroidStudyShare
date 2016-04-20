### RxJava
1. RxJava 的观察者模式简单介绍(#1)
*  map和flatmap的区别
*  lift和compose的区别
*  线程控制Scheduler
*  什么是subject
*  Rxjava extension library简单介绍

### RxJava 的观察者模式简单介绍{#1}
RxJava 有四个基本概念：Observable (可观察者，即被观察者)、 Observer (观察者)、 subscribe (订阅)、事件。Observable 和 Observer 通过 subscribe() 方法实现订阅关系，从而 Observable 可以在需要的时候发出事件来通知 Observer。
与传统观察者模式不同， RxJava 的事件回调方法除了普通事件 onNext() （相当于 onClick() / onEvent()）之外，还定义了两个特殊的事件：onCompleted() 和 onError()。
* onCompleted(): 事件队列完结。RxJava 不仅把每个事件单独处理，还会把它们看做一个队列。RxJava 规定，当不会再有新的 onNext() 发出时，需要触发 onCompleted() 方法作为标志。
* onError(): 事件队列异常。在事件处理过程中出异常时，onError() 会被触发，同时队列自动终止，不允许再有事件发出。
* 在一个正确运行的事件序列中, onCompleted() 和 onError() 有且只有一个，并且是事件序列中的最后一个。需要注意的是，onCompleted() 和 onError() 二者也是互斥的，即在队列中调用了其中一个，就不应该再调用另一个。


### 什么是subject
先看看Subject的定义
```
public abstract class Subject<T, R> extends Observable<R> implements Observer<T> {
    ......
}
```
可见Subject即是Observable又是Observer，因为它是一个Observer，所以它可以订阅一个或多个Observable，又因为它是一个Observable，所以它可以发射数据给Observer

Subject 与 Observable有什么区别呢？
主要的区别是它们发射数据的方式不同，原始的Observable是“冷发射”，就是说只有当Observer订阅它的时候才开始发射数据，
而Subject可以将“冷发射”转变成“热发射”,也就是说Subject可以在Observer订阅之前开始发送数据，当然Observer订阅之后也可以发送数据

```
//可见当执行subscribe()方法时，会调用call方法，在call方法里会发射hello1和hello2数据给Observer
Observable.create(new Observable.OnSubscribe<String>() {
    @Override
    public void call(Subscriber<? super String> subscriber) {
        subscriber.onNext("Hello1");
        subscriber.onNext("Hello2");
        subscriber.onCompleted();
    }
}).subscribe();

//可见subject可以在subscribe()之前或之后发射数据，完全在外面控制的
PublishSubject subject = PublishSubject.create();
subject.onNext("Hello1");
subject.onNext("Hello2");
subject.subscribe();
subject.onNext("Hello3");
subject.onCompleted();
```

Subject的种类
1. PublishSubject
PublishSubject只会把在订阅发生的时间点之后来自原始Observable的数据发射给观察者，需要注意的是：Subject被创建后到有观察者订阅它之前这个时间段内，可能会有一个或多个数据可能会丢失，例如上面的hello1和hello2会丢失，不会发射给观察者，hello3才会发射给观察者
![](images/S.PublishSubject.png)

* BehaviorSubject
当观察者订阅BehaviorSubject时，它开始发射原始Observable最近发射的数据（如果此时还没有收到任何数据，它会发射一个默认值），然后继续发射其它任何来自原始Observable的数据
```
BehaviorSubject subject = BehaviorSubject.create("default");
subject.subscribe(createObserver());//这是Observer1
subject.onNext("hello1");
subject.subscribe(createObserver());//这是Observer2
subject.onNext("hello2");
subject.onCompleted();
```
subject会将default、hello1、hello2发射给Observer1，会将hello1、hello2发射给Observer2
![](images/S.BehaviorSubject.png)

* AsyncSubject
只在原始Observable完成后，发射来自原始Observable的最后一个值，如果原始Observable没有发射任何值，AsyncObject也不发射任何值）它会把这最后一个值发射给任何后续的观察者。
```
AsyncSubject asyncSubject = AsyncSubject.create();
asyncSubject.subscribe(createObserver());//这是Observer1
asyncSubject.onNext("hello1");
asyncSubject.onNext("hello2");
asyncSubject.subscribe(createObserver());//这是Observer2
asyncSubject.onNext("hello3");
asyncSubject.onCompleted();
```
subject只会将hello3发送给Observer1和Observer2
![](images/S.AsyncSubject.png)

* ReplaySubject
ReplaySubject会发射所有来自原始Observable的数据给观察者，无论它们是何时订阅的。也有其它版本的ReplaySubject，在重放缓存增长到一定大小的时候或过了一段时间后会丢弃旧的数据（原始Observable发射的）。
```
ReplaySubject subject = ReplaySubject.create();
subject.subscribe(createObserver());//这是Observer1
subject.onNext("hello1");
subject.onNext("hello2");
subject.onNext("hello3");
subject.subscribe(createObserver());//这是Observer2
subject.onCompleted();
```
subject只会将hello1、hello2、hello3发送给Observer1和Observer2
![](images/S.ReplaySubject.png)




