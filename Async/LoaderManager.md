### LoaderManager源码阅读  

#### 前言  
前面看完了Loader的源码，要使用Loader的话，可以通过在Activity和Fragment中调用getLoaderManager()获取LoaderManager或实例化AsyncTaskLoader和CursorLoader来进行各种处理，当然，也可以自己实现一个Loader的子类。    

#### 源码阅读  
* 从抽象类LoaderManager开始  

接口LoaderCallbacks，定义了三个回调方法
```Java
	public interface LoaderCallbacks<D> {
        public Loader<D> onCreateLoader(int id, Bundle args);
        public void onLoadFinished(Loader<D> loader, D data);
        public void onLoaderReset(Loader<D> loader);
    }
```
用于操作loader的几个抽象方法
```Java
public abstract <D> Loader<D> initLoader(int id, Bundle args,
            LoaderManager.LoaderCallbacks<D> callback);
public abstract <D> Loader<D> restartLoader(int id, Bundle args,
            LoaderManager.LoaderCallbacks<D> callback);
public abstract void destroyLoader(int id);
public abstract <D> Loader<D> getLoader(int id);
```

* LoaderManager的实现类LoaderManagerImpl   

getLoaderManager()最终获取的也是这个实现类的实例

在LoaderManagerImpl中有一个内部类LoaderInfo
```Java
final class LoaderInfo implements Loader.OnLoadCompleteListener<Object>,
            Loader.OnLoadCanceledListener<Object>
```
其内部有Loader对象，并实现了Loader的两个接口，内部的方法主要进行一些Loader方法的封装和代理

* LoaderManagerImpl中的方法  

initLoader，log相关代码已忽略
```Java
public <D> Loader<D> initLoader(int id, Bundle args, LoaderManager.LoaderCallbacks<D> callback) {
        LoaderInfo info = mLoaders.get(id);
        if (info == null) {
            info = createAndInstallLoader(id, args,  (LoaderManager.LoaderCallbacks<Object>)callback);  
        } else {     
            info.mCallbacks = (LoaderManager.LoaderCallbacks<Object>)callback;
        }
        if (info.mHaveData && mStarted) {
            info.callOnLoadFinished(info.mLoader, info.mData);
        }
        return (Loader<D>)info.mLoader;
}
```
在LoadManagerImpl中维护了一个`SparseArray<LoaderInfo>` mLoaders，用于存放LoaderInfo，根据mLoaders中是否有该id的Loader来选择新建或重用

restartLoader，log相关代码已忽略(留着注释...)
```Java
public <D> Loader<D> restartLoader(int id, Bundle args, LoaderManager.LoaderCallbacks<D> callback) {       
        LoaderInfo info = mLoaders.get(id);
        if (info != null) {
            LoaderInfo inactive = mInactiveLoaders.get(id);
            if (inactive != null) {
                if (info.mHaveData) {
                    // This loader now has data...  we are probably being
                    // called from within onLoadComplete, where we haven't
                    // yet destroyed the last inactive loader.  So just do
                    // that now.
                    inactive.mDeliveredData = false;
                    inactive.destroy();
                    info.mLoader.abandon();
                    mInactiveLoaders.put(id, info);
                } else {
                    // We already have an inactive loader for this ID that we are
                    // waiting for!  What to do, what to do...
                    if (!info.mStarted) {
                        // The current Loader has not been started...  we thus
                        // have no reason to keep it around, so bam, slam,
                        // thank-you-ma'am.
                        mLoaders.put(id, null);
                        info.destroy();
                    } else {
                        // Now we have three active loaders... we'll queue
                        // up this request to be processed once one of the other loaders
                        // finishes or is canceled.
                        info.cancel();
                        if (info.mPendingLoader != null) {
                            info.mPendingLoader.destroy();
                            info.mPendingLoader = null;
                        }
                        info.mPendingLoader = createLoader(id, args, 
                                (LoaderManager.LoaderCallbacks<Object>)callback);
                        return (Loader<D>)info.mPendingLoader.mLoader;
                    }
                }
            } else {
                // Keep track of the previous instance of this loader so we can destroy
                // it when the new one completes.
                info.mLoader.abandon();
                mInactiveLoaders.put(id, info);
            }
        }
        
        info = createAndInstallLoader(id, args,  (LoaderManager.LoaderCallbacks<Object>)callback);
        return (Loader<D>)info.mLoader;
}
```
在LoadManagerImpl中维护了一个`SparseArray<LoaderInfo>` mInactiveLoaders，用于存放之前运行的Loader的LoaderInfo，根据mInactiveLoaders和mLoaders中是否有该id的Loader来进行各种处理

注：init和restart方法中都会判断是否正在创建一个loader，如果是会抛出异常  

上面两个方法在loader不存在的时候，都会调用createAndInstallLoader方法，如下
```Java
private LoaderInfo createAndInstallLoader(int id, Bundle args,
            LoaderManager.LoaderCallbacks<Object> callback) {
        try {
            mCreatingLoader = true;
            LoaderInfo info = createLoader(id, args, callback);
            installLoader(info);
            return info;
        } finally {
            mCreatingLoader = false;
        }
}

private LoaderInfo createLoader(int id, Bundle args,
            LoaderManager.LoaderCallbacks<Object> callback) {
        LoaderInfo info = new LoaderInfo(id, args,  (LoaderManager.LoaderCallbacks<Object>)callback);
        Loader<Object> loader = callback.onCreateLoader(id, args);
        info.mLoader = (Loader<Object>)loader;
        return info;
}
 
void installLoader(LoaderInfo info) {
        mLoaders.put(info.mId, info);
        if (mStarted) {
            // The activity will start all existing loaders in it's onStart(),
            // so only start them here if we're past that point of the activitiy's
            // life cycle
            info.start();
        }
}
```
在这里回调方法onCreateLoader会被调用

destroyLoader和getLoader的实现方法，比较简单，略过
```Java
public void destroyLoader(int id)
public <D> Loader<D> getLoader(int id)
```
各种对LoaderManager中所有loader进行操作的方法，分别对应LoaderInfo中的各种方法
```Java
doStart()
doStop()
doRetain()
finishRetain()
doReportNextStart()
doReportStart()
doDestroy()
```

#### 总结
LoaderManger主要还有用于管理各路Loader，真正的Loader实现，处理等，还是要在Loader的两个子类AsyncTaskLoader和CursorLoader中进行研究学习