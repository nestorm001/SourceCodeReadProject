### Handler源码阅读

#### 前言
由于Android中主线程的存在，及对异步任务的需求，线程间的通信就成为了一个重要的问题，Handler便是线程间通信的重要手段，Handler、Looper和Message这三个好基友在一起就是一个完整的异步通信框架。  


#### 源码阅读

* 首先看Handler中的一个内部类

在各种情况中，匿名类持有Activity的context是内存泄漏的重要原因，因此，能只持有Application的context的时候就不要用Activity的，在任务完成后及时释放，必要情况下使用软引用和弱引用。  
这里的这个回调接口便是Handler提供的一个解决方案
```Java
// return True if no further handling is desired
public interface Callback {
    public boolean handleMessage(Message msg);
}
```
* dispatchMessage  
```Java
// 对message的分发方法
public void dispatchMessage(Message msg) {
		// 如果message自身有callback，就会让message的callback进行处理
        if (msg.callback != null) {
            handleCallback(msg);
        } else {
	        // 若定义了handler的callback，则由callback处理
	        // 否则，由handler本身进行处理
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            handleMessage(msg);
        }
}
```

* 构造方法

```Java
public Handler()
public Handler(Callback callback)
public Handler(Looper looper)
public Handler(Looper looper, Callback callback)
// @hide方法
public Handler(boolean async)
public Handler(Callback callback, boolean async)
public Handler(Looper looper, Callback callback, boolean async)
```
其中
```Java
// public Handler()
// public Handler(Callback callback)
// 所调用的构造方法
public Handler(Callback callback, boolean async) {
        // 根据开关情况检查内存泄漏
        ...
		// 而looper用的是当前线程的looper
		// 关于looper的具体情况及使用一会到Looper类里看
        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
}
// public Handler(Looper looper)
// public Handler(Looper looper, Callback callback)
// 所调用的构造方法 
// 由调用方手动指定Looper，之前我们看到AsyncTask  
// 中便是用的这种方法使用的Handler
public Handler(Looper looper, Callback callback, boolean async) {
        mLooper = looper;
        mQueue = looper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
}
```
这里这个boolean async 的意义：
```Java
If true, the handler calls {@link Message#setAsynchronous(boolean)} for
each {@link Message} that is sent to it or {@link Runnable} that is posted to it.
```
具体到下面用到时再看

* obtainMessage

内部调用了获取Message对象的工厂方法，具体实现需要到Message类中研究，暂时略过
```Java
public final Message obtainMessage()
public final Message obtainMessage(int what)
public final Message obtainMessage(int what, Object obj)
public final Message obtainMessage(int what, int arg1, int arg2)
public final Message obtainMessage(int what, int arg1, int arg2, Object obj)
```
其中what、arg1、arg2、obj分别对应Message中的变量

* 一些Runnable的post方法，将Runnable发送到消息队列，并指定执行时间，remove方法，从消息队列中移除

```Java
// post
// 本质是讲Runnable封装为Message后调用sendMessageAtTime
public final boolean post(Runnable r)
public final boolean postAtTime(Runnable r, long uptimeMillis)
public final boolean postAtTime(Runnable r, Object token, long uptimeMillis)
public final boolean postDelayed(Runnable r, long delayMillis)
public final boolean postAtFrontOfQueue(Runnable r)
// remove
// 在需要时调用removeCallbacks可以有效避免内存泄漏的问题
public final void removeCallbacks(Runnable r)
public final void removeCallbacks(Runnable r, Object token)
```

* 直接sendMessage的各种方法，上面的post方法基本都调用了对应的sendMessage方法

```Java
public final boolean sendMessage(Message msg)
public final boolean sendEmptyMessage(int what)
public final boolean sendEmptyMessageDelayed(int what, long delayMillis)
public final boolean sendEmptyMessageAtTime(int what, long uptimeMillis)
public final boolean sendMessageDelayed(Message msg, long delayMillis)
// 上面的方法在最后都调用了这个方法
public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
        MessageQueue queue = mQueue;
        //做些判断
        ...
        return enqueueMessage(queue, msg, uptimeMillis);
}
```

```Java
public final boolean sendMessageAtFrontOfQueue(Message msg) {
        MessageQueue queue = mQueue;
        //做些判断 跟上面那段一毛一样
        ...
        return enqueueMessage(queue, msg, 0);
}
```
DRY、DRY、DRY
不如直接写成这个：
```Java
public final boolean sendMessageAtFrontOfQueue(Message msg) {
        return sendMessageAtTime(msg, 0);
}
```
不吐槽了
看enqueueMessage方法
```Java
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
		//将message的target设为this，即由当前handler处理message
        msg.target = this;
        // 终于看到这个boolean的用处了，至于在message里面干了什么，一会再看
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        // 最后，终于把message加进了队列
        return queue.enqueueMessage(msg, uptimeMillis);
}
```

* 剩下的还有一些个getter，对message存在判断的方法及移除message的方法就不看了

#### 总结
看完了好基友之一（要是算上MessageQueue得有四个好基友。。），对Handler进行异步处理的机制有了一定的认识，Message由Handler代理加入MessageQueue，并且在接收到Message的时候，按需要由Handler进行各种处理，至于MessageQueue中Message的分发和线程的控制，大概需要再看Looper才能彻底理解了吧