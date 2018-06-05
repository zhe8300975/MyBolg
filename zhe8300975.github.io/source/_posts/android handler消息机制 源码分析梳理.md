---
title: handler机制 梳理
---
# handler机制 梳理
>一直自认为对handler机制算是已经了解透彻了，一次偶然的机会发现 工作一段时间后我他丫的居然块忘干净了 今天重新梳理下，希望能更加深入

先总结几个核心的类，避免以后看这个记录会摸不到头绪
***ActivityThread***

***Loop***

***ThreadLocal***

***MessageQueue***

***Message***

***Handler***

好目录已经有了 ，现在要做的是定义一下这几个类干嘛用的了

#### ActivityThread是什么？
>**ActivityThread** 有人说的是主线程 或者 UI线程 ，个人认为这么说是不对的,我们来看下ActivityThread的定义
```
 public final class ActivityThread {
```

>ActivityThread不过是一个简单类和线程的Thread貌似好像没啥关系。
>
>那为什么有人会说ActivityThread是主线程呢？原因很简单就是在app启动的时候Zygote会被fork一个新的进程也就是app所运行的进程。进程的有了后会加载ActivitThread这个类，那么根据java基础，我们之后他会运行static main函数  mian里面new了个ActivityThread 所以ActivityThread开始运行在了主线程里（这里也有可能是我理解的不到位，如有好的理解方式还请赐教）

下面是main的5.0的源码，要是心烦可以直接跳过代码块 因为里面就4句话对我们有用的

~~~
5184 public static void main(String[] args) {
5185        SamplingProfilerIntegration.start();
5186
5187        // CloseGuard defaults to true and can be quite spammy.  We
5188        // disable it here, but selectively enable it later (via
5189        // StrictMode) on debug builds, but using DropBox, not logs.
5190        CloseGuard.setEnabled(false);
5191
5192        Environment.initForCurrentUser();
5193
5194        // Set the reporter for event logging in libcore
5195        EventLogger.setReporter(new EventLoggingReporter());
5196
5197        Security.addProvider(new AndroidKeyStoreProvider());
5198
5199        // Make sure TrustedCertificateStore looks in the right place for CA certificates
5200        final File configDir = Environment.getUserConfigDirectory(UserHandle.myUserId());
5201        TrustedCertificateStore.setDefaultUserDirectory(configDir);
5202
5203        Process.setArgV0("<pre-initialized>");
5204
5205        Looper.prepareMainLooper();
5206
5207        ActivityThread thread = new ActivityThread();
5208        thread.attach(false);
5209
5210        if (sMainThreadHandler == null) {
5211            sMainThreadHandler = thread.getHandler();
5212        }
5213
5214        AsyncTask.init();
5215
5216        if (false) {
5217            Looper.myLooper().setMessageLogging(new
5218                    LogPrinter(Log.DEBUG, "ActivityThread"));
5219        }
5220
5221        Looper.loop();
5222
5223        throw new RuntimeException("Main thread loop unexpectedly exited");
5224    }
5225}
~~~

>5207，5208行中创建了一个ActivityThread 并将当前的数据初始化到了ActivityThread中
>
**然后重点关注下**

>5205的``` Looper.prepareMainLooper();``` 里面创建了一个Looper实例，并保存在ThreadLocal中 （ThreadLocal之后说干什么的，先有个印象）
>
>5221的```Looper.loop();``` 开启Looper的循环进行处理

开启流程到这就结束了，而主线程从此就进入了一个死循环;PS：之后的生命周期管理呀，ui处理呀都是在loop里面的循环进行处理的

#### Loop是什么？
>>**Loop** 核心是一个死循环，用于循环**处理**MessageQueue的信息
这里注意下我用的词 **死循环**和**处理**  之后就会明白我为什么用这两个词了

这里先说一个概念 ***每个线程中默认是没有Looper的，如果想使用就必须需要自己为线程初始化一个Looper，而主线程在创建的时候就为初始化了一个Looper，这也就是主线程为什么可以时候用Handler的原因***

在上面的一节里我说了需要关注下5027和5028

* 先说下5027这里的```Looper.prepareMainLooper();``` 

