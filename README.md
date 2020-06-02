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
常用的集合类型：ArrayList, HashMap
```
```
线程/多线程：Thread, Runnable, Callable, ThreadPool
线程池：ThreadPoolExecutor, ScheduledThreadPoolExecutor
```
```
锁：synchronized(JVM), lock(ReentrantLock, CountDownLatch等), CAS, AQS
```
```
线程安全的集合类型及juc并发包：CopyOnWriteArrayList, ConCurrentHashMap, JUC包相关内容
简单阅读了ConCurrentHashMap的源码，才理解了别人口中所说的"抢锁(初始化table)，自旋(未抢到则等待其他线程完全init)，CAS(若当前Node为null则直接CAS插入数据)，synchronize(主动加锁完成对链表or树的插入操作)"
```
