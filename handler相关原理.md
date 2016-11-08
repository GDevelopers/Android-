#Handler、Looper、Message、MessageQueue原理及关系

##消息驱动的实现
- 消息的表示：Message
- 消息队列：MessageQueue
- 消息循环：Looper，用于循环取出消息进行处理
- 消息处理：Handler，从消息队列中取出消息后对消息进行处理

##消息发送与传递原理

1、Looper.prepare()：初始化一个MessageQueue，通过ThreadLocal类的set方法把新实例化的Looper对象设置进去。MessageQueue中包含消息，消息中含有一个字段target，属于Handler类型，当取到这个消息的时候，通过这个target调用dispatchMessage方法，可通过这个Handler处理相应的Message。

2、实例化Handler：在构造方法中关联Looper和MessageQueue

3、构造消息Message：通过obtain()方法从消息缓冲池中取出一个消息，并对消息中的参数进行赋值。可在此设置处理这个消息的runnable。

4、发送消息mHandler.sendMessage(msg)：此方法最终调用到sendMessageAtTime(msg, longMillis)方法，把消息放进一个队列当中，放进队列是通过关联的MessageQueue的enqueue方法，**并在此时把消息中的target设置为当前调用的Handler**。调用链为sendMessage()/post() -> enqueueMessage() -> queue.enqueueMessage() 。实际调用的处理方法handleMessage(msg)并不是在进消息队列的时候被调用。

5、从队列中循环取出消息Looper.loop()：通过queue.next()方法取出下一个消息msg，通过msg中的target变量(即与之关联的一个Handler变量)调用dispatchMessage()方法对消息进行处理。先处理对Message设置的Callback，再处理在Handler设置的Callback，这两个Callback都是一个Runnable接口，最后才调用handleMessage()方法处理这个Message消息。其中queue.next()调用了native方法实现，暂不做详述。

整个过程的调用链为：

-> Looper.prepare()，创建Looper、MessageQueue，关联当前线程 

-> new Handler()，内部关联Looper和MessageQueue 

-> Message.obtain()，创建消息，对消息进行赋值 

-> Handler.sendMessage()，发送消息，即设置进队的Message的target值为调用的Handler，通过MessageQueue对象queue把消息放进消息队列中 

-> Looper.loop()，通过关联的MessageQueue调用next()方法取出一个Message对象(注意此处的调用next()方法的队列并非一成不变，先是通过Looper.myLooper()取得当前线程的Looper，然后取出这个Looper关联的MessageQueue，调用的是这个MessageQueue的next方法)，通过Message对象中的target变量（即一个Handler）调用dispatchMessage()方法对消息做处理，先处理msg的callback，再到Handler的callback，然后到Handler的handleMessage()方法。

类跳转如下：（发送过程，不含实例化）

Handler.sendMessage() -> Handler.enqueueMessage() -> Message.target = this -> MessageQueue.enqueueMessage() -> Looper.loop() -> MessageQueue.next() -> Message.target.dispatchMessage() -> Handler.dispatchMessage() -> Handler.handleMessage()




##Looper
成员变量：

- MessageQueue mQueue：消息队列
- Thread mThread：Thread.currentThread()线程对象
- ThreadLocal<Looper> sThreadLocal
- Looper sMainLooper：静态变量，标识是否主线程的Looper，prepare(false)，消息循环不可退出

常用方法：

- Looper.prepare()
	- 用于初始化消息队列。调用后，会初始化MessageQueue和Looper。使用了ThreadLocal，每个线程只能对应于一个Looper
	- 调用了new Looper()构造方法，quitAllowed默认是true，表示消息循环是否可以退出
	- 通过ThreadLocal设置了当前的Looper值。Looper以线程为作用域，并且不同的线程有不同的数据副本，因此采用ThreadLocal。另外，ThreadLocal的get和set方法的操作对象都是当前线程的localValues对象的tables数组，其中数据的引用和对应的值在数组中的相邻位置上，如果知道数据引用的下标为index，则index+1就是数据引用对应的值。因此在不同的线程中访问同一个ThreadLocal的set和get方法，这两个方法所做的读写操作仅限于各自线程的内部。

```
public static void prepare() {
    prepare(true);
}

private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper(quitAllowed));
}
```	

- Looper构造方法
	- Looper初始化时，新建了MessageQueue对象保存在mQueue中。
	- 将Looper中的线程对象指向当前线程

```
private Looper(boolean quitAllowed) {
    mQueue = new MessageQueue(quitAllowed);
    mThread = Thread.currentThread();
}

// Message构造方法在Message类中，Message类并不是Looper的内部类
MessageQueue(boolean quitAllowed) {
    mQuitAllowed = quitAllowed;
    nativeInit();
}
```
- loop()
	- 每次从MessageQueue取出一个Message，调用msg.target.dispatchMessage(msg)分发消息，target将会在 Message被处理后会被recycle。当queue.next()返回null时会退出消息循环。

