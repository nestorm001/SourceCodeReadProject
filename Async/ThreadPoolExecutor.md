### ThreadPoolExecutor源码阅读  

#### 前言  
ThreadPoolExecutor为接口Executor的实现类，用于管理使用线程，复用线程，节约资源，提高效率，其它的Executor的实现类有ThreadPoolExecutor的子类ScheduledThreadPoolExecutor，同样继承于AbstractExecutorService的子类ForkJoinPool及工厂类Executors的几个内部类。
#### 源码阅读  
* 运行状态   

```Java
The runState provides the main lifecycle control, taking on values:
RUNNING:  	Accept new tasks and process queued tasks
SHUTDOWN: 	Don't accept new tasks, but process queued tasks
STOP:     	Don't accept new tasks, don't process queued tasks,
			and interrupt in-progress tasks
TIDYING:  	All tasks have terminated, workerCount is zero,
			the thread transitioning to state TIDYING
			will run the terminated() hook method
TERMINATED: terminated() has completed
```

* 状态间的变化，单调不可逆  

```Java
RUNNING -> SHUTDOWN
   On invocation of shutdown(), perhaps implicitly in finalize()
(RUNNING or SHUTDOWN) -> STOP
   On invocation of shutdownNow()
SHUTDOWN -> TIDYING
   When both queue and pool are empty
STOP -> TIDYING
   When pool is empty
TIDYING -> TERMINATED
   When the terminated() hook method has completed
Threads waiting in awaitTermination() will return when the
state reaches TERMINATED.
```

* 一些变量的定义

```Java
// 工作队列
// concurrent包中可选实现大致有以下几种，具体区别和使用场景待研究
// SynchronousQueue  特殊的一个队列，只有存在等待取出的线程时才能加入队列，可以说容量为0，是无界队列
// PriorityBlockingQueue
// LinkedTransferQueue
// ArrayBlockingQueue  有界队列 
// LinkedBlockingQueue 无界队列 
// LinkedBlockingDeque
// DelayQueue
private final BlockingQueue<Runnable> workQueue;
// 用于存放工作线程，只有在持有mainLock的时候可达
private final HashSet<Worker> workers = new HashSet<Worker>();
// 两个直观的变量
private int largestPoolSize;
private long completedTaskCount;
// 通过addWorker方法来增加线程的工厂类
private volatile ThreadFactory threadFactory;
// 当等待队列满或shutdown被执行时，所使用的拒绝策略
private volatile RejectedExecutionHandler handler;
// 主要有以下几种
// 放弃最旧的任务
ThreadPoolExecutor.DiscardOldestPolicy();
// 新任务抢占线程先进行
ThreadPoolExecutor.CallerRunsPolicy();
// 放弃新任务
ThreadPoolExecutor.DiscardPolicy();
// 报错，默认策略
ThreadPoolExecutor.AbortPolicy();
// 直观的变量
// corePoolSize：核心池的大小，这个参数跟后面讲述的线程池的实现原理有非常大的关系。  
// 在创建了线程池后，默认情况下，线程池中并没有 任何线程，而是等待有任务到来才创建  
// 线程去执行任务，除非调用了prestartAllCoreThreads()或者 prestartCoreThread()方法，  
// 从这2个方法的名字就可以看出，是预创建线程的意思，即在没有任务到来之前就创建   
// corePoolSize个线程或者一个线程。默认情况下，在创建了线程池后，线程池中的线程数为0，  
// 当有任务来之后，就会创建一个线程去执行任务，当线 程池中的线程数目达到corePoolSize后，  
// 就会把到达的任务放到缓存队列当中；  
// 
// maximumPoolSize：线程池最大线程数，这个参数也是一个非常重要的参数，  
// 它表示在线程池中最多能创建多少个线程；  
// 
// keepAliveTime：表示线程没有任务执行时最多保持多久时间会终止。默认情况下，  
// 只有当线程池中的线程数大于corePoolSize 时，keepAliveTime才会起作用，  
// 直到线程池中的线程数不大于corePoolSize。即当线程池中的线程数大于corePoolSize 时，  
// 如果一个线程空闲的时间达到keepAliveTime，则会终止，直到线程池中的线程数不超过corePoolSize。  
// 但是如果调用了 allowCoreThreadTimeOut(boolean)方法，在线程池中的线程数不大于corePoolSize 时，  
keepAliveTime参数也会起作用，直到线程池中的线程数为0；
private volatile long keepAliveTime;
private volatile boolean allowCoreThreadTimeOut;
// the minimum number of workers to keep alive
private volatile int corePoolSize;
// bounded by CAPACITY
private volatile int maximumPoolSize;
```
* 内部类Worker

