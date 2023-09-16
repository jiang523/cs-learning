# AQS

简单来说，AQS使用一个volatile的int state和一个等待队列FIFO来实现代码同步。

用ReentrantLock举例，如果state的值为0，当前线程可以执行同步块，若为1，则代码已经有线程持有锁，则当前线程进入FIFO队列等待。

使用tryAcquire获取锁时，会获取当前节点的前驱节点，如果前驱节点是头节点，则尝试获取锁，获取成功则将当前节点设置为头节点。

## JUC包

* ReentrantLock
* CountDownLatch
* 信号量