```
	public static void loop() {
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;

        // Make sure the identity of this thread is that of the local process,
        // and keep track of what that identity token actually is.
        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();

        for (;;) {
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }

            // This must be in a local variable, in case a UI event sets the logger
            Printer logging = me.mLogging;
            if (logging != null) {
                logging.println(">>>>> Dispatching to " + msg.target + " " +
                        msg.callback + ": " + msg.what);
            }

            msg.target.dispatchMessage(msg);

            if (logging != null) {
                logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
            }

            // Make sure that during the course of dispatching the
            // identity of the thread wasn't corrupted.
            final long newIdent = Binder.clearCallingIdentity();
            if (ident != newIdent) {
                Log.wtf(TAG, "Thread identity changed from 0x"
                        + Long.toHexString(ident) + " to 0x"
                        + Long.toHexString(newIdent) + " while dispatching to "
                        + msg.target.getClass().getName() + " "
                        + msg.callback + " what=" + msg.what);
            }

            msg.recycle();
        }
    }
```

- myLooper()

```
    public static @Nullable Looper myLooper() {
        return sThreadLocal.get();
    }
```

##Message
成员变量

- Message sPool：Message缓存池，指向一个缓存链表
- Message next：通过此对象实现了一个链表，详见obtain()和recycle()方法
- what、arg1、arg2、Object

常用方法：

- obtain()和recycle()
	- 调用此方法可在Message缓存池里面获得一个消息，Message使用后系统自动调用recycle()方法进行回收。如果直接new Message()，每次使用完后都放进缓存池里，则会导致缓存池中越来越多Message。

```
public static Message obtain() {
        synchronized (sPoolSync) {
            if (sPool != null) {
                Message m = sPool;
                sPool = m.next;
                m.next = null;
                sPoolSize--;
                return m;
            }
        }
        return new Message();
    }

public void recycle() {
        clearForRecycle();

        synchronized (sPoolSync) {
            if (sPoolSize < MAX_POOL_SIZE) {
                next = sPool;
                sPool = this;
                sPoolSize++;
            }
        }
    }
```

##Handler
成员变量

- Looper mLooper：调用了Looper.myLooper()方法关联了Handler所在的looper线程
- MessageQueue mQueue：消息队列，关联了Looper中消息队列


常用方法：

- Hander()构造方法：

```
public Handler(Callback callback, boolean async) {
        if (FIND_POTENTIAL_LEAKS) {
            final Class<? extends Handler> klass = getClass();
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                    klass.getCanonicalName());
            }
        }

        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
```


- sendMessageAtTime(Message msg, long uptimeMills)
	- Handler提供的所有发送消息的方法，最终都调用了这个方法来进行消息的发送。
	- 此方法的实质是把一个消息放进了一个消息队列当中。

```
public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, uptimeMillis);
    }
```
- enqueueMessage(MessageQueue queue， Message msg, long uptimeMillis)
	- 把消息放进消息队列的时候把message的target设置为当前的handler
	- 调用queue的enqueMessage方法，这个queue是关联的Looper线程中的消息队列。

```
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
}
```

- dispatchMessage(Message msg)
	- Handler用于分发和处理消息的方法。
	- 可对Message设置回调方法，将会在handleCallback(msg)进行处理。也可在Handler发送消息的时候传入Runnable接口，将会调用mCallback.handleMessage(msg)方法做处理。两个callback的类型都是Runnable接口类型。如果都没有设置，则会调用handleMessage(msg)做处理，handleMessage是一个空方法，需要重写该方法对消息进行处理。
	- 方法执行流程如下

```
    public void dispatchMessage(Message msg) {
        if (msg.callback != null) {
            handleCallback(msg);
        } else {
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            handleMessage(msg);
        }
    }
```

##MessageQueue

内部维护了一个单链表作为消息队列存储的数据结构。

成员变量

- Message mMessage：指向当前消息队列的队头元素

常用方法

- enqueueMessage(Message msg, long when)
	- 在进队之前，需要做如下判断：
		- 当前消息队列为空
		- 新添加的消息的执行时间when是0
		- 新添加的消息执行时间比消息队列头的消息执行时间更早
		
		代码描述为
		```
		if (p == null || when == 0 || when < p.when)
		```
		其中，p先被赋值为mMessage，即队头消息元素，when为消息的执行时间，如果满足这些条件，就把这个消息添加到队头（消息队列按时间先后排序），否则就要找到合适的位置将当前消息添加到队列。
		
	
	
```
final boolean enqueueMessage(Message msg, long when) {
        if (msg.isInUse()) {
            throw new AndroidRuntimeException(msg + " This message is already in use.");
        }
        if (msg.target == null) {
            throw new AndroidRuntimeException("Message must have a target.");
        }

        boolean needWake;
        synchronized (this) {
            if (mQuiting) {
                RuntimeException e = new RuntimeException(
                        msg.target + " sending message to a Handler on a dead thread");
                Log.w("MessageQueue", e.getMessage(), e);
                return false;
            }

            msg.when = when;
            Message p = mMessages;
            if (p == null || when == 0 || when < p.when) {
                // New head, wake up the event queue if blocked.
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
                // Inserted within the middle of the queue.  Usually we don't have to wake
                // up the event queue unless there is a barrier at the head of the queue
                // and the message is the earliest asynchronous message in the queue.
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
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
        }
        if (needWake) {
            nativeWake(mPtr);
        }
        return true;
    }
```
	