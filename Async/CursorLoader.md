### CursorLoader源码阅读

#### 前言  
CursorLoader 是AsyncTaskLoader的子类，并实例化了Loader的内部类ForceLoadContentObserver作为observer使用，在loadInBackground中使用 `cursor.registerContentObserver(mObserver)` 进行注册
#### 源码阅读  
* 定义  
显然，返回的是个Cursor
```Java
public class CursorLoader extends AsyncTaskLoader<Cursor>
```

* 构造方法
```Java
/**
     * Creates an empty unspecified CursorLoader.  You must follow this with
     * calls to {@link #setUri(Uri)}, {@link #setSelection(String)}, etc
     * to specify the query to perform.
     */
    public CursorLoader(Context context) {
        super(context);
        mObserver = new ForceLoadContentObserver();
    }

    /**
     * Creates a fully-specified CursorLoader.  See
     * {@link ContentResolver#query(Uri, String[], String, String[], String)
     * ContentResolver.query()} for documentation on the meaning of the
     * parameters.  These will be passed as-is to that call.
     */
    public CursorLoader(Context context, Uri uri, String[] projection, String selection,
            String[] selectionArgs, String sortOrder) {
        super(context);
        mObserver = new ForceLoadContentObserver();
        mUri = uri;
        mProjection = projection;
        mSelection = selection;
        mSelectionArgs = selectionArgs;
        mSortOrder = sortOrder;
    }
```

* 调用周期中的各种操作流程
```Java
	// Must be called from the UI thread
    @Override
    protected void onStartLoading() {
        if (mCursor != null) {
            deliverResult(mCursor);
        }
        if (takeContentChanged() || mCursor == null) {
            forceLoad();
        }
    }
	
	// Must be called from the UI thread
    @Override
    protected void onStopLoading() {
        // Attempt to cancel the current load task if possible.
        cancelLoad();
    }

	@Override
    public void onCanceled(Cursor cursor) {
        if (cursor != null && !cursor.isClosed()) {
            cursor.close();
        }
    }

	@Override
    protected void onReset() {
        super.onReset();
        
        // Ensure the loader is stopped
        onStopLoading();

        if (mCursor != null && !mCursor.isClosed()) {
            mCursor.close();
        }
        mCursor = null;
    }
```

#### 总结
在看完Loader、LoaderManager、AsyncTaskLoader时候再看CursorLoader还是比较清晰和简单的

