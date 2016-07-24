### MessageQueue源码阅读  

#### 前言  
Handler、Message和Looper都看完了，Handler进行异步任务的基本原理也清楚了，只剩最后一个MessageQueue需要再细细研究了。MessageQueue主要负责~~存储~~管理Message，并按需要(主要为Message发送时间)，根据分发策略分发Message，Looper仅调用MessageQueue的方法queue.next()来获取Message。
#### 源码阅读  
* 变量定义
```Java
// True if the message queue can be quit.
private final boolean mQuitAllowed;
// 看似只有一个Message，其实是当前的Message链表表头
Message mMessages;

private SparseArray<FileDescriptorRecord> mFileDescriptorRecords;

private boolean mQuitting;

// Indicates whether next() is blocked waiting in pollOnce() with a non-zero timeout.
private boolean mBlocked;

// The next barrier token.
// Barriers are indicated by messages with a null target whose arg1 field carries the token.
private int mNextBarrierToken;
```
* native方法
```java
private native static long nativeInit();
private native static void nativeDestroy(long ptr);
private native void nativePollOnce(long ptr, int timeoutMillis); /*non-static for callbacks*/
private native static void nativeWake(long ptr);
private native static boolean nativeIsPolling(long ptr);
private native static void nativeSetFileDescriptorEvents(long ptr, int fd, int events);
```
* 构造方法
```java
// 为package的，不为外部所动
MessageQueue(boolean quitAllowed) {
        mQuitAllowed = quitAllowed;
        mPtr = nativeInit();
}
```
各种方法，OnFileDescriptorEventListener意义不明，先跳过  
```java
// return True if the looper is idle.
// 当message链表为空或当前无message需要发送时即为空闲状态
public boolean isIdle() {
     synchronized (this) {
         final long now = SystemClock.uptimeMillis();
         return mMessages == null || now < mMessages.when;
    }
}
// return True if the looper is currently polling for events.
// @hide
public boolean isPolling()
```

* next 复杂的流程，需要好好梳理一下
```java
Message next() {
        // Return here if the message loop has already quit and been disposed.
        ...

        int pendingIdleHandlerCount = -1; // -1 only during first iteration
        int nextPollTimeoutMillis = 0;
        for (;;) {
            if (nextPollTimeoutMillis != 0) {
                Binder.flushPendingCommands();
            }

            nativePollOnce(ptr, nextPollTimeoutMillis);

            synchronized (this) {
                // 开始在链表中找下一个可用于返回的Message
                // 各种链表遍历
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                Message msg = mMessages;
                if (msg != null && msg.target == null) {
                    // Stalled by a barrier.  Find the next asynchronous message in the queue.
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                if (msg != null) {
                    if (now < msg.when) {
                        // Next message is not ready.  Set a timeout to wake up when it is ready.
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        // 正常情况下真正返回的message
                        // 从链表中移除
                        mBlocked = false;
                        if (prevMsg != null) {
                            prevMsg.next = msg.next;
                        } else {
                            mMessages = msg.next;
                        }
                        msg.next = null;
                        msg.markInUse();
                       
                        return msg;
                    }
                } else {
                    // No more messages.
                    nextPollTimeoutMillis = -1;
                }

                // Process the quit message now that all pending messages have been handled.
                if (mQuitting) {
                    dispose();
                    return null;
                }

                // 大段代码，获取需要执行的idleHandler队列
                ...
            }

            // 大段代码，执行idleHandler队列，调用回调方法
            ...

			// 重置idleHandler队列数量为0
            pendingIdleHandlerCount = 0;
            // 任务执行idleHandler的回调时，可能会有新Message
            // 立即进行下一次native方法调用
            nextPollTimeoutMillis = 0;
        }
    }
```
* quit
```java
void quit(boolean safe) {
        if (!mQuitAllowed) {
            throw new IllegalStateException("Main thread not allowed to quit.");
        }

        synchronized (this) {
            if (mQuitting) {
                return;
            }
            mQuitting = true;

            if (safe) {
                removeAllFutureMessagesLocked();
            } else {
                removeAllMessagesLocked();
            }

            // We can assume mPtr != 0 because mQuitting was previously false.
            nativeWake(mPtr);
        }
    }
```
* enqueueMessage
```java
boolean enqueueMessage(Message msg, long when) {
        // 做些判断
		...
		
        synchronized (this) {
            // 做些判断
			...

            msg.markInUse();
            msg.when = when;
            Message p = mMessages;
            boolean needWake;
            // 当表头为空，或message执行时间为0或小于表头p
            // msg作为新的表头
            if (p == null || when == 0 || when < p.when) {
                // New head, wake up the event queue if blocked.
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
                
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
                // 遍历链表寻找插入结点
                for (;;) {
                    prev = p;
                    p = p.next;
                    if (p == null || when < p.when) {
                        break;
                    }
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                msg.next = p; // invariant: p == prev.next
                prev.next = msg;
            }

            // We can assume mPtr != 0 because mQuitting is false.
            if (needWake) {
                nativeWake(mPtr);
            }
        }
        return true;
    }
```
```java
// return A token that uniquely identifies the barrier
public int postSyncBarrier()
public void removeSyncBarrier(int token)

// 各种判断Message存在，清除即取消Message的方法，略过
boolean hasMessages(Handler h, int what, Object object)
boolean hasMessages(Handler h, Runnable r, Object object)
void removeMessages(Handler h, int what, Object object)
void removeMessages(Handler h, Runnable r, Object object)
void removeCallbacksAndMessages(Handler h, Object object)
// 这两个方法只有在队列要退出的时候才会被调用
private void removeAllMessagesLocked()
private void removeAllFutureMessagesLocked() 
```

