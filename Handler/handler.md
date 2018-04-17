# 深入理解Android消息机制

在日常的开发中，Android 的消息机制作为系统运行的根本机制之一，显得十分的重要。
### 从 Handler 发送消息开始

查看源码，Handler的post、send方法最终都会走到
```java
public final boolean sendMessageDelayed(Message msg, long delayMillis) {
	if (delayMillis < 0) {
	    delayMillis = 0;
    }
    return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
}

```

sendMessageDelayed 会走到
```java
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
	msg.target = this;
    if (mAsynchronous) {
	    msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}
```

这里可以设置 Message 为异步消息

查看 queue 的 enqueueMessage 方法， 我们剥离出核心代码：

```java
if (p == null || when == 0 || when < p.when) {
	// New head, wake up the event queue if blocked.
	msg.next = p;
    mMessages = msg;
    needWake = mBlocked;
 }
```

如果是新的队列头，直接插入队列

如果队列里面已经有消息了，执行如下逻辑
```java
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
```

插入消息的时候，一般不会唤醒消息队列。如果消息是异步的，并且队列头不是一个异步消息的时候，会唤醒消息队列
```java
if (needWake) {
	nativeWake(mPtr);
}
```

消息队列的具体唤醒过程我们暂时不细看。把关注点移到 Looper 上。looper在执行的时候具体执行了什么逻辑呢？查看 Looper.java 的 looper() 方法

looper 方法中有一个死循环， 在死循环中，会获取下一个 Message
```java
for (;;) {
	Message msg = queue.next(); // might block
}
```

```java
if (msg != null && msg.target == null) {
// Stalled by a barrier.  Find the next asynchronous message in the queue.
do {
	prevMsg = msg;
	msg = msg.next;
} while (msg != null && !msg.isAsynchronous());
```

当存在一个 barrier 消息的时候，会寻找队列中下一个异步任务。而不是按照顺序。
例如3个消息，1，2，3， 2 是异步消息。如果不存在barrier的时候，next的顺序就是 1，2，3
但是如果存在barrier的时候，则是 2，1，3

```java
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
```

这里如果 next 的 Message 不为空，就返回，并且将它移出队列
在 MessageQueue 为空的时候，会顺便去处理一下 add 过的 IdleHandler,  处理一些不重要的消息
```java
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

```

查看 IdleHandler 的源码。
```java/**
     * Callback interface for discovering when a thread is going to block
     * waiting for more messages.
     */
    public static interface IdleHandler {
        /**
         * Called when the message queue has run out of messages and will now
         * wait for more.  Return true to keep your idle handler active, false
         * to have it removed.  This may be called if there are still messages
         * pending in the queue, but they are all scheduled to be dispatched
         * after the current time.
         */
        boolean queueIdle();
    }
```

当 queueIdle() 为 false 的时候，会将它从 mIdleHandlers 中 remove，仔细思考下，我们其实可以利用IdleHandler实现不少功能， 例如
```
Looper.myQueue().addIdleHandler(new MessageQueue.IdleHandler() {
	@Override
	public boolean queueIdle() {
		return false
	}
});
```
我们可以在 queueIdle 中，趁着没有消息要处理，统计一下页面的渲染时间（消息发送完了说明UI已经渲染完了），或者算一下屏幕是否长时间没操作等等。

