---
title: Hanlder源码分析
date: 2017-07-23 20:47:30
tags: 源码分析
---

# 开始
随着开源框架越来越多，我们使用 Handler 的次数越来越少，但是这不代表 Hanlder 被"淘汰"了，反而在很多有名的开源库里面，都有 Handler 的身影。比如在很多网络框架中都是使用下面这行代码来实现将回调在UI线程中执行的,伪代码: new Handler(Looper.getMainLooper()).post(回调)。只不过每个框架在实现细节上有点不同，但是实现方式都离不开上面这行代码。

<!-- more -->

# 一般用法

``` java

Handler mHandler = new Handler(){
    @Override
    public void handleMessage(Message msg){
        //更新UI 等操作...
    }
}

new Thread(() -> {
        //网络请求 IO操作 等等...
        Message message = Message.obtain();
        message.obj = 数据;
        mHandler.sendMessage(message);
}).start();

```

需要注意几个地方.

1. Handler 在哪个线程创建的，那么 handleMessage() 就会在哪个线程中执行(一般情况下)。
2. 如果在子线程中(非 UI 线程)中创建 Handelr，那么必须调用 Looper.prepare() 和 Looper.loop() 方法，不然会报错，原因后面分析。
3. 最好使用 Message.obtain() 方法来创建 Message，在obtain() 方法中有对 Message 的复用机制，可以减少性能消耗。

## Handler

首先看看 Handler 的构造函数，代码如下:

``` java

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

可以看到，如果 mLooper 等于 null 的情况，会抛出一个异常。那么什么情况下 Looper.myLooper() == null 呢？我们看看 myLooper 的实现:

``` java

static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();

public static @Nullable Looper myLooper() {
    return sThreadLocal.get();
}

```

ThreadLocal 的作用就是为当前线程存储一个对象，每个线程中互补干扰。这就很好解释了为什么我们在子线程中 new Handler 必须要要调用 Looper.prepare() 方法。

在看看 Handler 的 sendMessage 方法，看看内部做了些什么:

``` java

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

private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
    msg.target = this;
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}

```

sendMesage() 方法最终会调用 sendMessageAtTime() 方法，这个方法没什么好说的，只是调用了 enqueueMessage()，在 Handler 的 enqueueMessage() 方法中对 target 指向 this，然后调用 MessageQueue 的 enqueueMessage() 方法，下面我们分析分析这个方法。

## MesssageQueue

Handler的工作离不开 MessageQueue，因为 Handler 本身只是处理消息，消息并不是存在在 Handler 里面，很显然是存放在MessageQueue 中(废话~)，从类名推断是个消息队列，看看其内部的实现。

``` java

