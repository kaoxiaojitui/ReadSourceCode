# AbstractQueuedSynchronizer
```
需要结合各种锁的具体实现来看，才能体现各个锁的特性(ReentrantLock, CountDownLatch)
```


## 继承实现关系(abstract)与projected构造器
```
//抽象类，不可以直接实例化，需要被继承。
//例如：CountDownLatch的Sync， ThreadPoolExecutor的Worker
    public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer
    implements java.io.Serializable
    
    protected AbstractQueuedSynchronizer() { }
```
## 主要成员

```
    //核心就是维护了这个state，通过getState/setState/compareAndSetState(CAS)
    private volatile int state;
    
    protected final int getState() {
        return state;
    }
    
    protected final void setState(int newState) {
        state = newState;
    }
    
    protected final boolean compareAndSetState(int expect, int update) {
        return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
    }
```


## 内部类Node
```
    //维护一个FIFO的双向等待队列
    static final class Node {
        //SHARED 节点（Semaphore/CountDownLatch）
        static final Node SHARED = new Node();
        //独占节点（ReentrantLock） final -> null
        static final Node EXCLUSIVE = null;
        //线程各个状态值
        static final int CANCELLED =  1;

        static final int SIGNAL    = -1;

        static final int CONDITION = -2;

        static final int PROPAGATE = -3;

        //SIGNAL,CANCELLED,CONDITION,PROPAGATE,0
        volatile int waitStatus;

        //双向链表
        volatile Node prev;

        volatile Node next;

        
        volatile Thread thread;


        Node nextWaiter;

        /**
         * Returns true if node is waiting in shared mode.
         */
        final boolean isShared() {
            return nextWaiter == SHARED;
        }

        //返回链表前一个节点
        final Node predecessor() throws NullPointerException {
            Node p = prev;
            if (p == null)
                throw new NullPointerException();
            else
                return p;
        }

        Node() {    // Used to establish initial head or SHARED marker
        }

        Node(Thread thread, Node mode) {     // Used by addWaiter
            this.nextWaiter = mode;
            this.thread = thread;
        }

        Node(Thread thread, int waitStatus) { // Used by Condition
            this.waitStatus = waitStatus;
            this.thread = thread;
        }
    }
```

## acquire -- Node.EXCLUSIVE 互斥模式，可以参考ReentrantLock的实现
```
    public final void acquire(int arg) {
        //首先tryAcquire尝试获取资源，获取到（返回true，表达式为(false&&?)==true）则直接返回
        if (!tryAcquire(arg) &&
            //如果没有获取到
            //addWaiter将线程以Node.EXCLUSIVE加入等待队列尾部
            //acquireQueued是线程在等待队列获取资源
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            //线程自我中断
            selfInterrupt();
    }
```
### tryAcquire
```
    //子类需要重写的方法
    protected boolean tryAcquire(int arg) {
        throw new UnsupportedOperationException();
    }
```
### addWaiter
```
    //将当前线程和线程对应的模式节点插入到等待队列尾部
    //this.nextWaiter = mode;
    //Node.EXCLUSIVE for exclusive, Node.SHARED for shared
    private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        //尾插
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        enq(node);
        return node;
    }
```
### acquireQueued
```
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                //如果当前节点前一个节点是头结点，并且再次尝试获取资源成功
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    //不会执行cancelAcquire
                    failed = false;
                    //获取到了资源就返回false，acquire也就直接返回了
                    return interrupted;
                }
                //对当前节点的前一个节点的状态进行判断来决定操作(将前一个节点的状态改为SIGNAL)
                if (shouldParkAfterFailedAcquire(p, node) &&
                    //将线程挂起
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            //这里是只有出现异常时才会执行cancelAcquire
            if (failed)
                cancelAcquire(node);
        }
    }
```
### shouldParkAfterFailedAcquire
```

    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        //当前一个接地那状态为SIGNAL时
        if (ws == Node.SIGNAL)
            /*
             * This node has already set status asking a release
             * to signal it, so it can safely park.
             */
            return true;
        //如果是cancelled状态
        if (ws > 0) {
            /*
             * Predecessor was cancelled. Skip over predecessors and
             * indicate retry.
             */
            //一直向前找到waitStatus不为cancelled的节点
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            /*
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             */
            //否则将前一个节点的waitStatus设置为SIGNAL
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }
```
### parkAndCheckInterrupt
```
    private final boolean parkAndCheckInterrupt() {
        //当前线程将被挂起
        LockSupport.park(this);
        return Thread.interrupted();
    }
```


### selfInterrupt
```
    //自我中断
    static void selfInterrupt() {
        Thread.currentThread().interrupt();
    }
```


## enq

```
    //将node插入到队列中
    private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            //懒加载机制，当不存在尾节点时，才创建节点
            if (t == null) {
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                //尾插节点
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
```


## release 释放锁

```
    public final boolean release(int arg) {
        //尝试释放锁
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```

### tryRelease
```
    //需要被实现的方法
    protected boolean tryRelease(int arg) {
        throw new UnsupportedOperationException();
    }
```

