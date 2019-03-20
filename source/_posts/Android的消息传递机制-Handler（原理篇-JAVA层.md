---
title: Android的消息传递机制-Handler（原理篇-JAVA层）
date: 2019-03-20 20:01:10
---

从上一篇文章[Android消息传递机制Handler-使用篇](https://chaojian.github.io/2019/03/10/Android%E6%B6%88%E6%81%AF%E4%BC%A0%E9%80%92%E6%9C%BA%E5%88%B6Handler-%E4%BD%BF%E7%94%A8%E7%AF%87/)我们已经知道Handler的应用场景以及怎样使用Handler，本篇文章将从源码的层面，分析Handler的原理。

## 1. 先来看一下Handler的构造函数

```

// Handler.java
public Handler() {
    this(null, false);
}
    
public Handler(Callback callback) {
    this(callback, false);
} 

public Handler(Looper looper) {
    this(looper, null, false);
}

public Handler(Looper looper, Callback callback) {
    this(looper, callback, false);
}

@hide
public Handler(boolean async) {
    this(null, async);
}

@hide
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

@hide
public Handler(Looper looper, Callback callback, boolean async) {
        mLooper = looper;
        mQueue = looper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
```

从上述的构造函数可以知道，实际上通过外部创建的Handler最终都会调到以下两个hide的构造函数，分析两者的共同点：

```
//Handler.java
@hide
public Handler(Callback callback, boolean async) {
        ...
        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
}

@hide
public Handler(Looper looper, Callback callback, boolean async) {
    mLooper = looper;
    mQueue = looper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}
```

从上述代码可以发现，两者的区别是一个是通过Looper.myLooper()来给mLooper赋值，一个则是从构造函数中传过来的。Looper.myLooper()的源码如下，实际就是通过sThreadLocal.get()来获取当前的Looper对象:

```
/**
     * Return the Looper object associated with the current thread.  Returns
     * null if the calling thread is not associated with a Looper.
     */
    public static @Nullable Looper myLooper() {
        return sThreadLocal.get();
    }
```

知道了通过sThreadLocal.get()来获取当前Looper对象的，那么sThreadLocal是从哪里set该Looper对象的呢？我们再来看以下源码：

```
// Looper.java
 /** Initialize the current thread as a looper.
      * This gives you a chance to create handlers that then reference
      * this looper, before actually starting the loop. Be sure to call
      * {@link #loop()} after calling this method, and end it by calling
      * {@link #quit()}.
      */
    public static void prepare() {
        prepare(true);
    }

    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }

    /**
     * Initialize the current thread as a looper, marking it as an
     * application's main looper. The main looper for your application
     * is created by the Android environment, so you should never need
     * to call this function yourself.  See also: {@link #prepare()}
     */
    public static void prepareMainLooper() {
        prepare(false);
        synchronized (Looper.class) {
            if (sMainLooper != null) {
                throw new IllegalStateException("The main Looper has already been prepared.");
            }
            sMainLooper = myLooper();
        }
    }

    /**
     * Returns the application's main looper, which lives in the main thread of the application.
     */
    public static Looper getMainLooper() {
        synchronized (Looper.class) {
            return sMainLooper;
        }
    }
```

通过源码可知，再ActivityThread的入口main函数中，即程序的入口，在此处调用了Looper.prepareMainLooper(),此时创建了一个不允许退出的Looper对象，此时把该Looper对象set进了ThreadLocal当中，所以此时如果我们是在主线程中创建的Handler默认拿到就是通过ActivityThread创建的Looper。

注意：在这里也可以解释为，我们创建子线程的时候，默认不会像主线程那样帮助我们创建Looper的，如果要在子线程创建Handler，需要在主线程当中先调用Looper.prepare()方法进行创建子线程对应的Looper对象，然后再创建Handler对象。

**注意⚠️：Looper.loop()方法，我们先卖个关子，在后面处理消息的章节会说到**

```
// ActivityThread.java
public static void main(String[] args) {
        ...
        Looper.prepareMainLooper();

        ActivityThread thread = new ActivityThread();
        ...
        
        Looper.loop(); 
    }
```

至此，我们知道怎么把Looper对象的由来了，那么Looper对象是怎么跟我们的线程关联起来的呢？这时候我们需要分析以下ThreadLocal的源码如下的get(),set()方法，如下：

```

public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}

public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null)
            return (T)e.value;
    }
    return setInitialValue();
}

```

从上述源码可以知道，实际上ThreadLocal就是通过绑定Thread.currentThread()来绑定当前线程对应的value值，为什么笔者话费那么多时间来分析Looper对象，当前不是空穴来风，后面的分析当中，Looper也作为一个重要的角色，其实从命名，我们也大概能猜测到Looper对象的作用。
创建一个Handler对象，除了Looper对象，同时也初始化了赋值了一个callback，一个mQueue，同时标记是否同步。接下来，我们继续往下看：

## 2. 创建完Handler后，Handler的消息传递的原理是怎样的呢？

### 2.1 Handler是如何构建消息队列的。

从上一篇文章[Android消息传递机制Handler-使用篇](https://chaojian.github.io/2019/03/10/Android%E6%B6%88%E6%81%AF%E4%BC%A0%E9%80%92%E6%9C%BA%E5%88%B6Handler-%E4%BD%BF%E7%94%A8%E7%AF%87/)我们已经知道了Handler消息传递的两种方式了：
（1）post or postDelay
（2）sendMeesage or sendMeesageDelayed

接下来我们围绕这两种方式来分析他们的实现原理，看他们是怎样构建一个消息队列的，如下源码：

```
public final boolean post(Runnable r)
{
    return  sendMessageDelayed(getPostMessage(r), 0);
}
   
public final boolean postDelayed(Runnable r, long delayMillis)
{
    return sendMessageDelayed(getPostMessage(r), delayMillis);
} 

public final boolean sendMessage(Message msg)
{
    return sendMessageDelayed(msg, 0);
}

public final boolean sendMessageDelayed(Message msg, long delayMillis)
{
    if (delayMillis < 0) {
        delayMillis = 0;
    }
    return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
}

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

如上源码，我们可以发现，实际上post、postDelay以及sendMessage调用的就是sendMessageDelayed，而sendMessageDelayed最终调用的则是sendMessageAtTime。

⚠️注意注意，前方高能了，是不是看到了一个感觉很熟悉的对象呀，没错，你没记错，就是它，mQueue，记住了，就是他mQueue，这个mQueue拿到的就是Looper对象的mQueue，然后我们看下enqueueMessaged的源码：


```
 private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
    msg.target = this;
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}
```

从上述源码可以知道，msg.target关联了当前的Handler对象，猜测是为了后面知道是由哪个Handler发出的消息，然后找到对应的Handler进行回调。

这里有一个疑问🤔️点？，为啥要

```
if(mAsynchronous) {
    msg.setAsynchronous(true);
}
```
而不是

```
msg.setAsynchronous(mAsynchronous);
```
是不是Google爸爸代码没写好？哈哈哈，回到主题，我们再来看一下queue.enqueueMessage如下源码：

```

boolean enqueueMessage(Message msg, long when) {
        if (msg.target == null) {
            throw new IllegalArgumentException("Message must have a target.");
        }
        if (msg.isInUse()) {
            throw new IllegalStateException(msg + " This message is already in use.");
        }

        synchronized (this) {
            if (mQuitting) {
                IllegalStateException e = new IllegalStateException(
                        msg.target + " sending message to a Handler on a dead thread");
                Log.w(TAG, e.getMessage(), e);
                msg.recycle();
                return false;
            }

            msg.markInUse();
            msg.when = when;
            Message p = mMessages;
            boolean needWake;
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

            // We can assume mPtr != 0 because mQuitting is false.
            if (needWake) {
                nativeWake(mPtr);
            }
        }
        return true;
    }

```

天啊，怎么这代码撇一眼看起来那么复杂，别怕，慢慢来一一分析，大胆猜想，小心求证。从方法的名称我们可以分析，该函数就是把一个Message进入一个“队列“，为什么”队列“需要加入一个双引号，坏笑ing...，咱们走着瞧 - -

1. msg.target不能为null，为null直接抛出异常。
2. msg.isInUse()用来标识该Message是否被利用过，所以每次需要send一个Message都需要New一个，否则会抛出异常。

接下来再看if跟else的逻辑，

```
msg.when = when;
Message p = mMessages;
if (p == null || when == 0 || when < p.when) {
    // New head, wake up the event queue if blocked.
    msg.next = p;
    mMessages = msg;
    needWake = mBlocked;
} 
```

从if的逻辑，我们可以猜测，mMessages是当前处理的消息的最头部的消息，从if的判断逻辑可以知道，当==p=null，即mMessage为null，或者when=0==，即马上处理该Message，when < p.when，表示msg的处理优先级要比mMessage的要高，所以再if的方法体里面，把msg的next指向了p，并且把msg赋值给mMessage，即重新构建了“队列”的头部，故此我们也可以知道了为什么“队列”需要用引号，实际上的队列不是队列，而是一个链表。

看完了if的逻辑，接下来我们看一下else的逻辑，其实跟if大同小异，就是链表的知识，如下关键源码：


```
else {
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

```
从上述代码可以发现，实际上死循环上面做的事情实际上就是遍历这一条链表，当遇到==p=null或者when<p.when==,则结束循环，是什么意思呢？

首先p=null，则表示已经遍历到了该链表的最后一个了，此时直接跳出循环，后面的


```
msg.next = p; // invariant: p == prev.next
prev.next = msg;  
```
即把msg插入到prev以及p之间，同理when < p.when也是一样的。

**总结：实际上Handler是通过MeesageQueue来构建消息队列的，并且该消息队列并不是真正的队列，而是一个根据时间排序的链表, 分析了Handler怎样构建消息队列，那么接下来我们继续分析MessageQueue的队列是怎样被处理的呢？**


### 2.2 MessageQueue的消息队列是如何被处理的？

由构造函数我们可以知道，Handler持有的mQueue实际上就是mLooper.mQueue对象，通过查看源码，我们发现，原来mQueue实际上就是在Looper.loop()方法进行使用的，还记得我们在ActivityThread当中prepareMainLooper的时候，在ActivityThead中就调用了Looper.loop()方法，即在该方法中处理了消息队列，接下来我们分析Looper.loop()的源码，如下：


```
public static void loop() {
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;

        for (;;) {
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }
                ...
            try {
                msg.target.dispatchMessage(msg);
            } finally {
                ...
            }
            msg.recycleUnchecked();
        }
    }

```
简化了上述代码，可以看到，实际上通过调用queue.next（）方法，该方法是一个阻塞的方法，取出当前需要处理的msg，然后再调用msg.target.dispatchMessage(msg)来进行消息的分发，还记得我们文章前面说过的taget对象的作用吗，哈哈哈，实际上就是调用Handler对象的dispatchMessage方法，msg.target就在前面提到的enqueueMessage方法中被赋值的。

我们再来分析MessageQueue的next方法，源码如下：


```

Message next() {
    ....
    int nextPollTimeoutMillis = 0;
    for (;;) {
        if (nextPollTimeoutMillis != 0) {
            Binder.flushPendingCommands();
        }

        nativePollOnce(ptr, nextPollTimeoutMillis);

        synchronized (this) {
            // Try to retrieve the next message.  Return if found.
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;
            if (msg != null) {
                if (now < msg.when) {
                    // Next message is not ready.  Set a timeout to wake up when it is ready.
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
                    // Got a message.
                    
                    **在此处取出消息**
                    mBlocked = false;
                    if (prevMsg != null) {
                        prevMsg.next = msg.next;
                    } else {
                        mMessages = msg.next;
                    }
                    msg.next = null;
                    if (DEBUG) Log.v(TAG, "Returning message: " + msg);
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
        // so go back and look again for a pending message without waiting.
        nextPollTimeoutMillis = 0;
    }
...
}
```

这里面的代码关键是mMessages的方法，回顾一下，我们在enqueueMessage有看到，实际mMessages实际上就是消息头，并且把mMessages赋值给msg

当取出的msg不为空：
1. 对比当前的时间与传递过来的msg.when, 如果小于的话，则记录差值的时间，然后再通过nativePollOnce阻塞等待，超过nextPollTimeoutMillis则超时
2. 如果需要马上处理，则在此处取出msg，直接return

当取出的msg为空，则nextPollTimeoutMillis = -1，则表示一直阻塞，并不会超市。

在此仅仅在java层进行分析，具体原理可以关注我后续的系列文章哈。

看完了MessageQueue的next方法，接下来我们看一下msg.target.dispatchMessage(msg)的方法，源码如下。

```
  /**
     * Callback interface you can use when instantiating a Handler to avoid
     * having to implement your own subclass of Handler.
     *
     * @param msg A {@link android.os.Message Message} object
     * @return True if no further handling is desired
     */
    public interface Callback {
        public boolean handleMessage(Message msg);
    }
    
    /**
     * Subclasses must implement this to receive messages.
     */
    public void handleMessage(Message msg) {
    }
    
    
 /**
     * Handle system messages here.
     */
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
    
    private static void handleCallback(Message message) {
        message.callback.run();
    }
```

这里面的逻辑也很简单：

1.当msg.callback != null 的时候，直接调用message.callback.run()的方法，即对应我们调用Handler.post or postDelay的方法创建的Runnable对象来实现回调。

2. 当mCallback != null的时候，则创建Handler的时候，实现了Handler的Callback接口，直接回调，对应sendMessage or sendMessageDelay发送消息的方法。

3. 如果mCallback = null，则调用自身的handlerMessage方法，子类需要实现该方法才能对接受到消息然后再进行处理，对应sendMessage or sendMessageDelay发送消息的方法。




## 3. 总结：至此，Java层结合MessageQueue、Looper以及Handler来分析了Handler的消息传递机制。

**1. 子线程通过Looper.prepare()方法来创建对应的线程的Looper对象，主线程则通过Looper.prepare MainLooper()来创建Looper对象。**

**2. 创建完Looper然后再创建Handler对象，在构造函数当中可以通过Looper.myLooper获取到当前线程对应的Looper对象，同样也可以通过传递Looper对象给到Handler的构造函数。**

**3. 调用Looper.loop()方法循环阻塞获取消息。**

**4. 通过MessageQueue的enqueueMessage方法按照时间排序构建消息链表，然后通过**

**5. 通过步骤4的loop死循环，不断的获取消息，获取到消息，则调用对应的Handler的dispatchMessage方法来进行对应的消息处理**







