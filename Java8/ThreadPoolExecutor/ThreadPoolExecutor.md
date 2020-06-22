# ThreadPoolExecutor
## 主要成员

```
    //工作队列
    private final BlockingQueue<Runnable> workQueue;
    //可重入锁
    private final ReentrantLock mainLock = new ReentrantLock();
    //执行线程的worker
    private final HashSet<Worker> workers = new HashSet<Worker>();
    //线程等待条件
    private final Condition termination = mainLock.newCondition();
    
    private int largestPoolSize;
    //线程工厂
    private volatile ThreadFactory threadFactory;
    //拒绝策略
    private volatile RejectedExecutionHandler handler;
    //线程保活时间
    private volatile long keepAliveTime;
    //核心线程保活时间
    private volatile boolean allowCoreThreadTimeOut;
    //核心线程池大小
    private volatile int corePoolSize;
    //最大线程池大小
    private volatile int maximumPoolSize;
    //默认拒绝策略 -- 抛异常
    private static final RejectedExecutionHandler defaultHandler = new AbortPolicy();
    
    //通过构造器来解析上述成员
    public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) {
        //参数校验
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        //参数校验
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        //线程上下文
        this.acc = System.getSecurityManager() == null ?
                null :
                AccessController.getContext();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```

## 主要执行逻辑方法execute

```
public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        /*
         * Proceed in 3 steps:
         *
         * 1. If fewer than corePoolSize threads are running, try to
         * start a new thread with the given command as its first
         * task.  The call to addWorker atomically checks runState and
         * workerCount, and so prevents false alarms that would add
         * threads when it shouldn't, by returning false.
         *
         * 2. If a task can be successfully queued, then we still need
         * to double-check whether we should have added a thread
         * (because existing ones died since last checking) or that
         * the pool shut down since entry into this method. So we
         * recheck state and if necessary roll back the enqueuing if
         * stopped, or start a new thread if there are none.
         *
         * 3. If we cannot queue task, then we try to add a new
         * thread.  If it fails, we know we are shut down or saturated
         * and so reject the task.
         */
        //获取正在工作的线程 or 0
        int c = ctl.get();
        //判断线程是否超出corePoolSize
        if (workerCountOf(c) < corePoolSize) {
            //判断是否添加线程成功，true代表核心线程池
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        //尝试向workQueue添加任务
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            //确保线程处于Running，否则将回滚workQueue.offer
            if (! isRunning(recheck) && remove(command))
                reject(command);
            //如果还没有非核心线程则直接在非核心线程池创建新线程
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        //如果向非核心线程池创建线程失败，则执行拒绝策略
        else if (!addWorker(command, false))
            reject(command);
    }
```
## 添加线程任务 addWorker

```
    private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            //todo -- 没搞明白这个“仅在必要时检查队列是否为空”
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            for (;;) {
                int wc = workerCountOf(c);
                //判断当前线程数量是否超限
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                //通过CAS给workerCount + 1
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get();  // Re-read ctl
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    int rs = runStateOf(ctl.get());
                    //如果还是running状态 or shutdown状态 && task为null
                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        //如果线程还活着，那肯定不能再添加到线程池
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        //往HashSet里添加任务
                        workers.add(w);
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                //如果添加成功，就让线程启动，等待cpu时间片
                if (workerAdded) {
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }
    
    //如果线程添加失败或者线程启动失败则需要回滚
    private void addWorkerFailed(Worker w) {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            if (w != null)
                workers.remove(w);
            decrementWorkerCount();
            tryTerminate();
        } finally {
            mainLock.unlock();
        }
    }
```
### 拒绝策略

```
AbortPolicy  默认直接抛出异常RejectedExecutionException  
DiscardPolicy 抛弃新任务，无异常
DiscardOldestPolicy 抛弃队列里最早的没被处理的任务
CallerRunsPolicy 使用当前调用线程来执行当前任务
```


## 结合理解
```
通过源码，实际了解了线程池的工作原理：判断核心线程池，工作队列，非核心线程池，拒绝策略
进一步延申的话就需要进一步理解线程状态以及不同队列的工作原理以及AbstractQueuedSynchronizer(AQS)。
```

## 延申：为什么阿里规范必须使用ThreadPoolExecutor来创建线程池
### 资源较少/有限的情况下：
### newCachedThreadPool
```
    //SynchronousQueue为不存数据的队列，按照规则会一致创建非核心线程池
    //线程池内线程若一直创建，可能导致OOM
    public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }
```

### newSingleThreadExecutor
```
    //LinkedBlockingQueue的最大长度为Integer.MAX_VALUE，若提交任务过多也可能导致OOM
    public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }
```

### newFixedThreadPool
```
    //同newSingleThreadExecutor类似，LinkedBlockingQueue的问题
    public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }
```