### unparkSuccessor
```
    private void unparkSuccessor(Node node) {
        /*
         * If status is negative (i.e., possibly needing signal) try
         * to clear in anticipation of signalling.  It is OK if this
         * fails or if status is changed by waiting thread.
         */
        int ws = node.waitStatus;
        //如果状态不为cancelled，就将waitStatus置为0
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);

        /*
         * Thread to unpark is held in successor, which is normally
         * just the next node.  But if cancelled or apparently null,
         * traverse backwards from tail to find the actual
         * non-cancelled successor.
         */
        Node s = node.next;
        //如果队列中没有下一个节点，或者下个节点的waitStatus是cancelled
        if (s == null || s.waitStatus > 0) {
            //头节点的next指针就指向null，GC
            s = null;
            //后续非cancelled节点指针迁移
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        //唤醒头节点后的线程 Node s = node.next
        if (s != null)
            LockSupport.unpark(s.thread);
    }
```

## hasQueuedPredecessors -> FairSync 公平锁的执行过程
### ReentrantLock.FairSync.tryAcquire -> AQS.hasQueuedPredecessors
```
    //大前提是state==0，即当前没有线程获取锁资源
    //判断等待队列中是否有
    public final boolean hasQueuedPredecessors() {
        // The correctness of this depends on head being initialized
        // before tail and on head.next being accurate if the current
        // thread is first in queue.
        Node t = tail; // Read fields in reverse initialization order
        Node h = head;
        Node s;
        //1.queue为空时或者仅有一个节点时，返回false，则可以尝试获取资源
        return h != t &&
            //2.queue不为空，head.next不为空时，且 head.next.thread是当前线程时，返回false
            ((s = h.next) == null || s.thread != Thread.currentThread());
        /**这里的意思是1中判断队列仅有一个节点，那按顺序就时当前节点排对在第二个
         *2中判断队列中第二个节点就是当前线程创建的节点
         *综上，才可以按照公平条件直接让当前线程获取锁资源
         *如果返回true，外层的！判断会是逻辑结束，tryAcquire返回false后，则会进入AQS.acquire中acquireQueued，将线程加入到等待队列尾部
        */
    }
```

## Condition 线程执行时条件
### await
```
    public final void await() throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        //通过Condition队列添加新节点
        Node node = addConditionWaiter();
        int savedState = fullyRelease(node);
        int interruptMode = 0;
        //当此线程被其他线程唤醒时，isOnSyncQueue返回false，跳出循环
        while (!isOnSyncQueue(node)) {
            //node为Condition队列的头节点则挂起当前线程
            LockSupport.park(this);
            if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                break;
        }
        //通过acquireQueued来重新在等待队列中获取资源
        if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
            interruptMode = REINTERRUPT;
        if (node.nextWaiter != null) // clean up if cancelled
            unlinkCancelledWaiters();
        if (interruptMode != 0)
            reportInterruptAfterWait(interruptMode);
    }   
```
#### addConditionWaiter
```
    //将线程加入到Condition队列
    private Node addConditionWaiter() {
        Node t = lastWaiter;
        // If lastWaiter is cancelled, clean out.
        if (t != null && t.waitStatus != Node.CONDITION) {
            unlinkCancelledWaiters();
            t = lastWaiter;
        }
        //创建新节点加入Condition队列
        Node node = new Node(Thread.currentThread(), Node.CONDITION);
        if (t == null)
            firstWaiter = node;
        else
            t.nextWaiter = node;
        lastWaiter = node;
        return node;
    }
```
#### fullyRelease
```
    //将当前线程资源释放release()，等待被唤醒
    final int fullyRelease(Node node) {
        boolean failed = true;
        try {
            int savedState = getState();
            if (release(savedState)) {
                failed = false;
                return savedState;
            } else {
                throw new IllegalMonitorStateException();
            }
        } finally {
            if (failed)
                node.waitStatus = Node.CANCELLED;
        }
    }
```
#### isOnSyncQueue
```
    //判断当前线程在Condition队列中是否是头节点
    final boolean isOnSyncQueue(Node node) {
        if (node.waitStatus == Node.CONDITION || node.prev == null)
            return false;
        //当后面的线程唤醒前面线程后，此处waitStatus就被设置为0，且队列中已有多个节点，返回true
        if (node.next != null) // If has successor, it must be on queue
            return true;
        /*
         * node.prev can be non-null, but not yet on queue because
         * the CAS to place it on queue can fail. So we have to
         * traverse from tail to make sure it actually made it.  It
         * will always be near the tail in calls to this method, and
         * unless the CAS failed (which is unlikely), it will be
         * there, so we hardly ever traverse much.
         */
        return findNodeFromTail(node);
    }
```


### signal
```
    public final void signal() {
        //判断当前线程是否为获取锁的线程
        if (!isHeldExclusively())
            throw new IllegalMonitorStateException();
        Node first = firstWaiter;
        if (first != null)
            doSignal(first);
    }
```
#### doSignal
```
    private void doSignal(Node first) {
        do {
            if ( (firstWaiter = first.nextWaiter) == null)
                lastWaiter = null;
            first.nextWaiter = null;
        } while (!transferForSignal(first) &&
                 (first = firstWaiter) != null);
    }
```
#### transferForSignal
```
    final boolean transferForSignal(Node node) {
        //尝试将Condition节点的waitStatus设置为0
        if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
            return false;

        //将当前线程尾插到等待队列
        Node p = enq(node);
        int ws = p.waitStatus;
        //1.判断前一个节点的状态是否cancelled
        //2.将前一个节点的状态设置为SIGNAL
        //3.唤醒前一个节点线程，回到await中的while + isOnSyncQueue判断 
        //or 前一个节点线程完毕，唤醒当前节点线程
        if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
            LockSupport.unpark(node.thread);
        return true;
    }
```



