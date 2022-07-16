## ForkJoin

任务由 `ForkJoinPool` 执行，这个线程池大小默认为机器核心大小

任务类型为 `ForkJoinTask` ，一般使用`RecursiveAction` ，这个类对外提供唯一一个函数抽象函数`compute` 只要实现这个函数即可，一般的模式是

```java
if (任务足够小或不可分) {
	 顺序计算该任务
} else {
	 将任务分成两个子任务
	 递归调用本方法，拆分每个子任务，等待所有子任务完成
	 合并每个子任务的结果
}
```

fork() 方法如下

```java
public final ForkJoinTask<V> fork() {
    Thread t; ForkJoinWorkerThread w;
		// push 入本线程的workQueue, 因为工作窃取机制，其他空闲的工作线程会获取这个task
    if ((t = Thread.currentThread()) instanceof ForkJoinWorkerThread)
        (w = (ForkJoinWorkerThread)t).workQueue.push(this, w.pool);
    // 这个方法说明可以不要创建Pool，直接调用task.compute()也行
		else
        ForkJoinPool.common.externalPush(this);
    return this;
}
```

join方法等待fork出去的task完成

```java
public final V join() {
    int s;
    if ((s = status) >= 0)
        s = awaitDone(null, false, false, false, 0L);
    if ((s &ABNORMAL) != 0)
        reportException(s);
    return getRawResult();
}
```

**工作窃取机制：**

每个线程都为分配给它的任务保存一个双向链式队列，每完成一个任务，就会从队列头上取出下一个任务开始执行。某个线程可能早早完成了分配给它的所有任务，也就是它的队列已经空了，而其他的线程还很忙。这时，这个线程并没有闲下来，而是随机选了一个别的线程，从队列的尾巴上“偷走”一个任务。这个过程一直继续下去，直到所有的任务都执行完毕，所有的队列都清空。这就是为什么要划成许多小任务而不是少数几个大任务，这有助于更好地在工作线程之间平衡负载。