IdleHandler会在queue空闲时调用（队列为空或队列中的消息发送时间都在当前时间之后）
```java
// 表示等待中的IdleHandler队列，会在next()中使用
// mIdleHandlers用于存储IdleHandler，而mPendingIdleHandlers为
// 临时使用的数组
// 在next中检测到队列空闲时，使用循环遍历，调用队列中每一个
// IdleHandler的回调方法queueIdle()
private final ArrayList<IdleHandler> mIdleHandlers = new ArrayList<IdleHandler>();
private IdleHandler[] mPendingIdleHandlers;



public void addIdleHandler(@NonNull IdleHandler handler)
public void removeIdleHandler(@NonNull IdleHandler handler)

public static interface IdleHandler {
    boolean queueIdle();
}
```
FileDescriptorEvent待研究
```java

public void addOnFileDescriptorEventListener(@NonNull FileDescriptor fd,
            @OnFileDescriptorEventListener.Events int events,
            @NonNull OnFileDescriptorEventListener listener)
public void removeOnFileDescriptorEventListener(@NonNull FileDescriptor fd)
private void updateOnFileDescriptorEventListenerLocked(FileDescriptor fd, int events,
            OnFileDescriptorEventListener listener)

    /**
     * A listener which is invoked when file descriptor related events occur.
     */
    public interface OnFileDescriptorEventListener {
        public static final int EVENT_INPUT = 1 << 0;
        public static final int EVENT_OUTPUT = 1 << 1;
        public static final int EVENT_ERROR = 1 << 2;
        
        @Events int onFileDescriptorEvents(@NonNull FileDescriptor fd, @Events int events);
    }

    private static final class FileDescriptorRecord {
        public final FileDescriptor mDescriptor;
        public int mEvents;
        public OnFileDescriptorEventListener mListener;
        public int mSeq;

        public FileDescriptorRecord(FileDescriptor descriptor,
                int events, OnFileDescriptorEventListener listener) {
            mDescriptor = descriptor;
            mEvents = events;
            mListener = listener;
        }
	}
```

* 总结
MessageQueue虽然叫Queue，但主要维护的是一个链表，链表中每个结点都是一个Message，结点根据时间排序，表头为需要发送的最新的Message，其中的主要方法enqueueMessage、removeMessage、next等主要就是对链表进行各种操作(结点交换、删除、添加。。。)  
MessageQueue的作用大致理清了，但其中还有些意义不太明确FileDescriptorRecord和SyncBarrier、各种native方法，先放着，以后再看