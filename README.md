# 复习笔记
记录学习点滴
## 基本数据类型
```
	字节型：byte
	字符型：char
	数值型：int, long, short
	浮点型：float, double
	布尔型：boolean
```
## 包装类及自动装箱/拆箱(编译时)
```
如常用的String， Integer等
```
## String与StringBuffer与StringBuilder
```
区别与使用场景，涉及的多线程性能，安全问题
```
## 常用的集合类型
```
ArrayList(复制新数组), HashMap(kv, hash散落, 红黑树O(n)=log(n))
常用数据集合的实现及特性
```
## 常用的设计模式
```
单例，工厂，代理等
设计模式经典应用场景
```
## 线程/多线程
```
Thread, Runnable(return void, not throws exception), Callable(return Object, throws exception), ThreadPool
线程池：ThreadPoolExecutor(内部执行过程，自定义线程池-BlockingQueue, ThreadFactory, RejectedExecutionHandler), ScheduledThreadPoolExecutor(带定时任务功能的线程池)
为了保证线程安全，无法通过向线程注入对象，需要自行通过ApplicationContext获取
```
## 锁
```
synchronized(JVM锁), lock(ReentrantLock, CountDownLatch等), CAS(轻量级锁,结合自旋), AQS(上述jdk锁的实现基类)
```
## juc并发包
```
线程安全的集合类型及：CopyOnWriteArrayList(写时复制, 读多写少), ConCurrentHashMap(线程安全实现原理,锁的粒度,kv不可存null), JUC包相关内容
简单阅读了ConCurrentHashMap的源码，才理解了别人口中所说的"抢锁(初始化table)，自旋(未抢到则等待其他线程完全init)，CAS(若当前Node为null则直接CAS插入数据)，synchronize(主动加锁完成对链表or树的插入操作)"
```
## JVM入门
```

```
## JVM工具集-JMC与jsp,jstack命令
```

```
## Spring:IOC, AOP实现及应用
```
IOC理念对应的DI实现。
Spring创建bean的BeanFactory(Spring默认)和ApplicationContext(基于用户使用)的过程，以及过程中通过三级缓存&分离创建、属性注入来解决单例模式下基于setter的循环依赖

AOP则需要理解代理模式，可用于方法前后日志输出和异常处理等(例如spring中的事务注解@Transactional底层通过aop实现)
```
## SpringBoot
```
启动流程，自动装配原理，自定义starter
```
## DB
```

```
## Memory Cache
```

```
## MQ
```

```
## 分布式框架
```
服务暴露过程, 分布式锁
```