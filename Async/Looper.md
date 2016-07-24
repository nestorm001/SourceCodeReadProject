### Looper源码阅读

#### 前言
之前看完了Handler，对于Handler、Message、MessageQueue的作用有了大致的了解，而对Looper没有一个很好的认识，现在就开始看Looper


#### 源码阅读

* 首先看各种变量的定义

```Java
// ThreadLocal又是一个深坑需要研究，大致就是
// a thread-local storage, that is, a variable for 
// which each thread has its own value
static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
// 主线程所使用的Looper，全局只有这一个
private static Looper sMainLooper;
// Looper所维护的MessageQueue
final MessageQueue mQueue;
// Looper当前所在线程
final Thread mThread;
```
* 私有构造方法
```Java
private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mThread = Thread.currentThread();
}
```
* prepare，初始化当前线程的Looper，每个线程只能初始化一次
```Java
// 普通线程的prepare，可以退出
public static void prepare()
// 初始化主线程Looper，不可退出，且由系统调用，不应该由应用自身调用
public static void prepareMainLooper()
// 上面两个方法都调用了
private static void prepare(boolean quitAllowed) {
        // 判断是否已存在
        ...
        sThreadLocal.set(new Looper(quitAllowed));
}
```
* looper的关键部分，忽略了部分代码，message的分发处理方法loop()
```Java
public static void loop() {
        // myLooper()方法从sThreadLocal中获取当前线程的Looper
        // 即在prepare中new出来的那个Looper
        final Looper me = myLooper();
        final MessageQueue queue = me.mQueue;

        // Make sure the identity of this thread is that of the local process,
        // and keep track of what that identity token actually is.
        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();

        for (;;) {
	        // 获取消息
            Message msg = queue.next(); // might block
            if (msg == null) {
                return;
            }
			// 分发消息
            msg.target.dispatchMessage(msg);

            // Make sure that during the course of dispatching the
            // identity of the thread wasn't corrupted.
            final long newIdent = Binder.clearCallingIdentity();
            if (ident != newIdent) {
				...
				// 打log
            }

            msg.recycleUnchecked();
        }
}
```
可以看到，只要队列中有Message，就会不停发送给target进行处理

* quit

```Java
public void quit()
public void quitSafely()
```

#### 总结
Looper作为好基友之一，在非主线程中使用时，需要手动prepare后进行loop，主要用于分发MessageQueue中的消息，对于线程的控制主要由Thread.currentThread()实现，一个Thread对应一个Looper，这么看来主线程并没有特别特殊嘛