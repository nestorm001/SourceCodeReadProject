### AsyncTaskLoader源码阅读

#### 前言  
前面看完了Loader、LoaderManager和AsyncTask的源码，接下来就是AsyncTaskLoader了，顾名思义，这个Loader使用了一个AsyncTask来实现异步调用，进行线程切换，将AsyncTask的流程与Loader的流程绑定    

#### 源码阅读  
* 定义  
```Java
public abstract class AsyncTaskLoader<D> extends Loader<D>
```

* 内部类LoadTask继承了AsyncTask，是AsyncTaskLoader异步加载的核心要素

从`AsyncTask<Void, Void, D>`可以看到，对于这个AsyncTask，我们只关注异步任务的结果`<D>`
```Java
final class LoadTask extends AsyncTask<Void, Void, D> implements Runnable {
        private final CountDownLatch mDone = new CountDownLatch(1);
        boolean waiting;

        /* Runs on a worker thread */
        @Override
        protected D doInBackground(Void... params) {
            try {
                D data = AsyncTaskLoader.this.onLoadInBackground();
                return data;
            } catch (OperationCanceledException ex) {
                if (!isCancelled()) {
                    throw ex;
                }
                return null;
            }
        }

        /* Runs on the UI thread */
        @Override
        protected void onPostExecute(D data) {
            try {
                AsyncTaskLoader.this.dispatchOnLoadComplete(this, data);
            } finally {
                mDone.countDown();
            }
        }

        /* Runs on the UI thread */
        @Override
        protected void onCancelled(D data) {
            try {
                AsyncTaskLoader.this.dispatchOnCancelled(this, data);
            } finally {
                mDone.countDown();
            }
        }

        /* Runs on the UI thread, when the waiting task is posted to a handler.
         * This method is only executed when task execution was deferred (waiting was true). */
        @Override
        public void run() {
            waiting = false;
            AsyncTaskLoader.this.executePendingTask();
        }
}
```
可以看到，所需要进行的异步任务在LoadTask中的doInBackground中执行，AsyncTask中的各种回调方法调用了AsyncTaskLoader中的各种对应的方法，因此，我们只要实现AsyncTaskLoader中对应的钩子（回调）方法，便可以进行异步任务的处理。

#### 总结  
各类Loader的大致操作流程如下  
==>	使用LoaderManager进行初始化init  
==>  由Activity等调用LoaderManager的start方法(?)
==>  由LoaderManager调用Loader的startLoading方法  
==>  调用Loader的onStartLoading回调，一般需要自己实现，并在其中根据是否已有结果调用deliverResult或forceLoad方法   
==>  正常流程，还未有结果，调用forceLoad方法后，通过各种手段执行异步任务  
==>  在获得任务结果后，Loader根据任务的状态(完成或取消)来分发结果  
==>	成功则进行成功的回调onLoadComplete，取消则进行取消的回调onLoadCanceled  
==>	最后由LoadManager调用方法callOnLoadFinished来调用回调方法callOnLoadFinished  

以上便是Loader的正常流程，除此之外，还有start对应的stop状态、abandon、reset、contentChanged、processingChange等状态及对应的流程就先跳过了

