---
layout:     post
title:      Reference 、ReferenceQueue 详解
subtitle:   
date:       2019-10-16
author:     tomas家的小拨浪鼓
header-img: img/post-bg-20191014-article1.jpg
catalog: true
tags:
    - Java
    - Java 引用类型
---

# Reference 、ReferenceQueue 详解

<br>
## ReferenceQueue
引用队列，在检测到适当的可到达性更改后，垃圾回收器将已注册的引用对象添加到该队列中

实现了一个队列的入队(enqueue)和出队(poll还有remove)操作，内部元素就是泛型的Reference，并且Queue的实现，是由Reference自身的链表结构( 单向循环链表 )所实现的。

ReferenceQueue名义上是一个队列，但实际内部并非有实际的存储结构，它的存储是依赖于内部节点之间的关系来表达。可以理解为queue是一个类似于链表的结构，这里的节点其实就是reference本身。可以理解为queue为一个链表的容器，其自己仅存储当前的head节点，而后面的节点由每个reference节点自己通过next来保持即可。

* 属性  
head：始终保存当前队列中最新要被处理的节点，可以认为queue为一个后进先出的队列。当新的节点进入时，采取以下的逻辑：

~~~java
            r.next = (head == null) ? r : head;
            head = r;
~~~
然后，在获取的时候，采取相应的逻辑：

