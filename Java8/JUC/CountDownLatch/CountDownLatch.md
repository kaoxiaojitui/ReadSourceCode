# CountDownLatch
## 内部成员介绍
```
    //内部类Sync，继承队列同步器
    private final Sync sync;
    private static final class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 4982264981922014374L;

        Sync(int count) {
            setState(count);
        }

        int getCount() {
            return getState();
        }
        
        //tryAcquireShared & tryReleaseShared 实现AQS定义的方法来维护state
        protected int tryAcquireShared(int acquires) {
            return (getState() == 0) ? 1 : -1;
        }

        protected boolean tryReleaseShared(int releases) {
            // Decrement count; signal when transition to zero
            for (;;) {
                int c = getState();
                if (c == 0)
                    return false;
                int nextc = c-1;
                if (compareAndSetState(c, nextc))
                    return nextc == 0;
            }
        }
    }

```
## 构造器

```
    //传入一个计数器总数，就会创建一个继承了AQS的Sync对象
    public CountDownLatch(int count) {
        if (count < 0) throw new IllegalArgumentException("count < 0");
        this.sync = new Sync(count);
    }
```

## countDown

```
    //减少闭锁数量count-1，当闭锁数为0时，释放所有等待的线程。
    //如果当前闭锁数已经是0，那就什么都不会做
    public void countDown() {
        //结合AQS，这里实际时执行Sync.tryReleaseShared
        sync.releaseShared(1);
    }
```
## await
```
    //线程等待就可能被中断，从而有InterruptedException。例如Thread.sleep()
    //是当前线程等待闭锁数count变成0，除非出现InterruptedException
    public void await() throws InterruptedException {
        //这里会走到AQS的acquireSharedInterruptibly，再回到Sync.tryAcquireShared
        sync.acquireSharedInterruptibly(1);
    }
    
    //方法重载，timeout为最大等待时间
    public boolean await(long timeout, TimeUnit unit)
        throws InterruptedException {
        return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
    }
```
## 结合理解
```
可以理解为：创建一个闭锁CountDownLatch及对应信号量（n），将锁分发给各个不同线程。
在线程执行任务过程后，将闭锁的信号量减一。
但是当调用await方法时，主线程不会继续向下执行，而是必须等分发给予锁的线程将任务执行完毕，并调用countDown后，是闭锁的信号量变为0，才可以继续执行await方法的逻辑。

主要方法还是通过继承了AQS的Sync调用实现，故还是要进一步理解AQS。
```




