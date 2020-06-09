# ReadSourceCode
记录学习点滴
```
基本数据类型：
	字节型：byte
	字符型：char
	数值型：int, long, short
	浮点型：float, double
	布尔型：boolean
```
```
包装类及自动装箱/拆箱(编译时)
```
```
String与StringBuffer与StringBuilder的区别与使用场景
```
```
常用的集合类型：ArrayList(复制新数组), HashMap(kv, hash散落, 红黑树O(n)=log(n))
```
```
常用的设计模式：单例，工厂，代理，观察者等
```
```
线程/多线程：Thread, Runnable(return void, not throws exception), Callable(return Object, throws exception), ThreadPool
线程池：ThreadPoolExecutor(内部执行过程，自定义线程池-BlockingQueue, ThreadFactory, RejectedExecutionHandler), ScheduledThreadPoolExecutor(带定时任务功能的线程池)
为了保证线程安全，无法通过注解向线程注入对象，需要自行通过ApplicationContext获取Bean！
```
```
锁：synchronized(JVM), lock(ReentrantLock, CountDownLatch等), CAS(轻量级锁,结合自旋), AQS(实现锁的基础)
```
```
线程安全的集合类型及juc并发包：CopyOnWriteArrayList(写时复制, 读多写少), ConCurrentHashMap(线程安全实现原理,锁的粒度,kv不可存null), JUC包相关内容
简单阅读了ConCurrentHashMap的源码，才理解了别人口中所说的"抢锁(初始化table)，自旋(未抢到则等待其他线程完全init)，CAS(若当前Node为null则直接CAS插入数据)，synchronize(主动加锁完成对链表or树的插入操作)"
```
```
JVM入门
```
```
JVM工具集-JMC与jsp,jstack命令
```
```
Spring:IOC, AOP实现及应用
```
```
SpringBoot：自动装配原理，自定义starter
```
```
DB
```
```
缓存
```
```
MQ
```
```
分布式框架(服务暴露过程, 分布式锁)
```