~~~java
Reference<? extends T> r = head;
        if (r != null) {
            head = (r.next == r) ?
                null :
                r.next; // Unchecked due to the next field having a raw type in Reference
            r.queue = NULL;
            r.next = r;
~~~

* 方法  
enqueue()：待处理引用入队

~~~java
    boolean enqueue(Reference<? extends T> r) { /* Called only by Reference class */
        synchronized (lock) {
            // Check that since getting the lock this reference hasn't already been
            // enqueued (and even then removed)
            ReferenceQueue<?> queue = r.queue;
            if ((queue == NULL) || (queue == ENQUEUED)) {
                return false;
            }
            assert queue == this;
            r.queue = ENQUEUED;
            r.next = (head == null) ? r : head;
            head = r;
            queueLength++;
            if (r instanceof FinalReference) {
                sun.misc.VM.addFinalRefCount(1);
            }
            lock.notifyAll(); // ①
            return true;
        }
    }
~~~

① lock.notifyAll(); 👈通知外部程序之前阻塞在当前队列之上的情况。( 即之前一直没有拿到待处理的对象，如ReferenceQueue的remove()方法 )

<br/>
## Reference
java.lang.ref.Reference 为 软（soft）引用、弱（weak）引用、虚（phantom）引用的父类。

因为Reference对象和垃圾回收密切配合实现，该类可能不能被直接子类化。  
可以理解为Reference的直接子类都是由jvm定制化处理的,因此在代码中直接继承于Reference类型没有任何作用。但可以继承jvm定制的Reference的子类。  
例如：Cleaner 继承了 PhantomReference  
`public class Cleaner extends PhantomReference<Object> `

#### 构造函数
其内部提供2个构造函数，一个带queue，一个不带queue。其中queue的意义在于，我们可以在外部对这个queue进行监控。即如果有对象即将被回收，那么相应的reference对象就会被放到这个queue里。我们拿到reference，就可以再作一些事务。

**而如果不带的话，就只有不断地轮询reference对象，通过判断里面的get是否返回null( phantomReference对象不能这样作，其get始终返回null，因此它只有带queue的构造函数 )。这两种方法均有相应的使用场景，取决于实际的应用。如weakHashMap中就选择去查询queue的数据，来判定是否有对象将被回收。而ThreadLocalMap，则采用判断get()是否为null来作处理。**

~~~java
    /* -- Constructors -- */

    Reference(T referent) {
        this(referent, null);
    }

    Reference(T referent, ReferenceQueue<? super T> queue) {
        this.referent = referent;
        this.queue = (queue == null) ? ReferenceQueue.NULL : queue;
    }
~~~

如果我们在创建一个引用对象时，指定了ReferenceQueue，那么当引用对象指向的对象达到合适的状态（根据引用类型不同而不同）时，GC 会把引用对象本身添加到这个队列中，方便我们处理它，因为**“引用对象指向的对象 GC 会自动清理，但是引用对象本身也是对象（是对象就占用一定资源），所以需要我们自己清理。”**

#### Reference链表结构内部主要的成员有
① pending 和 discovered

~~~java
    /* List of References waiting to be enqueued.  The collector adds
     * References to this list, while the Reference-handler thread removes
     * them.  This list is protected by the above lock object. The
     * list uses the discovered field to link its elements.
     */
    private static Reference<Object> pending = null;

    /* When active:   next element in a discovered reference list maintained by GC (or this if last)
     *     pending:   next element in the pending list (or null if last)
     *   otherwise:   NULL
     */
    transient private Reference<T> discovered;  /* used by VM */
~~~

**可以理解为jvm在gc时会将要处理的对象放到这个静态字段上面。同时，另一个字段discovered：表示要处理的对象的下一个对象。即可以理解要处理的对象也是一个链表**，通过discovered进行排队，这边只需要不停地拿到pending，然后再通过discovered不断地拿到下一个对象赋值给pending即可，直到取到了最有一个。因为这个pending对象，两个线程都可能访问,因此需要加锁处理。

~~~java
if (pending != null) {
     r = pending;
    // 'instanceof' might throw OutOfMemoryError sometimes
    // so do this before un-linking 'r' from the 'pending' chain...
    c = r instanceof Cleaner ? (Cleaner) r : null;
    // unlink 'r' from 'pending' chain
    pending = r.discovered;
    r.discovered = null;
~~~

![](http://upload-images.jianshu.io/upload_images/4235178-5853714dfc64dedd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

② referent  
`private T referent;         /* Treated specially by GC */`  
👆referent字段由GC特别处理
referent：表示其引用的对象，即我们在构造的时候需要被包装在其中的对象。对象即将被回收的定义：此对象除了被reference引用之外没有其它引用了( 并非确实没有被引用，而是gcRoot可达性不可达,以避免循环引用的问题 )。如果一旦被回收，则会直接置为null，而外部程序可通过引用对象本身( 而不是referent，这里是reference#get() )了解到回收行为的产生( PhntomReference除外 )。

③ next  

~~~java
    /* When active:   NULL
     *     pending:   this
     *    Enqueued:   next reference in queue (or this if last)
     *    Inactive:   this
     */
    @SuppressWarnings("rawtypes")
    Reference next;
~~~
next：即描述当前引用节点所存储的下一个即将被处理的节点。但next仅在放到queue中才会有意义( 因为，只有在enqueue的时候，会将next设置为下一个要处理的Reference对象 )。为了描述相应的状态值，在放到队列当中后，其queue就不会再引用这个队列了。而是引用一个特殊的ENQUEUED。因为已经放到队列当中，并且不会再次放到队列当中。

④ discovered

~~~java
    /* When active:   next element in a discovered reference list maintained by GC (or this if last)
     *     pending:   next element in the pending list (or null if last)
     *   otherwise:   NULL
     */
    transient private Reference<T> discovered;  /* used by VM */
~~~

👆被VM使用  
discovered：当处于active状态时：discoverd reference的下一个元素是由GC操纵的( 如果是最后一个了则为this )；当处于pending状态：discovered为pending集合中的下一个元素( 如果是最后一个了则为null )；其他状态：discovered为null

⑤ lock

~~~java
    static private class Lock { }
    private static Lock lock = new Lock();
~~~

lock：在垃圾收集中用于同步的对象。收集器必须获取该锁在每次收集周期开始时。因此这是至关重要的：任何持有该锁的代码应该尽快完成，不分配新对象，并且避免调用用户代码。

⑥ pending

~~~java
    /* List of References waiting to be enqueued.  The collector adds
     * References to this list, while the Reference-handler thread removes
     * them.  This list is protected by the above lock object. The
     * list uses the discovered field to link its elements.
     */
    private static Reference<Object> pending = null;
~~~

pending：等待被入队的引用列表。收集器会添加引用到这个列表，直到Reference-handler线程移除了它们。这个列表被上面的lock对象保护。这个列表使用discovered字段来连接它自己的元素( 即pending的下一个元素就是discovered对象 )。

⑦ queue

`volatile ReferenceQueue<? super T> queue;`  
queue：是对象即将被回收时所要通知的队列。当对象即被回收时，整个reference对象( 而不是被回收的对象 )会被放到queue里面，然后外部程序即可通过监控这个queue拿到相应的数据了。  
**这里的queue( 即，ReferenceQueue对象 )名义上是一个队列，但实际内部并非有实际的存储结构，它的存储是依赖于内部节点之间的关系来表达。可以理解为queue是一个类似于链表的结构，这里的节点其实就是reference本身。可以理解为queue为一个链表的容器，其自己仅存储当前的head节点，而后面的节点由每个reference节点自己通过next来保持即可。**

* Reference 实例( 即Reference中的真是引用对象referent )的4中可能的内部状态值  
Queue的另一个作用是可以区分不同状态的Reference。Reference有4种状态，不同状态的reference其queue也不同：  
  * Active：新创建的引用对象都是这个状态，在 GC 检测到引用对象已经到达合适的reachability时，GC 会根据引用对象是否在创建时制定ReferenceQueue参数进行状态转移，如果指定了，那么转移到Pending，如果没指定，转移到Inactive。
  * Pending：pending-Reference列表中的引用都是这个状态，它们等着被内部线程ReferenceHandler处理入队（会调用ReferenceQueue.enqueue方法）。没有注册的实例不会进入这个状态。
  * Enqueued：相应的对象已经为待回收，并且相应的引用对象已经放到queue当中了。准备由外部线程来询问queue获取相应的数据。调用ReferenceQueue.enqueued方法后的Reference处于这个状态中。当Reference实例从它的ReferenceQueue移除后，它将成为Inactive。没有注册的实例不会进入这个状态。
  * Inactive：即此对象已经由外部从queue中获取到，并且已经处理掉了。即意味着此引用对象可以被回收，并且对内部封装的对象也可以被回收掉了( 实际的回收运行取决于clear动作是否被调用 )。可以理解为进入到此状态的肯定是应该被回收掉的。一旦一个Reference实例变为了Inactive，它的状态将不会再改变。

<br/>
jvm并不需要定义状态值来判断相应引用的状态处于哪个状态，只需要通过计算next和queue即可进行判断。

* Active：queue为创建一个Reference对象时传入的ReferenceQueue对象；如果ReferenceQueue对象为空或者没有传入ReferenceQueue对象，则为ReferenceQueue.NULL；next\==null；
* Pending：queue为初始化时传入ReferenceQueue对象；next\==this(由jvm设置)；
* Enqueue：当queue!=null && queue != ENQUEUED 时；设置queue为ENQUEUED；next为下一个要处理的reference对象，或者若为最后一个了next\==this；
* Inactive：queue = ReferenceQueue.NULL; next = this.

通过这个组合，收集器只需要检测next属性为了决定是否一个Reference实例需要特殊的处理：如果next\==null，则实例是active；如果next!=null，为了确保并发收集器能够发现active的Reference对象，而不会影响可能将enqueue()方法应用于这些对象的应用程序线程，收集器应通过discovered字段链接发现的对象。discovered字段也用于链接pending列表中的引用对象。
![](http://upload-images.jianshu.io/upload_images/4235178-9064e91bd7b20f31.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![](http://upload-images.jianshu.io/upload_images/4235178-3b78098f7c37053d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
👆外部从queue中获取Reference

* WeakReference对象进入到queue之后,相应的referent为null。
* SoftReference对象，如果对象在内存足够时，不会进入到queue，自然相应的referent不会为null。如果需要被处理( 内存不够或其它策略 )，则置相应的referent为null，然后进入到queue。通过debug发现，SoftReference是pending状态时，referent就已经是null了，说明此事referent已经被GC回收了。
* FinalReference对象，因为需要调用其finalize对象，因此其reference即使入queue，其referent也不会为null，即不会clear掉。
* PhantomReference对象，因为本身get实现为返回null。因此clear的作用不是很大。因为不管enqueue还是没有，都不会清除掉。

Q：👆如果PhantomReference对象不管enqueue还是没有，都不会清除掉reference对象，那么怎么办？这个reference对象不就一直存在这了？？而且JVM是会直接通过字段操作清除相应引用的，那么是不是JVM已经释放了系统底层资源，但java代码中该引用还未置null？？  
A：不会的，虽然PhantomReference有时候不会调用clear，如Cleaner对象 。但Cleaner的clean()方法只调用了remove(this)，这样当clean()执行完后，Cleaner就是一个无引用指向的对象了，也就是可被GC回收的对象。

active ——> pending ：Reference#tryHandlePending  
pending ——> enqueue ：ReferenceQueue#enqueue  
enqueue ——> inactive ：Reference#clear  

![](http://upload-images.jianshu.io/upload_images/4235178-4cd80e66a142755e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

<br/>
#### 重要方法
① clear()

~~~java
    /**
     * Clears this reference object.  Invoking this method will not cause this
     * object to be enqueued.
     *
     * <p> This method is invoked only by Java code; when the garbage collector
     * clears references it does so directly, without invoking this method.
     */
    public void clear() {
        this.referent = null;
    }
~~~

调用此方法不会导致此对象入队。此方法仅由Java代码调用；当垃圾收集器清除引用时，它直接执行，而不调用此方法。  
clear的语义就是将referent置null。  
清除引用对象所引用的原对象，这样通过get()方法就不能再访问到原对象了( PhantomReference除外 )。**从相应的设计思路来说，既然都进入到queue对象里面，就表示相应的对象需要被回收了，因为没有再访问原对象的必要。此方法不会由JVM调用，而JVM是直接通过字段操作清除相应的引用，其具体实现与当前方法相一致。**

② ReferenceHandler线程

~~~java
    static {
        ThreadGroup tg = Thread.currentThread().getThreadGroup();
        for (ThreadGroup tgn = tg;
             tgn != null;
             tg = tgn, tgn = tg.getParent());
        Thread handler = new ReferenceHandler(tg, "Reference Handler");
        /* If there were a special system-only priority greater than
         * MAX_PRIORITY, it would be used here
         */
        handler.setPriority(Thread.MAX_PRIORITY);
        handler.setDaemon(true);
        handler.start();

        // provide access in SharedSecrets
        SharedSecrets.setJavaLangRefAccess(new JavaLangRefAccess() {
            @Override
            public boolean tryHandlePendingReference() {
                return tryHandlePending(false);
            }
        });
    }
~~~

其优先级最高，可以理解为需要不断地处理引用对象。

~~~java
    private static class ReferenceHandler extends Thread {

        private static void ensureClassInitialized(Class<?> clazz) {
            try {
                Class.forName(clazz.getName(), true, clazz.getClassLoader());
            } catch (ClassNotFoundException e) {
                throw (Error) new NoClassDefFoundError(e.getMessage()).initCause(e);
            }
        }

        static {
            // pre-load and initialize InterruptedException and Cleaner classes
            // so that we don't get into trouble later in the run loop if there's
            // memory shortage while loading/initializing them lazily.
            ensureClassInitialized(InterruptedException.class);
            ensureClassInitialized(Cleaner.class);
        }

        ReferenceHandler(ThreadGroup g, String name) {
            super(g, name);
        }

        public void run() {
            while (true) {
                tryHandlePending(true);
            }
        }
    }
~~~

③ tryHandlePending()

~~~java
    /**
     * Try handle pending {@link Reference} if there is one.<p>
     * Return {@code true} as a hint that there might be another
     * {@link Reference} pending or {@code false} when there are no more pending
     * {@link Reference}s at the moment and the program can do some other
     * useful work instead of looping.
     *
     * @param waitForNotify if {@code true} and there was no pending
     *                      {@link Reference}, wait until notified from VM
     *                      or interrupted; if {@code false}, return immediately
     *                      when there is no pending {@link Reference}.
     * @return {@code true} if there was a {@link Reference} pending and it
     *         was processed, or we waited for notification and either got it
     *         or thread was interrupted before being notified;
     *         {@code false} otherwise.
     */
    static boolean tryHandlePending(boolean waitForNotify) {
        Reference<Object> r;
        Cleaner c;
        try {
            synchronized (lock) {
                if (pending != null) {
                    r = pending;
                    // 'instanceof' might throw OutOfMemoryError sometimes
                    // so do this before un-linking 'r' from the 'pending' chain...
                    c = r instanceof Cleaner ? (Cleaner) r : null;
                    // unlink 'r' from 'pending' chain
                    pending = r.discovered;
                    r.discovered = null;
                } else {
                    // The waiting on the lock may cause an OutOfMemoryError
                    // because it may try to allocate exception objects.
                    if (waitForNotify) {
                        lock.wait();
                    }
                    // retry if waited
                    return waitForNotify;
                }
            }
        } catch (OutOfMemoryError x) {
            // Give other threads CPU time so they hopefully drop some live references
            // and GC reclaims some space.
            // Also prevent CPU intensive spinning in case 'r instanceof Cleaner' above
            // persistently throws OOME for some time...
            Thread.yield();
            // retry
            return true;
        } catch (InterruptedException x) {
            // retry
            return true;
        }

        // Fast path for cleaners
        if (c != null) {
            c.clean();
            return true;
        }

        ReferenceQueue<? super Object> q = r.queue;
        if (q != ReferenceQueue.NULL) q.enqueue(r);
        return true;
    }
~~~

这个线程在Reference类的static构造块中启动，并且被设置为高优先级和daemon状态。此线程要做的事情，是不断的检查pending 是否为null，如果pending不为null，则将pending进行enqueue，否则线程进入wait状态。

由此可见，pending是由jvm来赋值的，当Reference内部的referent对象的可达状态改变时，jvm会将Reference对象放入pending链表。并且这里enqueue的队列是我们在初始化( 构造函数 )Reference对象时传进来的queue，如果传入了null( 实际使用的是ReferenceQueue.NULL )，则ReferenceHandler则不进行enqueue操作，所以只有非RefernceQueue.NULL的queue才会将Reference进行enqueue。

ReferenceQueue是作为 JVM GC与上层Reference对象管理之间的一个消息传递方式，它使得我们可以对所监听的对象引用可达发生变化时做一些处理

##### 参考
http://www.importnew.com/21633.html
http://hongjiang.info/java-referencequeue/
http://www.cnblogs.com/jabnih/p/6580665.html
http://www.importnew.com/20468.html
http://liujiacai.net/blog/2015/09/27/java-weakhashmap/