# 线程池

## 线程的状态

新建 就绪 阻塞 运行 结束

### 线程池提交任务

```java
 public static void main(String[] args) throws InterruptedException {
     Executor threadPoolExecutor = Executors.newSingleThreadExecutor();
     threadPoolExecutor.execute(() -> System.out.println("task is executing"));
 }
```

### 线程池执行任务过程

1. 当前线程数小于核心线程数，新建线程执行任务
2. 如果阻塞队列未满，将当前任务加到阻塞队列
3. 如果最大线程数未满，新建线程执行当前任务
4. 最大线程数已满，执行拒绝策略

```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    int c = ctl.get();
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        if (! isRunning(recheck) && remove(command))
            reject(command);
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    else if (!addWorker(command, false))
        reject(command);
}
```

##

### addWorker

addWorker做的最主要的一件事就是新建了一个Worker对象，Worker类的构造方法

```java
Worker(Runnable firstTask) {
    setState(-1); // inhibit interrupts until runWorker
    this.firstTask = firstTask;
    this.thread = getThreadFactory().newThread(this);
}
```

初始化了firstTask和thread两个对象，其中thread对象是调用线程池传入的自定义ThreadFactory来创建的一个线程。

而firstTask，指的是当前worker执行的第一个task。这么说有点难以理解，具体的来说，一个worker对应线程池的一个线程，而一个线程池的线程执行task分为两种情况，一种是任务被提交到线程池新建了一个worker来执行，一种是worker主动到阻塞队列消费task来执行，而这里的firstTask其实就是worker新建的时候执行的那个task，如果定义一个secondTask，则是worker从阻塞队列里消费的task。



### runWorker

worker实现了Runnable接口，线程启动后会执行run()方法，run方法中调用了runWorker()方法

```java
final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
            while (task != null || (task = getTask()) != null) {
                w.lock();
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        afterExecute(task, thrown);
                    }
                } finally {
                    task = null;
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            processWorkerExit(w, completedAbruptly);
        }
    }
```

runWorker方法用了一个死循环，在task不为空且阻塞队列里的任务不为空时这个循环将一直执行。

也就是说，每个worker会一直从阻塞队列里获取任务执行，直到阻塞队列为空或者发生异常，会跳出循环。

这两种跳出循环的姿势会在processWorkerExit中有不同的执行方式。

正常跳出循环completedAbruptly变量会是false，发生异常的情况会是true。

## ForkJoinPool

窃取算法



### 线程池如何处理异常