boolean enqueueMessage(Message msg, long when) {
    
    //省略验证判断代码

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

从上面的代码中可以看出，MessageQueue 内部是用一个单链表实现的，因为 Message 需要频繁的添加和删除，所以用单链表性能更高。
enqueueMessage() 方法的作用就是将msg 添加到单链表中，mMessage 指向的是单链表的表头，每次添加消息都是添加到表头。

既然有添加消息的方法，那么肯定还有取出消息的方法。也就是 MessageQueue 中的 next() 方法。代码如下:

``` java

Message next() {
    // Return here if the message loop has already quit and been disposed.
    // This can happen if the application tries to restart a looper after quit
    // which is not supported.
    final long ptr = mPtr;
    if (ptr == 0) {
        return null;
    }

    int pendingIdleHandlerCount = -1; // -1 only during first iteration
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
                    // Got a message.
                    mBlocked = false;
                    if (prevMsg != null) {
                        prevMsg.next = msg.next;
                    } else {
                        //将 mMessage 指向 mMessage.next
                        mMessages = msg.next;
                    }
                    //next 设置为 null
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

            // If first time idle, then get the number of idlers to run.
            // Idle handles only run if the queue is empty or if the first message
            // in the queue (possibly a barrier) is due to be handled in the future.
            if (pendingIdleHandlerCount < 0
                    && (mMessages == null || now < mMessages.when)) {
                pendingIdleHandlerCount = mIdleHandlers.size();
            }
            if (pendingIdleHandlerCount <= 0) {
                // No idle handlers to run.  Loop and wait some more.
                mBlocked = true;
                continue;
            }

            if (mPendingIdleHandlers == null) {
                mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
            }
            mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
        }

        // Run the idle handlers.
        // We only ever reach this code block during the first iteration.
        for (int i = 0; i < pendingIdleHandlerCount; i++) {
            final IdleHandler idler = mPendingIdleHandlers[i];
            mPendingIdleHandlers[i] = null; // release the reference to the handler

            boolean keep = false;
            try {
                keep = idler.queueIdle();
            } catch (Throwable t) {
                Log.wtf(TAG, "IdleHandler threw exception", t);
            }

            if (!keep) {
                synchronized (this) {
                    mIdleHandlers.remove(idler);
                }
            }
        }

        // Reset the idle handler count to 0 so we do not run them again.
        pendingIdleHandlerCount = 0;

        // While calling an idle handler, a new message could have been delivered
        // so go back and look again for a pending message without waiting.
        nextPollTimeoutMillis = 0;
    }
}  

```

next() 方法的做用就是取一个  Message 并返回，其实就是将 mMessage 指向的那个 Message 从链表中移除。next() 方法的下半部分代码是关于 mPendingIdleHandlers，看声明的地方，其实是一个 ArrayList。这个集合中的 IdleHandler 对象，主要是在UI线程空闲的时候进行执行，比如 leakcanary 框架中就使用这个特性，在空闲的时候进行 GC 从而检查内存泄漏。因为 GC 操作是会影响应用性能的，频繁的 GC 的话 leakcanary 这个框架就没人用了。

具体用法如下:

``` java

Looper.myQueue().addIdleHandler(new MessageQueue.IdleHandler() {
    @Override
    public boolean queueIdle() {
        //do something ...
        return false;
    }
});

```

## Looper

我们知道 enqueueMessage() 方法是在调用 Handler.sendMessage() 方法的时候调用的，那 next() 方法是在哪调用的呢？我们来看看。

``` java

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
        final Printer logging = me.mLogging;
        if (logging != null) {
            logging.println(">>>>> Dispatching to " + msg.target + " " +
                    msg.callback + ": " + msg.what);
        }

        final long traceTag = me.mTraceTag;
        if (traceTag != 0 && Trace.isTagEnabled(traceTag)) {
            Trace.traceBegin(traceTag, msg.target.getTraceName(msg));
        }
        try {
            msg.target.dispatchMessage(msg);
        } finally {
            if (traceTag != 0) {
                Trace.traceEnd(traceTag);
            }
        }

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

        msg.recycleUnchecked();
    }
}

```

在 loop() 方法中调用 mQueue.next() 取出 Message 对象，然后调用其 message.target.dispatchMessage(message) 方法。其内部会调用 Handler 的 handleMessage 方法，这就形成了一个调用链。首先由 Handelr 调用 sendMessage()，在这个方法内部会指定 message.target 为发消息的 Handelr.然后调用 MessageQueue 的 enqueueMessage() 方法将 message 添加到链表中，最后又 Looper 的 loop 方法取出 message，并调用其 target 的 handleMessage 方法，也就回调了我们重写的 handlerMessage() 方法。

自此整个 Handler 的过程就分析完毕了，最后在来看看 Message 中的 obtain() 方法。这个方法的作用是用一个优化机制来防止 我们使用 new Message 带来的性能消耗，我们来看看内部是这么操作的:

``` java

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

void recycleUnchecked() {
    // Mark the message as in use while it remains in the recycled object pool.
    // Clear out all other details.
    flags = FLAG_IN_USE;
    what = 0;
    arg1 = 0;
    arg2 = 0;
    obj = null;
    replyTo = null;
    sendingUid = -1;
    when = 0;
    target = null;
    callback = null;
    data = null;

    synchronized (sPoolSync) {
        if (sPoolSize < MAX_POOL_SIZE) {
            next = sPool;
            sPool = this;
            sPoolSize++;
        }
    }
}

```

在 obtain() 方法中，其实就是根据返回 sPool 这个静态的 Message，然后将 sPool 指向 sPool 的 next，这样就可以实现一个链式的复用，在 recycleUnchecked() 方法中，会先清空 Message 的属性，然后将 next 指向 sPool，sPool 指向 this，这样就实现复用 Message 的添加。

# 总结
Handler 这个类设计很巧妙，暴露给开发者的都是简单的几个类和方法，将最复杂的实现逻辑很好的隐藏在内部，这样我们就能使用 Handler 很方便的在子线程中更新UI，或是实现 无限循环消息等功能...,而至于 Looper 和 MessageQueue 这两个类，我们只需要其大致的实现就够了。