```Java
private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable
    {
        // 工作的线程
        final Thread thread;
        // 工作内容
        Runnable firstTask;
        /** Per-thread task counter */
        volatile long completedTasks;

        // 构造方法
        Worker(Runnable firstTask) {
            setState(-1); // inhibit interrupts until runWorker
            this.firstTask = firstTask;
            this.thread = getThreadFactory().newThread(this);
        }

        // runWorker方法等下面再看
        public void run() {
            runWorker(this);
        }
		
		// 以下为流程控制相关
        // Lock methods
        //
        // The value 0 represents the unlocked state.
        // The value 1 represents the locked state.

        protected boolean isHeldExclusively() {
            return getState() != 0;
        }

        protected boolean tryAcquire(int unused) {
            if (compareAndSetState(0, 1)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }

        protected boolean tryRelease(int unused) {
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }

        public void lock()        { acquire(1); }
        public boolean tryLock()  { return tryAcquire(1); }
        public void unlock()      { release(1); }
        public boolean isLocked() { return isHeldExclusively(); }

        void interruptIfStarted() {
            Thread t;
            if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
                try {
                    t.interrupt();
                } catch (SecurityException ignore) {
                }
            }
        }
    }
```
* 几个流程控制方法

```Java
// 私有方法
private void advanceRunState(int targetState)
final void tryTerminate()
private void interruptWorkers()
private void interruptIdleWorkers(boolean onlyOne)
// 呵呵
/**
 * Common form of interruptIdleWorkers, to avoid having to
 * remember what the boolean argument means.
 */
private void interruptIdleWorkers() {
    interruptIdleWorkers(false);
}
// 执行拒绝策略，会在execute中被调用
final void reject(Runnable command) {
        handler.rejectedExecution(command, this);
}
// Drains the task queue into a new list
private List<Runnable> drainQueue()

// public方法
// 可使用流程控制方法
public void shutdown()  
public List<Runnable> shutdownNow()  
protected void finalize()
public boolean prestartCoreThread()
void ensurePrestart() 
public int prestartAllCoreThreads()
public boolean remove(Runnable task)
public void purge()
// 获取各种状态
public boolean isShutdown()
public boolean isTerminating()
public boolean isTerminated()
public boolean awaitTermination(long timeout, TimeUnit unit)
        throws InterruptedException
// 其他各种getter，setter略过
```

* addWorker

```Java
private boolean addWorker(Runnable firstTask, boolean core) {
        ...
        //大段代码，检查队列的情况
        // boolean core
		// if true use corePoolSize as bound, else maximumPoolSize
        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
		        //各种检查
		        ...
                        workers.add(w);
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
	            // 添加worker或线程启动失败后进行回滚
                addWorkerFailed(w);
        }
        return workerStarted;
}
```

* runWorker 由Worker的run方法代理  

Main worker run loop.  Repeatedly gets tasks from queue and executes them  
```Java
final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
            while (task != null || (task = getTask()) != null) {
                w.lock();
                // 进行各种判断若不满足条件中断worker
                ...
                try {
	                // 空方法，可被子类覆盖，做些事情
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                        task.run();
                    } 
                    // 各种catch
                    ...
                    finally {
	                    // 同beforeExecute
                        afterExecute(task, thrown);
                    }
                } finally {
                    task = null;
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            processWorkerExit(w, completedAbruptly);
        }
    }
```

* 构造方法  

历经千幸万苦，总算看到了构造方法  
```Java
public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit, BlockingQueue<Runnable> workQueue);
 
public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit, BlockingQueue<Runnable> workQueue,ThreadFactory threadFactory);
 
public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit, BlockingQueue<Runnable> workQueue,RejectedExecutionHandler handler);
 
public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit, BlockingQueue<Runnable> workQueue,ThreadFactory threadFactory,RejectedExecutionHandler handler);
```
直接看最完整的这个，这些个变量在上面都大致介绍过了，跳过
```Java
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
}
```
* execute
```Java
public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        // If fewer than corePoolSize threads are running, try to
        // start a new thread with the given command as its first
        // task.
        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        // If a task can be successfully queued, then we still need
        // to double-check whether we should have added a thread
        // (because existing ones died since last checking) or that
        // the pool shut down since entry into this method. So we
        // recheck state and if necessary roll back the enqueuing if
        // stopped, or start a new thread if there are none.
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        // If we cannot queue task, then we try to add a new
        // thread.  If it fails, we know we are shut down or 
        // saturated and so reject the task.
        else if (!addWorker(command, false))
            reject(command);
    }
```

#### 总结
水平有限，大多内容都是从源码中进行复制粘贴，通过看ThreadPoolExecutor的源码，大致理清了线程池的使用流程及构建策略，通过组装各类BlockingQueue、ThreadFactory、RejectedExecutionHandler及线程池大小的设置，可以灵活应对各种使用场景，对于一些对定制化要求不高的场景，可以使用Executors中的静态工厂方法获取Executor。