~~~
87     public static void prepareMainLooper() {
88         prepare(false);
89         synchronized (Looper.class) {
90             if (sMainLooper != null) {
91                 throw new IllegalStateException("The main Looper has already been prepared.");
92             }
93             sMainLooper = myLooper();
94         }
95     }
~~~
初始化在第一句话 ```prepare(false)```那里面

~~~
74     private static void prepare(boolean quitAllowed) {
75         if (sThreadLocal.get() != null) {
76             throw new RuntimeException("Only one Looper may be created per thread");
77         }
78         sThreadLocal.set(new Looper(quitAllowed));
79     }

~~~
prepare函数里面也非常明确地告诉我们new了一个不可退出的Looper放在了ThreadLocal中

~~~
186    private Looper(boolean quitAllowed) {
187        mQueue = new MessageQueue(quitAllowed);
188        mThread = Thread.currentThread();
189    }
~~~ 

Looper创建的过程中为自己创建了一个MessageQueue，还保存了当前的线程，至此Looper创建完成。

* 然后及时5221的```Looper.loop();``` 开启Looper的循环进行处理

又一波长段代码了 不过还是一样 不想看的可以不看 没几句话有用的（其实除了注释也没几行）

~~~
109    public static void loop() {
110        final Looper me = myLooper();
111        if (me == null) {
112            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
113        }
114        final MessageQueue queue = me.mQueue;
115
116        // Make sure the identity of this thread is that of the local process,
117        // and keep track of what that identity token actually is.
118        Binder.clearCallingIdentity();
119        final long ident = Binder.clearCallingIdentity();
120
121        for (;;) {
122            Message msg = queue.next(); // might block
123            if (msg == null) {
124                // No message indicates that the message queue is quitting.
125                return;
126            }
127
128            // This must be in a local variable, in case a UI event sets the logger
129            Printer logging = me.mLogging;
130            if (logging != null) {
131                logging.println(">>>>> Dispatching to " + msg.target + " " +
132                        msg.callback + ": " + msg.what);
133            }
134
135            msg.target.dispatchMessage(msg);
136
137            if (logging != null) {
138                logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
139            }
140
141            // Make sure that during the course of dispatching the
142            // identity of the thread wasn't corrupted.
143            final long newIdent = Binder.clearCallingIdentity();
144            if (ident != newIdent) {
145                Log.wtf(TAG, "Thread identity changed from 0x"
146                        + Long.toHexString(ident) + " to 0x"
147                        + Long.toHexString(newIdent) + " while dispatching to "
148                        + msg.target.getClass().getName() + " "
149                        + msg.callback + " what=" + msg.what);
150            }
151
152            msg.recycleUnchecked();
153        }
154    }
~~~

关注点 
>110的```final Looper me = myLooper();``` 获取当前线程的Looper
>121的```for (;;) ```  死循环来处理message中的信息
>
>122的```Message msg = queue.next();``` 获取messageQueue的信息（这里是怎么获取的之后MessageQueue我们再说）
>
>135的```msg.target.dispatchMessage(msg);``` 处理信息（怎么处理？肯定是handler里面处理了， tag里面保存的是handler 而处理什么 之后handler会详细的说到）

~~~
160    public static Looper myLooper() {
161        return sThreadLocal.get();
162    }
~~~
myLooper是在ThreadLocal中获取到的 



#### ThreadLocal是什么？
>>**ThreadLocal**  每次一看名字，第一反应就是xx线程。这里不紧自骂一句线程”你妹呀，什么就线程呀，他明明是一个负责存储的类“  存储类 不同线程不同的副本互不影响，作用域整个线程内

* 先说set

~~~
179    public void set(T value) {
180        Thread t = Thread.currentThread();
181        ThreadLocalMap map = getMap(t);
182        if (map != null)
183            map.set(this, value);
184        else
185            createMap(t, value);
186    }
~~~

看到这可能之前的我就不看了，用的map存的。但事实不是这样的。。。```static class ThreadLocalMap {```。。。尼玛Map是自己定义的和Map类没有直系关系还是个内部类，你他妈玩我，这要是去除面试问我细节我按Map来回答不他妈二逼了？事实上确实被某位面试官下过套。