拿到 Message 对象后，会将 Message 分发到对应的 target 去
```java
msg.target.dispatchMessage(msg);
```
查看源码
```java
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

当 msg 的 callback 不为 null 的时候，即通过 post(Runnable) 发送信息的会执行 handlerCallback(msg) 方法。如果 mCallback 不为 null并且 handleMessage 的结果为 false，则执行 handleMessage 方法。否则会停止分发。
```java
private static void handleCallback(Message message) {
	message.callback.run();
}
```

`
查看 handlerCallback 方法源码， callback 会得到执行。到这里基本的Android消息机制就分析完了，简而言之就是，Handler 不断的将Message发送到一 根据时间进行排序的优先队列里面，而线程中的 Looper 则不停的从MQ里面取出消息，分发到相应的目标Handler执行。
`

# 为什么主线程不卡？
分析完基本的消息机制，既然 Looper 的 looper 方法是一个for(;;;)循环，那么新的问题提出来了。为什么Android会在主线程使用死循环？执行死循环的时候为什么主线程的阻塞没有导致CPU占用的暴增？

继续分析在源码中我们没有分析的部分：

1.  消息队列构造的时候是否调用了jni部分
2.  nativeWake、nativePollOnce这些方法的作用是什么

先查看MQ的构造方法：
```java
MessageQueue(boolean quitAllowed) {
	mQuitAllowed = quitAllowed;
	mPtr = nativeInit();
}
```

会发现消息队列还是和native层有关系，继续查看android/platform/frameworks/base/core/jni/android_os_MessageQueue_nativeInit.cpp中nativeInit的实现：
```cpp
static jlong android_os_MessageQueue_nativeInit(JNIEnv* env, jclass clazz) {
    NativeMessageQueue* nativeMessageQueue = new NativeMessageQueue();
    if (!nativeMessageQueue) {
        jniThrowRuntimeException(env, "Unable to allocate native queue");
        return 0;
    }

    nativeMessageQueue->incStrong(env);
    return reinterpret_cast<jlong>(nativeMessageQueue);
}
```

这里会发现我们初始化了一个 NativeMessageQueue ，查看这个消息队列的构造函数
```cpp
NativeMessageQueue::NativeMessageQueue() :
        mPollEnv(NULL), mPollObj(NULL), mExceptionObj(NULL) {
    mLooper = Looper::getForThread();
    if (mLooper == NULL) {
        mLooper = new Looper(false);
        Looper::setForThread(mLooper);
    }
}
```

这里会发现在mq中初始化了 native 的 Looper 对象，查看android/platform/framework/native/libs/utils/Looper.cpp中 Looper 对象的构造函数

```cpp
// 简化后的代码
Looper::Looper(bool allowNonCallbacks) :
        mAllowNonCallbacks(allowNonCallbacks), mSendingMessage(false),
        mResponseIndex(0), mNextMessageUptime(LLONG_MAX) {

	int wakeFds[2];
	int result = pipe(wakeFds);

	mWakeReadPipeFd = wakeFds[0];
	mWakeWritePipeFd = wakeFds[1];
	
	result = fcntl(mWakeReadPipeFd, F_SETFL, O_NONBLOCK);
	result = fcntl(mWakeWritePipeFd, F_SETFL, O_NONBLOCK);

	mEpollFd = epoll_create(EPOLL_SIZE_HINT);

	struct epoll_event eventItem;
    memset(& eventItem, 0, sizeof(epoll_event)); 
    eventItem.events = EPOLLIN;
    eventItem.data.fd = mWakeReadPipeFd;
    result = epoll_ctl(mEpollFd, EPOLL_CTL_ADD, mWakeReadPipeFd, & eventItem);
}
```

这里我们会发现，在 native 层创建了一个epoll，并且对 epoll 的 event 事件进行了监听。

##### 什么是epoll
在继续分析源码之前，我们先分析一下，什么是epoll

***

epoll是Linux中的一种IO多路复用方式，也叫做event-driver-IO。

Linux的select 多路复用IO通过一个select()调用来监视文件描述符的数组，然后轮询这个数组。如果有IO事件，就进行处理。

select的一个缺点在于单个进程能够监视的文件描述符的数量存在最大限制，select()所维护的存储大量文件描述符的数据结构，随着文件描述符数量的增大，其复制的开销也线性增长。

epoll在select的基础上（实际是在poll的基础上）做了改进，epoll同样只告知那些就绪的文件描述符，而且当我们调用epoll_wait()获得就绪文件描述符时，返回的不是实际的描述符，而是一个代表就绪描述符数量的值，你只需要去epoll指定的一个数组中依次取得相应数量的文件描述符即可。

另一个本质的改进在于epoll采用基于事件的就绪通知方式（设置回调）。在select中，进程只有在调用一定的方法后，内核才对所有监视的文件描述符进行扫描，而epoll事先通过epoll_ctl()来注册一个文件描述符，一旦基于某个文件描述符就绪时，内核会采用类似callback的回调机制，迅速激活这个文件描述符，当进程调用epoll_wait()时便得到通知

关于epoll和select，可以举一个例子来表达意思。select的情况和班长告诉全班同学交作业类似，会挨个去询问作业是否完成，如果没有完成，班长会继续询问。

而epoll的情况则是班长询问的时候只是统计了待交作业的人数，然后告诉同学作业完成的时候告诉把作业放在某处，然后喊一下他。然后班长每次都去这个地方收作业。
***


大致了解了epoll之后，我们继续查看nativePollOnce方法，同理，会调用native Looper的pollOnce方法

```cpp
while (mResponseIndex < mResponses.size()) {
            const Response& response = mResponses.itemAt(mResponseIndex++);
            int ident = response.request.ident;
            if (ident >= 0) {
                int fd = response.request.fd;
                int events = response.events;
                void* data = response.request.data;
                if (outFd != NULL) *outFd = fd;
                if (outEvents != NULL) *outEvents = events;
                if (outData != NULL) *outData = data;
                return ident;
            }
        }
