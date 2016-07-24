### AsyncTask源码阅读

#### 前言
众所周知，在Android开发中，主线程中不应进行耗时操作，阻塞UI线程，造成界面卡顿，甚至ANR，特别是在主线程中进行网络请求会导致NetworkOnMainThreadException。  
在Android和Java中，提供了很多进行异步任务的方法，如Thread、ThreadPoolExecutor、Handler、AsyncTask等。  
在RxJava之前，AsyncTask做为一个被封装的较为完善的异步任务框架，被广泛使用。  
AsyncTask正如其名，主要用于进行异步任务，内部通过Message、Handler和ThreadPoolExecutor构建了一整套处理框架，在异步线程进行耗时处理，在主线程进行UI操作。  


#### 源码阅读

* 定义
```Java
public abstract class AsyncTask<Params, Progress, Result>
```
Params : 异步任务(doInBackground)执行时所用的参数类型  
Progress : 异步任务执行过程中更新进度(onProgressUpdate)所用的类型  
Result : 异步任务结束后所返回的类型(onPostExecute)  
当某个参数并不需要时，可用Void作为类型(从RxBinding里面学来的)  

* 内部定义的Handler
```Java
private static InternalHandler sHandler;
```
InternalHandler 的构造和定义
```Java
private static class InternalHandler extends Handler {
        public InternalHandler() {
            super(Looper.getMainLooper());
        }

        @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
        @Override
        public void handleMessage(Message msg) {
            AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
            switch (msg.what) {
                case MESSAGE_POST_RESULT:
                    // There is only one result
                    result.mTask.finish(result.mData[0]);
                    break;
                case MESSAGE_POST_PROGRESS:
                    result.mTask.onProgressUpdate(result.mData);
                    break;
            }
        }
}
```  
可以看到sHandler的使用Looper.getMainLooper()作为构造参数，因此，会在主线程进行各种处理: finish(Result result)和onProgressUpdate(Progress... values)。  

* AsyncTask运行过程中的各种回调  
```Java  
// 在主线程调用，用于任务开始前的一些处理
@MainThread protected void onPreExecute() {}

// 在Worker线程调用，任务的主要工作就是在这里进行的，必须要override
@WorkerThread protected abstract Result doInBackground(Params... params);

// 在主线程调用，用于任务结束获取到结果后进行的处理
 @MainThread protected void onPostExecute(Result result) {}
 
// 在主线程调用，用于任务进行时，更新进度
// 通过在doInBackground中调用publishProgress来更新进度
@MainThread protected void onProgressUpdate(Progress... values) {}

// 在主线程调用，当doInBackground完成后调用cancel时执行
@MainThread protected void onCancelled(Result result) {
        onCancelled();
} 

// 同上
@MainThread protected void onCancelled() {}
```  
知道了这些回调，就该看这些回调会在什么地方执行了  
AsyncTask的执行方法execute(Params... params)
```Java  
@MainThread public final AsyncTask<Params, Progress, Result> execute(Params... params) {
        return executeOnExecutor(sDefaultExecutor, params);
}
```
可以看到execute调用了executeOnExecutor(Executor exec, Params... params)，接着往下看  
```Java
@MainThread public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
            Params... params) {
        if (mStatus != Status.PENDING) {
			//判断运行状态，抛出异常
			//AsyncTask的状态有PENDING, RUNNING, FINISHED，比较直观
			...
        }

        mStatus = Status.RUNNING;
        onPreExecute();
        mWorker.mParams = params;
        exec.execute(mFuture);
        return this;
}
```
可以看到，在任务执行之前，调用了onPreExecute()

mWorker为WorkerRunnable，需要实现Callable接口，定义如下
```Java
private static abstract class WorkerRunnable<Params, Result> implements Callable<Result> {
        Params[] mParams;
}  
```
mFuture使用了FutureTask的构造方法
```Java
public FutureTask(Callable<V> callable)
```
mWorker和mFuture在AsyncTask的构造函数中进行初始化，方法如下
```Java
	    mWorker = new WorkerRunnable<Params, Result>() {
            public Result call() throws Exception {
                mTaskInvoked.set(true);
	            Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                //noinspection unchecked
                Result result = doInBackground(mParams);
                Binder.flushPendingCommands();
                return postResult(result);
            }
        };

        mFuture = new FutureTask<Result>(mWorker) {
            @Override
            protected void done() {
                try {
                    postResultIfNotInvoked(get());
                } catch (InterruptedException e) {
                    android.util.Log.w(LOG_TAG, e);
                } catch (ExecutionException e) {
                    throw new RuntimeException("An error occurred while executing doInBackground()",
                            e.getCause());
                } catch (CancellationException e) {
                    postResultIfNotInvoked(null);
                }
            }
        };
```
从上面可以看到mFuture构造时，使用了实现了Callable的mWorker  
在FutureTask中的run()方法中会执行Callable的call()方法，并且，不论成功失败亦或是task被取消，都会去调用FutureTask的done()方法  
正常情况mWorker的call()调用结束后会调用postResult(Result result)，若call()未被调用，结果还是会在mFuture 的done()中处理  

postResult(Result result)  
```Java
private Result postResult(Result result) {
        @SuppressWarnings("unchecked")
        Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
                new AsyncTaskResult<Result>(this, result));
        message.sendToTarget();
        return result;
}
```

postResultIfNotInvoked
```Java
private void postResultIfNotInvoked(Result result) {
        final boolean wasTaskInvoked = mTaskInvoked.get();
        if (!wasTaskInvoked) {
            postResult(result);
        }
}
```
从上面的方法可以看到，除非在done时发生了异常，其他情况下postResult都会被调用。
在postResult中handler获取了MESSAGE_POST_RESULT
这时会调用AsyncTask的finish(Result result)方法
```Java
private void finish(Result result) {
        if (isCancelled()) {
            onCancelled(result);
        } else {
            onPostExecute(result);
        }
        mStatus = Status.FINISHED;
}
```
finish中根据任务是否被取消，分别调用onPostExecute(Result result)和onCancelled(Result result)这两个回调。

#### 总结
至此AsyncTask的大致流程源码也看完了，里面还有不少待深究的东西，如Handler、Looper和Message的关系，AsyncTask内定义的两个Executor(THREAD_POOL_EXECUTOR, SERIAL_EXECUTOR)等