#about-handler
> Handler의 관한 것을 정리

> present time : 2018-04-01-SUN  
> presentation file : https://docs.google.com/presentation/d/13F7J1ZK8wa_7_VwpQTovt3sy1aRPlNZZe6nNyaCsoSE/edit?usp=sharing  
> last updated : 2018-04-05-03:21-seohyun99

----------------

## 0. 공식 문서
* [Handler](https://developer.android.com/reference/android/os/Handler.html)
* [HandlerThread](https://developer.android.com/reference/android/os/HandlerThread.html)
* [Message](https://developer.android.com/reference/android/os/Message.html)
* [MessageQueue](https://developer.android.com/reference/android/os/MessageQueue.html)
* [Looper](https://developer.android.com/reference/android/os/Looper.html)

----------------

## 1. Handler
> **There are two main uses for a Handler:**  
>(1) to schedule messages and runnables to be executed as some point in the future; and     
>(2) to enqueue an action to be performed on a different thread than your own.

### 1-1. Sub Thread에서 UI 작업 하기
#### Handler를 사용하지 않고 Sub Thread에서 접근할 경우 발생하는 에러
```
java.lang.RuntimeException: Can't create handler inside thread that has not called Looper.prepare()
```

#### Sub Thread에서 UI 작업 하기
Handler.obtainMessage()를 통해 Message를 리턴 받고 sendMessage()를 통해 MessageQueue에 작업을 넣어준다.  
MessageQueue에 들어간 Message는 Handler의 Looper를 통해 읽고 handlerMessage()를 통해 UI작업을 할 수 있다.


### 1-2. Handler Constructor
우리가 자주 사용하는 Handler()의 Constructor는 Handler(Callback, boolean)으로 연결된다.  
```java
public Handler() {
    this(null, false);
}
```

```java
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

### 1-3. method

**post**(Runnable r), **postDelayed**(Runnable r, long delayMillis), **sendMessage**(Message msg), **sendEmptyMessage**(int what)의 내부는 **sendMessageDelayed**(Message msg, long delayMillis)와 **sendMessageAtTime**(Message msg, long uptimeMillis)를 거쳐 전부 **enqueueMessag**(MessageQueue queue, Message msg, long uptimeMillis)로 연결되어 있는 것을 볼 수 있다.

- **post(Runnable r)** →   
sendMessageDelayed(getPostMessage(r), 0) →   
sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis) →  
**enqueueMessage**(queue, msg, uptimeMillis)

- **postDelayed(Runnable r, long delayMillis)** →   
sendMessageDelayed(getPostMessage(r), delayMillis) →  
sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis) →  
**enqueueMessage**(queue, msg, uptimeMillis)

- **sendMessage(Message msg)** →  
sendMessageDelayed(msg, 0) →  
sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis) →  
**enqueueMessage**(queue, msg, uptimeMillis)

- **sendEmptyMessage(int what)** →  
sendMessageDelayed(msg, 0) →  
sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis) →  
**enqueueMessage**(queue, msg, uptimeMillis)

```java
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
    msg.target = this;
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}
```


----------------
## 2. HandlerThread
> Handy class for starting a new thread that has a looper.  
> The looper can then be used to create handler classes. Note that start() must still be called.

HandlerThread는 MessageQueue를 소유한 Thread의 일종이다.
***(Thread의 일종으로 start()를 호출해야 한다.)***  
MessageQueue를 소유하고 있기 때문에 일을 순차적으로 처리한다.  
따라서 MessageQueue의 통제가 필요한 백그라운드 작업을 처리할 때 용이하다는 장점을 갖고 있다.

아래는 HandlerThread의 **run()**이다.  
**run()** 내부에서 Looper.prepare()를 호출하는 것을 볼 수 있다.

```java
@Override
public void run() {
    mTid = Process.myTid();
    Looper.prepare();
    synchronized (this) {
        mLooper = Looper.myLooper();
        notifyAll();
    }
    Process.setThreadPriority(mPriority);
    onLooperPrepared();
    Looper.loop();
    mTid = -1;
}
```

아래는 HandlerThread의 아주 기본적인 사용 예제이다.   
HandlerThread 내부에 있는 Looper를 가지고 와 Message를 처리할 수 있다.

```java
HandlerThread example = new HandlerThread();
example.start();

Handler handler = new Handler(example.getLooper()) {
	@Override
	public void handleMessage(Message msg) {
	switch (msg.what) {
	   case 1:
		   //작업
	   case 2:
		   //작업
    }
  }
}
```


----------------


## 3. Message
> Defines a message containing a description and arbitrary data object that can be sent to a Handler. This object contains two extra int fields and an extra object field that allow you to not do allocations in many cases.

#### Message 생성
Message의 생성자는 public이지만, Message msg = new Message() 대신,     
Message.obtain () 또는 Handler.obtainMessage () 메소드 중 하나를 호출하여 재활용 된 객체 풀에서 가져온다.

### 3.1 important fields & method
- **obtain()**
> Return a new Message instance from the global pool. Allows us to avoid allocating new objects in many cases.

- **what** (int)
> User-defined message code so that the recipient can identify what this message is about.

----------------

## 4. MessageQueue
>  Low-level class holding the list of messages to be dispatched by a Looper. Messages are not added directly to a MessageQueue, but rather through Handler objects associated with the Looper.  

다른 스레드나 혹은 자기 자신으로부터 전달받은 Message를 기본적으로 선입선출 형식으로 보관하는 Queue이다.

#### MessageQueue의 종료
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

----------------

## 5. Looper
> Class used to run a message loop for a thread. Threads by default do not have a message loop associated with them; to create one, call **prepare()** in the thread that is to run the loop, and then loop() to have it process messages until the loop is stopped.

MessageQueue의 Message를 읽어온다.

### 5.1 important method

#### 5.1.1 prepare()
```java
private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper(quitAllowed));
}
```
#### 5.1.2 loop()
```java
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
        final long slowDispatchThresholdMs = me.mSlowDispatchThresholdMs;
        final long traceTag = me.mTraceTag;
        if (traceTag != 0 && Trace.isTagEnabled(traceTag)) {
            Trace.traceBegin(traceTag, msg.target.getTraceName(msg));
        }
        final long start = (slowDispatchThresholdMs == 0) ? 0 : SystemClock.uptimeMillis();
        final long end;
        try {
            msg.target.dispatchMessage(msg);
            end = (slowDispatchThresholdMs == 0) ? 0 : SystemClock.uptimeMillis();
        } finally {
            if (traceTag != 0) {
                Trace.traceEnd(traceTag);
            }
        }
        if (slowDispatchThresholdMs > 0) {
            final long time = end - start;
            if (time > slowDispatchThresholdMs) {
                Slog.w(TAG, "Dispatch took " + time + "ms on "
                        + Thread.currentThread().getName() + ", h=" +
                        msg.target + " cb=" + msg.callback + " msg=" + msg.what);
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
for(;;)로 이루어져 무한하게 루프를 돌며 queue의 Message를 읽기 때문에 종료하기 위해서는 queue.next()가 null인 경우 이거나 사용자가 직접 Looper.quit()을 실행시켜야 한다.

#### 5.1.2 quit()와 quitSafely()
```java
public void quit() {
    mQueue.quit(false);
}
```
quit()를 사용하면 loop()가 종료하지만, looper가 끝나기 전에 전달되지 않은 일부 Message들이 unsafe할 수 있으므로 모든 작업이 순서대로 완료되게 하려면 아래 메소드를 사용해야 한다.

```java
public void quitSafely() {
    mQueue.quit(true);
}
```

quit()는 **MessageQueue의 quit(boolean unsafe)**로 넘어가게 된다. 파라미터로 넘어오는 boolean 값의 따라 내부가 분기된다.  
*(MessageQueue의 quit는 3. MessageQueue 부분에 정리되어 있다.)*




### 5.2 compare method

#### 5.2.1 return Looper
```java
public static @Nullable Looper myLooper() {
    return sThreadLocal.get();
}
```
```java
public static Looper getMainLooper() {
    synchronized (Looper.class) {
        return sMainLooper;
    }
}
```
- myLooper() : Return the Looper object associated with the current thread.  Returns null if the calling thread is not associated with a Looper.
- getMainLooper() : Returns the application's main looper, which lives in the main thread of the application.

getMainLooper()는 Application의 Main Thread의 Looper를 리턴하고, myLooper()는 현재 스레드의 Looper를 반환해 null이 반환 될 수 있다.

#### 5.2.2 return Queue
```java
public static @NonNull MessageQueue myQueue() {
    return myLooper().mQueue;
}
```
```java
public @NonNull MessageQueue getQueue() {
    return mQueue;
}
```
- myQueue() : Return the MessageQueue object associated with the current thread. This must be called from a thread running a Looper, or a NullPointerException will be thrown.
- getQueue() : Gets this looper's message queue.

myQueue()는 현재 스레드와 관계있는 Messeage Queue를 리턴한다. 하지만, Looper를 실행중인 스레드에서 호출하지 않으면 NullPointerException이 발생한다. getQueue()는 public method로 Looper의 Message queue를 반환한다.