~~~
416 private void set(ThreadLocal key, Object value) {
417
418            // We don't use a fast path as with get() because it is at
419            // least as common to use set() to create new entries as
420            // it is to replace existing ones, in which case, a fast
421            // path would fail more often than not.
422
423            Entry[] tab = table;
424            int len = tab.length;
425            int i = key.threadLocalHashCode & (len-1);
426
427            for (Entry e = tab[i];
428                 e != null;
429                 e = tab[i = nextIndex(i, len)]) {
430                ThreadLocal k = e.get();
431
432                if (k == key) {
433                    e.value = value;
434                    return;
435                }
436
437                if (k == null) {
438                    replaceStaleEntry(key, value, i);
439                    return;
440                }
441            }
442
443            tab[i] = new Entry(key, value);
444            int sz = ++size;
445            if (!cleanSomeSlots(i, sz) && sz >= threshold)
446                rehash();
447        }
~~~

>>看了代码我们知道了 这里面用的是table数组存的！！！
看货hashMap源码的应该知道425的```key.threadLocalHashCode & (len-1);```是干什么的 是取余数的  利用线程的hashCode进行计算
这块代码块的整体作用是利用余数作为下表，当发生冲突时，将下标自增向后移的方式解决冲突。在最后进行判断，然后来确定是否rehash进行扩容。原理上与hashMap相似 但是hashMap是以数组加链表来解决冲突的。解决冲突的方式不相同。这里就不在啊多做赘述了。

***ps：ThreadLocal 本质就是在线程调用的时候获取到以当前进程所对应的变量 不同的变量获取的不是一个，从而实现了线程变量相互隔离，而线程中的变量作用域就成了整个线程***










#### MessageQueue是什么？
>>** MessageQueue** 管理消息的消息队列 从上面Looper的代码可以看出 每个Looper中持有一个MessageQueue对象

>* 一个消息队列最重要的方法肯定是插入和取出了
>
>* *   enqueueMessage()方法--------->插入
>* *   next()方法------------------->取出并移除

***那首先要看的肯定是怎么放入MessageQueue里面了， 为什么说放入是这个方法 后面看到handler就能知道了***

~~~
boolean enqueueMessage(Message msg, long when) {
316        if (msg.target == null) {
317            throw new IllegalArgumentException("Message must have a target.");
318        }
319        if (msg.isInUse()) {
320            throw new IllegalStateException(msg + " This message is already in use.");
321        }
322
323        synchronized (this) {
324            if (mQuitting) {
325                IllegalStateException e = new IllegalStateException(
326                        msg.target + " sending message to a Handler on a dead thread");
327                Log.w("MessageQueue", e.getMessage(), e);
328                msg.recycle();
329                return false;
330            }
331
332            msg.markInUse();
333            msg.when = when;
334            Message p = mMessages;
335            boolean needWake;
336            if (p == null || when == 0 || when < p.when) {
337                // New head, wake up the event queue if blocked.
338                msg.next = p;
339                mMessages = msg;
340                needWake = mBlocked;
341            } else {
342                // Inserted within the middle of the queue.  Usually we don't have to wake
343                // up the event queue unless there is a barrier at the head of the queue
344                // and the message is the earliest asynchronous message in the queue.
345                needWake = mBlocked && p.target == null && msg.isAsynchronous();
346                Message prev;
347                for (;;) {
348                    prev = p;
349                    p = p.next;
350                    if (p == null || when < p.when) {
351                        break;
352                    }
353                    if (needWake && p.isAsynchronous()) {
354                        needWake = false;
355                    }
356                }
357                msg.next = p; // invariant: p == prev.next
358                prev.next = msg;
359            }
360
361            // We can assume mPtr != 0 because mQuitting is false.
362            if (needWake) {
363                nativeWake(mPtr);
364            }
365        }
366        return true;
367    }
~~~
前面的一部分是判断一些状态 核心在于336行开始将message放入列表中，这个位置的放入和的for(;;)代码块，这个位置可不是死循环哦。因为内部是一个单链表实现的，单链表还不是循环链表肯定有结束的时候，所以这个for不会是死循环。
而这个的目的就是将message放入这个自己的链表的尾部实现message的插入