```
在pollOnce中，会先处理没有callback的response(ALOOPER_POLL_CALLBACK = -2)，处理完后会执行pollInner方法

```cpp
// 移除了部分细节处理和日志代码
// 添加了分析源码的日志
int Looper::pollInner(int timeoutMillis) {
	if (timeoutMillis != 0 && mNextMessageUptime != LLONG_MAX) {
	        nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
	        int messageTimeoutMillis = toMillisecondTimeoutDelay(now, mNextMessageUptime);
	        if (messageTimeoutMillis >= 0
	                && (timeoutMillis < 0 || messageTimeoutMillis < timeoutMillis)) {
	            timeoutMillis = messageTimeoutMillis;
	        }
	  }

	// Poll.
    int result = ALOOPER_POLL_WAKE;
    mResponses.clear();
    mResponseIndex = 0;

    struct epoll_event eventItems[EPOLL_MAX_EVENTS];
	// 等待事件发生或者超时
    int eventCount = epoll_wait(mEpollFd, eventItems, EPOLL_MAX_EVENTS, timeoutMillis);

    // Acquire lock.
    mLock.lock();


	// Check for poll error.
	// epoll 事件小于0， 发生错误
    if (eventCount < 0) {
        if (errno == EINTR) {
            goto Done;
        }
        result = ALOOPER_POLL_ERROR;
        goto Done;
    }

	if (eventCount == 0) {
		// epoll事件为0，超时，直接跳转到Done
        result = ALOOPER_POLL_TIMEOUT;
        goto Done;
    }

	//循环遍历，处理所有的事件
	for (int i = 0; i < eventCount; i++) {
        int fd = eventItems[i].data.fd;
        uint32_t epollEvents = eventItems[i].events;
        if (fd == mWakeReadPipeFd) {
            if (epollEvents & EPOLLIN) {
                awoken();  //唤醒，读取管道里面的事件
            } else {
            }
        } else {
            ssize_t requestIndex = mRequests.indexOfKey(fd);
            if (requestIndex >= 0) {
                int events = 0;         
                // 处理request，生成response对象，push到相应的Vector
                pushResponse(events, mRequests.valueAt(requestIndex));
            } else {               
            }
        }
    }

Done: ;

	// Invoke pending message callbacks.
	// 发生超时的逻辑处理
    mNextMessageUptime = LLONG_MAX;
    while (mMessageEnvelopes.size() != 0) {
	    // 处理Native端的Message
        nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
        const MessageEnvelope& messageEnvelope = mMessageEnvelopes.itemAt(0);
        if (messageEnvelope.uptime <= now) {
            // Remove the envelope from the list.
            // We keep a strong reference to the handler until the call to handleMessage
            // finishes.  Then we drop it so that the handler can be deleted *before*
            // we reacquire our lock.
            { // obtain handler
                sp<MessageHandler> handler = messageEnvelope.handler;
                Message message = messageEnvelope.message;
                mMessageEnvelopes.removeAt(0);
                mSendingMessage = true;
                mLock.unlock();
                handler->handleMessage(message);   // 处理消息事件
            } // release handler

            mLock.lock();
            mSendingMessage = false;
            result = ALOOPER_POLL_CALLBACK;   // 设置回调
        } else {
            // The last message left at the head of the queue determines the next wakeup time.
            mNextMessageUptime = messageEnvelope.uptime;
            break;
        }
    }

    // Release lock.
    mLock.unlock();

    // Invoke all response callbacks.
    // 执行回调
    for (size_t i = 0; i < mResponses.size(); i++) {
        Response& response = mResponses.editItemAt(i);
        if (response.request.ident == ALOOPER_POLL_CALLBACK) {
            int fd = response.request.fd;
            int events = response.events;
            void* data = response.request.data;
            int callbackResult = response.request.callback->handleEvent(fd, events, data);
            if (callbackResult == 0) {
                removeFd(fd);  //移除fd
            }
            // Clear the callback reference in the response structure promptly because we
            // will not clear the response vector itself until the next poll.
            response.request.callback.clear();  // 清除reponse引用的回调方法
            result = ALOOPER_POLL_CALLBACK;  // 发生回调
        }
    }
    return result;
}
```

看到这里，我们其实可以看出来整体消息模型由 native 和 Java 2层组成，2层各自有自己的消息系统。 Java层通过调用 pollonce 来达到调用底层epoll 让死循环进入阻塞休眠的状态，以避免浪费CPU， 所以这也解释了为什么Android Looper的死循环为什么不会让主线程CPU占用率飙升。

java层和native层的对应图如下：

![](https://github.com/shaomaicheng/Article/blob/master/imgs/handler.png?raw=true)

**备注**
1. Java 层和 native 层通过 MessageQueue 里面持有一个 native 的MessageQueue 对象进行交互。WeakMessageHandler 继承自MessageHandler，NativeMessageQueue 继承自 MessageQueue
2. Java 层和 native 层实质是各自维护了一套相似的消息系统。C层发出的消息和Java层发出的消息可以没有任何关系。所以 Framework 层只是很巧的利用了底层 epoll 的机制达到阻塞的目的。
3. 通过 pollOnce 的分析，可以发现消息的处理其实是有顺序的，首先是处理native message，然后处理native request，最后才会执行java层，处理java层的message


### 可以在子线程中创建Handler吗?为什么每个线程只会有一个Looper?
在很多时候，我们可以遇到这2个问题。既然看了 Handler 的源码，那么，我们就顺便分析一下这 2 个问题。

查看Handler的构造方法，无参构造方法最后会调用
```java
public Handler(Callback callback, boolean async) {
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

可以看到，这里会直接获取Looper
```java
public static @Nullable Looper myLooper() {
    return sThreadLocal.get();
}
```

这里会把每个 Looper 存到相应的ThreadLocal对象中，如果子线程直接创建了Handler，Looper 就会是一个null，所以会直接跑出一个"Can't create handler inside thread that has not called Looper.prepare()"的RuntimeException

那么我们是何时把Looper放入ThreadLocal对象的呢？可以在Looper.prepare()中找到答案
```java
private static void prepare(boolean quitAllowed) {
	if (sThreadLocal.get() != null) {
		throw new RuntimeException("Only one Looper may be created per thread");
	}
	sThreadLocal.set(new Looper(quitAllowed));
}
```

这也解释了，在每个 Thread 中，只会存在一个 Looper 对象。如果我们想在子线程中正常创建 Handler，就需要提前运行当前线程的 Looper，调用
```java
Looper.prepare()
```
就不会抛出异常了。

### 总结
消息机制作为 Android 的基础，还是非常有深入了解的必要。对于我们遇到Handler发送消息的时候跑出的系统异常的排查也很有意义。

### 特别感谢
本次源码的阅读过程中，遇到了很多不了解的问题例如epoll，这里非常感谢IO哥[（查看IO哥大佬）](https://geminiwen.com)助和指导。让我在某些细节问题上暂时绕过和恍然大悟。
