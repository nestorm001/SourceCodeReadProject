### Loader源码阅读  

#### 前言  
看源码学习Loader的使用    
Loader，用于异步加载数据，有AsyncTaskLoader和CursorLoader两个子类    

#### 关于线程的官方说明
Note on threading: Clients of loaders should as a rule perform any calls on to a Loader from the main thread of their process (that is, the thread the Activity callbacks and other things occur on). Subclasses of Loader (such as AsyncTaskLoader) will often perform their work in a separate thread, but when delivering their results this too should be done on the main thread.   
Subclasses generally must implement at least onStartLoading(), onStopLoading(), onForceLoad(), and onReset().

#### 源码阅读  
* 从类中定义的变量开始  
```Java
	// 用于标识Loader的id
	int mId;
	// 很直观，Loader工作完成时的回调接口
    OnLoadCompleteListener<D> mListener;
    // Loader取消时的回调接口
    OnLoadCanceledListener<D> mOnLoadCanceledListener;
    // 从构造函数中的context中获取的application 
    // context(在非必要的情况下，持有Application的context
    // 比持有Activity的context更为安全，减少内存泄漏的风险)
    Context mContext;
    // 各路flag，变量名代表一切
    boolean mStarted = false;
    boolean mAbandoned = false;
    boolean mReset = true;
    boolean mContentChanged = false;
    boolean mProcessingChange = false;
```
* 内部类ForceLoadContentObserver，顾名思义，监听内容变化的Observer  
```Java
public final class ForceLoadContentObserver extends ContentObserver {
        public ForceLoadContentObserver() {
            super(new Handler());
        }

        @Override
        public boolean deliverSelfNotifications() {
            return true;
        }

        @Override
        public void onChange(boolean selfChange) {
            onContentChanged();
        }
}
```
* 两个回调接口  
```Java
public interface OnLoadCompleteListener<D> {
        public void onLoadComplete(Loader<D> loader, D data);
}

public interface OnLoadCanceledListener<D> {
        public void onLoadCanceled(Loader<D> loader);
}
```

* 构造方法  

如上文所述，使用application的context  
```Java
public Loader(Context context) {
        mContext = context.getApplicationContext();
}
```
* 回调方法的调用时机  
```Java
public void deliverResult(D data) {
        if (mListener != null) {
            mListener.onLoadComplete(this, data);
        }
}

public void deliverCancellation() {
        if (mOnLoadCanceledListener != null) {
            mOnLoadCanceledListener.onLoadCanceled(this);
        }
}
```
* 两个回调接口的注册与注销方法
```Java
public void registerListener(int id, OnLoadCompleteListener<D> listener)
public void unregisterListener(OnLoadCompleteListener<D> listener)
public void registerOnLoadCanceledListener(OnLoadCanceledListener<D> listener)
public void unregisterOnLoadCanceledListener(OnLoadCanceledListener<D> listener)
```

* Loader的一些操作  

启动(main thread)  
其中onStartLoading()回调需要实现类进行覆盖来进行一些操作
```Java
public final void startLoading() {
        mStarted = true;
        mReset = false;
        mAbandoned = false;
        onStartLoading();
}
```  
 取消(main thread)  
其中onCancelLoad()回调需要实现类进行覆盖来进行一些操作  
```Java
public boolean cancelLoad() {
        return onCancelLoad();
}
```
强制加载(main thread)  
会忽视前一次的结果  
其中onForceLoad()回调需要实现类进行覆盖来进行一些操作  
```Java
public void forceLoad() {
        onForceLoad();
}
```
停止加载(main thread)  
其中onStopLoading()回调需要实现类进行覆盖来进行一些操作  
```Java
public void stopLoading() {
        mStarted = false;
        onStopLoading();
}
```
放弃加载，一般由LoaderManager进行调用，不应该手动调用  
其中onAbandon()回调需要实现类进行覆盖来进行一些操作 
```Java
public void abandon() {
        mAbandoned = true;
        onAbandon();
}
```
重置(main thread)  
其中onReset()回调需要实现类进行覆盖来进行一些操作
```Java
public void reset() {
        onReset();
        mReset = true;
        mStarted = false;
        mAbandoned = false;
        mContentChanged = false;
        mProcessingChange = false;
}
```

其他还有一些getter、setter还有序列化的方法就不赘述了

#### 总结
Loader类作为AsyncTaskLoader和CursorLoader的非抽象基类，主要还是搭了一个框架，相比于直接实例化，似乎用子类继承它，并进行流程的处理和回调函数的扩充更为合适