***然后就是取出并移除了，这个方法是不是看的很眼熟呢？***
>原因就是ActivityThread的Main函数里的Looper.loop()方法里面调用过
>
>* 下面又是一大段源码 这里面涉及两大部分一个是Native层的，一个是Java层的。这里呢我不过多的对Native层的做过多的赘述了 咱么只看Java层的的.

~~~
127    Message next() {
128        // Return here if the message loop has already quit and been disposed.
129        // This can happen if the application tries to restart a looper after quit
130        // which is not supported.
131        final long ptr = mPtr;
132        if (ptr == 0) {
133            return null;
134        }
135
136        int pendingIdleHandlerCount = -1; // -1 only during first iteration
137        int nextPollTimeoutMillis = 0;
138        for (;;) {
139            if (nextPollTimeoutMillis != 0) {
140                Binder.flushPendingCommands();
141            }
142
143            nativePollOnce(ptr, nextPollTimeoutMillis);
144
145            synchronized (this) {
146                // Try to retrieve the next message.  Return if found.
147                final long now = SystemClock.uptimeMillis();
148                Message prevMsg = null;
149                Message msg = mMessages;
150                if (msg != null && msg.target == null) {
151                    // Stalled by a barrier.  Find the next asynchronous message in the queue.
152                    do {
153                        prevMsg = msg;
154                        msg = msg.next;
155                    } while (msg != null && !msg.isAsynchronous());
156                }
157                if (msg != null) {
158                    if (now < msg.when) {
159                        // Next message is not ready.  Set a timeout to wake up when it is ready.
160                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
161                    } else {
162                        // Got a message.
163                        mBlocked = false;
164                        if (prevMsg != null) {
165                            prevMsg.next = msg.next;
166                        } else {
167                            mMessages = msg.next;
168                        }
169                        msg.next = null;
170                        if (false) Log.v("MessageQueue", "Returning message: " + msg);
171                        return msg;
172                    }
173                } else {
174                    // No more messages.
175                    nextPollTimeoutMillis = -1;
176                }
177
178                // Process the quit message now that all pending messages have been handled.
179                if (mQuitting) {
180                    dispose();
181                    return null;
182                }
183
184                // If first time idle, then get the number of idlers to run.
185                // Idle handles only run if the queue is empty or if the first message
186                // in the queue (possibly a barrier) is due to be handled in the future.
187                if (pendingIdleHandlerCount < 0
188                        && (mMessages == null || now < mMessages.when)) {
189                    pendingIdleHandlerCount = mIdleHandlers.size();
190                }
191                if (pendingIdleHandlerCount <= 0) {
192                    // No idle handlers to run.  Loop and wait some more.
193                    mBlocked = true;
194                    continue;
195                }
196
197                if (mPendingIdleHandlers == null) {
198                    mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
199                }
200                mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
201            }
202
203            // Run the idle handlers.
204            // We only ever reach this code block during the first iteration.
205            for (int i = 0; i < pendingIdleHandlerCount; i++) {
206                final IdleHandler idler = mPendingIdleHandlers[i];
207                mPendingIdleHandlers[i] = null; // release the reference to the handler
208
209                boolean keep = false;
210                try {
211                    keep = idler.queueIdle();
212                } catch (Throwable t) {
213                    Log.wtf("MessageQueue", "IdleHandler threw exception", t);
214                }
215
216                if (!keep) {
217                    synchronized (this) {
218                        mIdleHandlers.remove(idler);
219                    }
220                }
221            }
222
223            // Reset the idle handler count to 0 so we do not run them again.
224            pendingIdleHandlerCount = 0;
225
226            // While calling an idle handler, a new message could have been delivered
227            // so go back and look again for a pending message without waiting.
228            nextPollTimeoutMillis = 0;
229        }
230    }
~~~


