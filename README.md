# ReadSourceCode
记录学习点滴
```
基本数据类型：
	字符型：String, Char
	数值型：int, long, short
	浮点型：float, double
	布尔型：boolean
```
```
String与StringBuffer与StringBuilder的区别与使用情况
```
```
常用的集合类型：ArrayList, HashMap
```
```
锁：synchronized, lock, CAS, AQS
```
```
多线程：Thread, Runnable, Callable, ThreadPool
线程池：ThreadPoolExecutor, ScheduledThreadPoolExecutor
```
```
线程安全的集合类型及juc并发包：CopyOnWriteArrayList, ConCurrentHashMap, JUC包相关内容
简单阅读了ConCurrentHashMap的源码，才理解了别人口中所说的"抢锁(初始化table)，自旋(未抢到则等待其他线程完全init)，CAS(若当前Node为null则直接CAS插入数据)，synchronize(主动加锁完成对链表or树的插入操作)"
```
