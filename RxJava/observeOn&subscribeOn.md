### RxJava线程切换分析   

#### 前言  
RxJava近年来十分受欢迎，它不仅仅是一个框架，更是一种思路。RxJava的基本思想便是观察者模式，使用RxJava写代码，特别是写原来需要各种回调才能做到的异步任务代码，变得更为简单明了。使用RxJava也有一段时间了，对于各种操作符的使用也算是比较熟悉了，虽然还没有深入的研究过，但是对于RxJava的整体思路也是有了一定的理解，然而，RxJava的线程切换是如何做到的，哪些操作符使用的是observeOn，哪些用的是subcribeOn，现在就针对这些问题，去看看源码，也算是RxJava源码学习的第一步吧。

对于RxJava的入门学习，建议先看这两篇  
[NotRxJava懒人专用指南](http://www.devtf.cn/?p=323)  
[给 Android 开发者的 RxJava 详解](http://gank.io/post/560e15be2dca930e00da1083)  

#### 源码分析
* subscribeOn
```Java
// subscribeOn决定订阅的内容在哪个线程执行，即对上负责
//Asynchronously subscribes Observers to this Observable on the specified scheduler.
//@param scheduler the Scheduler to perform subscription actions on
//@return the source Observable modified so that its subscriptions happen on the specified Scheduler
public final Observable<T> subscribeOn(Scheduler scheduler) {
   // 判断ScalarSynchronousObservable相关，略过
   ...
   
   return create(new OperatorSubscribeOn<T>(this, scheduler));
}
```

subscribeOn最后的return调用了`create(OnSubscribe<T> f)`，可见` OperatorSubscribeOn<T>(this, scheduler)`是一个OnSubscribe。

```java
// OperatorSubscribeOn<T> 构造方法
public OperatorSubscribeOn(Observable<T> source, Scheduler scheduler) {
    this.scheduler = scheduler;
    this.source = source;
} 
// OnSubscribe的call方法
@Override
public void call(final Subscriber<? super T> subscriber) {
	// 创建工作者inner
    final Worker inner = scheduler.createWorker();
    // 有点疑惑
    subscriber.add(inner);
    
    // inner安排一个任务
    inner.schedule(new Action0() {
        @Override
        public void call() {
            final Thread t = Thread.currentThread();
            
            // inner中new 一个Subscriber代理原Subscriber的三个Action
            // onNext onCompleted onError及setProducer
            Subscriber<T> s = new Subscriber<T>(subscriber) {
                @Override
                public void onNext(T t) {
                    subscriber.onNext(t);
                }
                
                @Override
                public void onError(Throwable e) {
                    try {
                        subscriber.onError(e);
                    } finally {
                        inner.unsubscribe();
                    }
                }
                
                @Override
                public void onCompleted() {
                    try {
                        subscriber.onCompleted();
                    } finally {
                        inner.unsubscribe();
                    }
                }
                
                @Override
                public void setProducer(final Producer p) {
                    subscriber.setProducer(new Producer() {
                        @Override
                        public void request(final long n) {
                            if (t == Thread.currentThread()) {
                                p.request(n);
                            } else {
                                inner.schedule(new Action0() {
                                    @Override
                                    public void call() {
                                        p.request(n);
                                    }
                                });
                            }
                        }
                    });
                }
            };
            // 最后由新的subscriber订阅原来的observable
            // 为了保持链式调用，这个调用看上去是反的
            source.unsafeSubscribe(s);
        }
    });
}
```
从上面可以看出subscribeOn主要改变了subcriber订阅时所在的线程或者说是OnSubscribe执行call方法时所在的线程  
在这里，subscribeOn的本质是新建一个Observable，并在指定线程执行原Observable中OnSubscribe的call方法  
多个subscribeOn无效是因为无论如何改变线程，最后都会由第一个subscribeOn改变线程进行数据的生产过程   
而其它的subscribeOn改变的只是它之前的subscribeOn中的OnSubscribe的call方法执行的线程，对于非生产行为的操作符(各种do系列)，并不能起到任何效果  

* observeOn
```Java
// observeOn决定订阅的的结果在哪个线程执行，即对下负责
//Modifies an Observable to perform its emissions and notifications on a specified scheduler asynchronously with a bounded buffer.
//@param scheduler the Scheduler to notify observers on on
//@return the source Observable modified so that its observers are notified on the specified scheduler

public final Observable<T> observeOn(Scheduler scheduler) {
        // 判断ScalarSynchronousObservable相关，略过
        ...
        
        return lift(new OperatorObserveOn<T>(scheduler, false));
}

//与上面那个类似，上面那个delayError默认false了而已
public final Observable<T> observeOn(Scheduler scheduler, boolean delayError) {
        // 判断ScalarSynchronousObservable相关，略过
        ...
        
        return lift(new OperatorObserveOn<T>(scheduler, delayError));
}
```

```Java
// lift方法 本质就是返回一个新的Observable
// 这个Observable中的OnSubscribe通知原Observable执行call系列方法，包括call之前的onStart
// 而原Observable的结果通过新Observable的operator发送给后面的Subscriber
public final <R> Observable<R> lift(final Operator<? extends R, ? super T> operator) {
        return new Observable<R>(new OnSubscribe<R>() {
            @Override
            public void call(Subscriber<? super R> o) {
                try {
                    Subscriber<? super T> st = hook.onLift(operator).call(o);
                    try {
                        st.onStart();
                        // onSubscribe为原始Observable中的OnSubscribe
                        onSubscribe.call(st);
                    } catch (Throwable e) {
	                    ...   
                } catch (Throwable e) {
                    ...
                }
            }
        });
}
```

```Java
//OperatorObserveOn

@Override
public Subscriber<? super T> call(Subscriber<? super T> child) {
	    //一些判断，若满足直接返回
	    ...
    
	    // 把传入的Subscriber又包装了一遍
        ObserveOnSubscriber<T> parent = new ObserveOnSubscriber<T>(scheduler, child, delayError);
        parent.init();
        return parent;
    }
}
```

```Java
// 三个重要的变量，分别对应了onCompleted、onError、onNext的调用的判断
volatile boolean finished;
Throwable error;
final Queue<Object> queue;

// ObserveOnSubscriber中的方法
@Override
public void onNext(final T t) {
    if (isUnsubscribed() || finished) {
        return;
    }
    // onNext结果入队
    if (!queue.offer(on.next(t))) {
        onError(new MissingBackpressureException());
        return;
    }
    schedule();
}
@Override
public void onCompleted() {
    if (isUnsubscribed() || finished) {
        return;
    }
    finished = true;
    schedule();
}
@Override
public void onError(final Throwable e) {
    if (isUnsubscribed() || finished) {
        RxJavaPlugins.getInstance().getErrorHandler().handleError(e);
        return;
    }
    error = e;
    finished = true;
    schedule();
}

// 上面几个方法，包括init都调用了schedule
protected void schedule() {
    if (counter.getAndIncrement() == 0) {
	    // 
        recursiveScheduler.schedule(this);
    }
}
// schedule方法
public abstract Subscription schedule(Action0 action);

// 所以，最后的schedule通过调用Action0的call方法控制了各个action所执行的线程
// 省略部分代码
@Override
public void call() {
	...
    // 关键部分 放置next的queue
    final Queue<Object> q = this.queue;
    ...
        
    for (;;) {
	    // 根据完成与否，出错与否，调用onError和onCompleted
        if (checkTerminated(finished, q.isEmpty(), localChild, q)) {
            return;
        }
        
        ...
        
        while (requestAmount != 0L) {
            boolean done = finished;
            // 获取next的值
            Object v = q.poll();
            boolean empty = v == null;
            // 再次check，根据完成与否，出错与否，调用onError和onCompleted
            if (checkTerminated(done, empty, localChild, q)) {
                return;
            }
            if (empty) {
                break;
            }
            // 正式调用onNext
            localChild.onNext(localOn.getValue(v));
            ...
        }
        ...
    }
    ...
}
```
observeOn改变的是消费行为所在的线程、因此调用observeOn之后  
无论怎么使用subscribeOn(虽说只有第一次subscribeOn有用)，对消费行为不会产生任何影响  

#### 总结  

~~大致对这两个方法研究了一下，虽说整体梳理了一遍，知道了这两个方法到底改变了哪个调用的线程：~~  
~~subscribeOn改变的是OnSubscribe的call，Subscriber的onStart一般会在call之前调用，因此，会在call执行的线程被调用~~  
~~observeOn改变的是Subscriber的onNext、onComplete、onNext等~~   
~~但内部的一些细节还是不太能理解，对于这两个方法对各个操作符的影响也不太能确定(大概还得看看操作符的实现)，经过试验和分析大致得到的一些规律：~~  
~~subscribeOn对上负责，改变数据产生的线程，即只对生产负责~~    
~~observeOn对下负责(屏幕意义上的上下)，改变数据消费的线程，即只对消费负责~~    
~~如果先调用observeOn，则subscribeOn只对observeOn之上的操作符有效~~    
~~普通操作，各种变换之类的操作，subscribeOn只有第一个有用~~    
~~一旦调用了observeOn，事件就从生产变成了消费，以此为分界线，想要切换线程只能使用observeOn了（大误）~~    
~~无论如何observeOn不会影响到OnSubscribe调用的线程~~  

#### 总结2.0  
对于这两个方法梳理了好几遍，现在对其原理和行为明确多了，对RxJava也有了更深入的了解  

subscribeOn改变生产行为，对于普通操作符(非do系列)，只有第一次有效  
后面的subscribeOn改变的是前面的subscribeOn的生产行为，因此对原始Observable的OnSubscribe的call方法并不能产生影响  

observeOn改变了消费行为，因此无论如何用subscribeOn改变生产行为，对于消费行为所执行的线程并没有影响  

对于do系列分别调用subscribeOn有用，待研究   


总结参考了  
[迷之RxJava （三）update 2 —— subscribeOn 和 observeOn 的区别](https://segmentfault.com/a/1190000004856071)