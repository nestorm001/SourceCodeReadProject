### Message源码阅读

#### 前言
Message作为Handler和Looper间传递的主体，相对比较简单，大概过一遍


#### 源码阅读

* 变量定义
```Java
// 消息的标识
public int what;
// 两个int，用于轻量数据传输
public int arg1; 
public int arg2;
// message需要传输的实际内容
public Object obj;

// Optional 一般成对使用，暂时略过
public Messenger replyTo;
public int sendingUid = -1;

// 变量如其名
/*package*/ Handler target;

// 变量如其名 
// 之前看到过，如果message本身有callback，handler就不会处理这个
// message了   
/*package*/ Runnable callback;
// 链表结点式使用方式，会在MessageQueue里看到    
// sometimes we store linked lists of these things
/*package*/ Message next;
```

handler获取发送message的主要流程
不打破链式调用的典范，又工厂又代理的，很设计，很模式
```Java
handler.obtainMessage().sendToTarget();
// Handler类中的方法
// message内部会保存handler的引用target
public final Message obtainMessage() {
     return Message.obtain(this);
}
// Message类中的方法，代理了handler的sendMessage
public void sendToTarget() {
    target.sendMessage(this);
}
```

* 各种获取Message的工厂方法
```Java
	// 有一个全局Message池，按链表来存储
	// 相关变量定义
	private static final Object sPoolSync = new Object();
	// message pool头结点,每次obtain从链表中取出头结点
    private static Message sPool;
    private static int sPoolSize = 0;
    private static final int MAX_POOL_SIZE = 50;
    // 从message pool中获取message实例，若为空则new一个出来
    public static Message obtain() {
        synchronized (sPoolSync) {
            if (sPool != null) {
                Message m = sPool;
                sPool = m.next;
                m.next = null;
                m.flags = 0; // clear in-use flag
                sPoolSize--;
                return m;
            }
        }
        return new Message();
    }
    
    // 从传入Message中复制
	public static Message obtain(Message orig)
	// 使用工厂方法获取message, 对其中对应域赋值
	public static Message obtain(Handler h, Runnable callback)
	public static Message obtain(Handler h, int what)
	public static Message obtain(Handler h, int what, Object obj)
	public static Message obtain(Handler h, int what, int arg1, int arg2)
	public static Message obtain(Handler h, int what, 
            int arg1, int arg2, Object obj)
```

各种getter、setter略过

* 总结
Message主要是用于异步过程中存储和传递的实体，对于Message队列的具体控制主要还是在MessageQueue中