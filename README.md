# ReadSourceCode
记录学习点滴
```
20200520 -- 简单阅读了ConCurrentHashMap的源码，才理解了别人口中所说的"抢锁(初始化table)，自旋(未抢到则等待其他线程完全init)，CAS(若当前Node为null则直接CAS插入数据)，synchronize(主动加锁完成对链表or树的插入操作)"
```
