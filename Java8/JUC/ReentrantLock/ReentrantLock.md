# ReentrantLock 可重入锁
```
重点就在“可重入”
```

## 继承实现方法
```
public class ReentrantLock implements Lock, java.io.Serializable
```

## 主要成员对象
```
//继承了AQS
private final Sync sync;
abstract static class Sync extends AbstractQueuedSynchronizer {
    private static final long serialVersionUID = -5179523762034025860L;

    abstract void lock();

    final boolean nonfairTryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        //当state为0时，再次尝试通过CAS获取锁
        if (c == 0) {
            if (compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        //如果独占线程就是当前线程
        else if (current == getExclusiveOwnerThread()) {
            //state + acquires，这里就是可重入锁的核心内容
            //会将state累加来判断锁的获取次数
            //release的时候也会根据累加类释放锁资源
            int nextc = c + acquires;
            if (nextc < 0) // overflow
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }

    protected final boolean tryRelease(int releases) {
        //这里就在释放锁资源的时候，只将当前state - 1 
        int c = getState() - releases;
        if (Thread.currentThread() != getExclusiveOwnerThread())
            throw new IllegalMonitorStateException();
        boolean free = false;
        //如果释放完了，就将AOS(AQS extents AOS)的exclusiveOwnerThread置为null
        if (c == 0) {
            free = true;
            setExclusiveOwnerThread(null);
        }
        //重置state，根据返回值判断是否全部锁资源释放完毕
        setState(c);
        return free;
    }
    
    //非公平锁
    static final class NonfairSync extends Sync {
        private static final long serialVersionUID = 7316153563782823691L;

        final void lock() {
            //尝试通过CAS来获取锁
            if (compareAndSetState(0, 1))
                //将AOS的exclusiveOwnerThread设置为当前线程
                setExclusiveOwnerThread(Thread.currentThread());
            else
                //通过AQS的acquire方法获取资源
                //其中AQS的tryAcquire在下面被重写，实际执行Sync.nonfairTryAcquire方法
                acquire(1);
        }

        protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
    }


    //公平锁
    static final class FairSync extends Sync {
        private static final long serialVersionUID = -3000897897090466540L;

        final void lock() {
            acquire(1);
        }

        //公平锁方式获取资源
        protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                //AQS的hasQueuedPredecessors
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            //同非公平锁，通过state累加记录获取锁资源次数
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
    }
```

## 构造方法
```
    //公平锁和非公平锁（默认）
    public ReentrantLock() {
        sync = new NonfairSync();
    }
    public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }
```

## 主要方法
### lock/tryLock && unlock
```
    public void lock() {
        sync.lock();
    }
    //根据获取资源次数
    public boolean tryLock() {
        return sync.nonfairTryAcquire(1);
    }
    //根据设置超时时间获取
    public boolean tryLock(long timeout, TimeUnit unit)
            throws InterruptedException {
        return sync.tryAcquireNanos(1, unit.toNanos(timeout));
    }
    
    public void unlock() {
        //进入AQS执行tryRelease，所以最终还是执行Sync.tryRelease
        sync.release(1);
    }
```


## 综合理解
```
可重入：线程可以重复获取锁资源，通过state成员累加来记录获取锁的次数，并在release时释放同等次数。

公平or非公平锁：
非公平锁是默认让所有线程根据OS层面来随机抢占锁资源
公平锁则会根据判断，让所有线程依次排对的情况，插入等待队列尾部等待被执行

核心还是继承了AQS并实现内部方法
```