>***又到了代码分析的时候了***
>>* 在143行```nativePollOnce(ptr, nextPollTimeoutMillis);``` 这个是一个native层的处理 多的不说了 ***先总结一句："native层也有一套类似的消息机制，每次处理的时候先处理native的，然后是java层的，java层的里面也有区分就是先取异步的后取同步的 最后处理闲时handler”***  
>>这里给个[地址](http://www.jb51.net/article/93199.htm)看看，里面有提到

>>* 150-156行 看151的注释 提到 被一个barrier开启，获取第一个一异步message。 这里是找出异步Message 然后并处理message  
>>这里Message居然还分同步和异步的，同步和异步的区别是啥？  这个在之后Message里在说。  
>>细心的朋友可能会发现150行代码判断条件是```msg.target==null``` 之前可是提过一嘴msg.target里面应该存的是handler的呀 null是什么鬼 这里要引入一个Barrier（障碍物）的概念  
>>***Barrier是一种特殊的Message，他的target是null。（只有Barrier的target可以为null，如果我们自己试图设置Message的target为null的话会报异常）作用呢?用于区别拦截队伍中的同步信息，放行异步消息***  
这里也给个[地址](https://hjdzone.gitbooks.io/thinkandroid/content/ThreadMessage/Chapter_1_6.html)进行拓展，有介绍虽然说的不是很细，可以作为入门参考。

>>* 157-176这块的功能是 进行当前时间和任务时间的比较，看任务时间是不是已经到了，如果到了那么就返回当前任务，没有就是继续往下走 死循环去

>>* 178-182 看注释就能知道要退出了 mQuitting的值是在quit()中设置的，当然设置之前要判断是否mQuitAllowed==true了 就是最开始创建时候的设置的

>>* 之后运行的就是闲时消息的获取了。 这个很好理解，在MessageQueue也有一些函数是专门为了设置闲时消息的了 这里就不是说了
>>


***PS：综上所属的就是整个messageQueue的获取流程，突然发现这玩意越写越多，还是点到未知的，要不 一个handler就没头了，其他的还搞个球球。之后有时间在完善吧***





####Message是什么？
>>**Message** 消息  就是这种类型作为存储类型的媒介，存入MessageQueue中 实现获取处理用的

Message的构造方法上有这么一句注释 ```/** Constructor (but the preferred way to get a Message is to call {@link #obtain() Message.obtain()}).*/```
Message不建议使用new来创建一个消息，建议使用obtain来创建，简而言之obtain方法里面会有一些初始化的方法，找到里面对我们有用的,也是几乎每个obtion重载方法里面都有的```m.target=h```
>* m代表message  
>* h代表Handler  
这也证明了之前所说的msg.target里面存的是handler这个东东，也证明了在Looper.loop()方法里面的msg.target.dispatchMessage(msg)会将msg分发到handler的dispatchMessage进行分发处理（怎么处理的话之后handler里面会说到）





~~~
118  /**
119     * Return a new Message instance from the global pool. Allows us to
120     * avoid allocating new objects in many cases.
121     */
122 public static Message obtain() {
123        synchronized (sPoolSync) {
124            if (sPool != null) {
125                Message m = sPool;
126                sPool = m.next;
127                m.next = null;
128                m.flags = 0; // clear in-use flag
129                sPoolSize--;
130                return m;
131            }
132        }
133        return new Message();
134    }
~~~
  

在这会发现 我查查Message处理MessageQueue 居然还有个一个sPool 全局的信息池来维护的。所以老师是敲黑板了 这个信息池是做什么用的呢，功能就是复用。从上面的代码块可以看出来，sPool是一个链表结构的。***这个链表里存什么呢，存的就是可复用的Message***  为什么这么说看完下面的你就知道了 ，总所周知链表肯定要有一个存和一个取得过程，上面的代码我们看到的是取，那么存在哪呢就在下面，当looper.loop()中处理完dispatchMessage会调用到下面的函数。

~~~
114   private static final int MAX_POOL_SIZE = 50;
...

291    void recycleUnchecked() {
292        // Mark the message as in use while it remains in the recycled object pool.
293        // Clear out all other details.
294        flags = FLAG_IN_USE;
295        what = 0;
296        arg1 = 0;
297        arg2 = 0;
298        obj = null;
299        replyTo = null;
300        sendingUid = -1;
301        when = 0;
302        target = null;
303        callback = null;
304        data = null;
305
306        synchronized (sPoolSync) {
307            if (sPoolSize < MAX_POOL_SIZE) {
308                next = sPool;
309                sPool = this;
310                sPoolSize++;
311            }
312        }
313    }
~~~


最后306行到313行 就可以看出，当sPool的个数没有大于50的时候，这个使用完的message就将会放回sPool中去，而且放在了第一个位置哦。当大于50是就是不忘pool中存了。因为篇幅限制这就不多写了[看这里吧](http://www.jianshu.com/p/f6f357b3db89)
这样message大部分核心的东西就完成了 




###Handler是什么？

Handler 翻译过来是处理者的意思，从翻译上来看显而易见handler里面承载着消息的处理，如果看过上述章节，你肯定知道handler.dispatchMessage来处理消息

~~~
93     public void dispatchMessage(Message msg) {
94         if (msg.callback != null) {
95             handleCallback(msg);
96         } else {
97             if (mCallback != null) {
98                 if (mCallback.handleMessage(msg)) {
99                     return;
100                }
101            }
102            handleMessage(msg);
103        }
104    }

~~~
从上面代码可以看出 如果有handlerCallback就他处理，没有就handlerMessage来处理 对于这个方法我就不说了，毕竟使用过handler的人大多数肯定事实现过这个方法。然后处理这的逻辑就通了，looper开启循环。messageQueue来获取message，获取到之后从message.target中获取handler根据得到的handler调用dispatchMessage来进行消息处理

**那么感觉好像少了点什么，那就是消息是怎么插入到messageQueue中的**

在我们使用handler消息机制的时候我们肯定有到过post或者post方法postDelayed等方法，其实内部都是汇集到一处的。我们以post为例

~~~
324  public final boolean post(Runnable r)
325    {
326       return  sendMessageDelayed(getPostMessage(r), 0);
327    }
~~~
这里有个地方很容易呗忽略getPostMessage方法

~~~
725    private static Message getPostMessage(Runnable r) {
726        Message m = Message.obtain();
727        m.callback = r;
728        return m;
729    }
~~~

看到了吧传说中的Message被建立了出来

~~~
565    public final boolean More ...sendMessageDelayed(Message msg, long delayMillis)
566    {
567        if (delayMillis < 0) {
568            delayMillis = 0;
569        }
570        return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
571    }
~~~

这里吧时间转换了下对应的出发时间后调用了sendMessageAtTime而这个方法就是post等方法最终调用方法

~~~
592    public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
593        MessageQueue queue = mQueue;
594        if (queue == null) {
595            RuntimeException e = new RuntimeException(
596                    this + " sendMessageAtTime() called with no mQueue");
597            Log.w("Looper", e.getMessage(), e);
598            return false;
599        }
600        return enqueueMessage(queue, msg, uptimeMillis);
601    }

~~~
要往MessageQueue里面插了

~~~
626    private boolean More ...enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
627        msg.target = this;
628        if (mAsynchronous) {
629            msg.setAsynchronous(true);
630        }
631        return queue.enqueueMessage(msg, uptimeMillis);
632    }
~~~

在像MessageQueue插入之前，对message的target赋值，将handler传了进去。也断代码也证实了之前所说的。之后的流程大家应该就已将知道了

总结下handler的整体流程  
***sendMessage->sendMessageDelayed->sendMessageAtTime->enqueueMessage


至此整个消息流程的分发机制算是整理完了 打算弄个流程图 不过暂时没有图片存取的地方 之后再添加吧 


拓展个知识点 
ActivityThread里面```1161    private class H extends Handler ```
这个里面定义了一些跟生命周期有关的东东 可以看一下 右后有时间写app启动流程的时候再细说


使用：怎么使用自己的Looper呢：[官方示例](https://developer.android.com/reference/android/os/Looper.html)
~~~
  class LooperThread extends Thread {
      public Handler mHandler;

      public void run() {
          Looper.prepare();

          mHandler = new Handler() {
              public void handleMessage(Message msg) {
                  // process incoming messages here
              }
          };

          Looper.loop();
      }
  }
~~~

感觉有没有很多？ 有封装哦 HandlerThread 这个类使用简单，代码两少



如果你有耐心开完这么多代码 你应该对消息机制有了一个整体的认识了









