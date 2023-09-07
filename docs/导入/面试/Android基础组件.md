## Bitmap

Bitmap是一个对象，如果要存储成本地可以查看的图片文件，必须对bitmap进行编码然后通过io流写到本地file。

**Bitmap对象内存变化**：Android8.0之前，Bitmap像素占用的内存分配在堆中；8.0之后占用的内存分配在Native堆。Native堆内存分配上限很大32位应用可用内存为3-4G。

[高性能图片优化方案 - 掘金](https://juejin.cn/post/7154949546882105374)
## Handler

源码：[【源码篇】Handler那些事（万字图文）\_Java\_小呆呆666\_InfoQ写作社区](https://xie.infoq.cn/article/1b3325e36483a51ed801b59e0)
### 主线程为什么不会因为 Looper.loop里的死循环卡死？

当获取消息的next方法没有消息时，会触发`nativePollOnce`线程进入休眠状态释放CPU资源，MessageQueue`nativeWake`方法可以解除休眠状态。底层其实使用epoll + pipe实现的，没有消息主线程会阻塞在pipe的读端，block让出cpu，有消息epoll会往pie里写一个字符，唤醒主线程执行消息。

[Android中为什么主线程不会因为Looper.loop()里的死循环卡死？ - 知乎](https://www.zhihu.com/question/34652589)

### 为什么Android的Handler不适用Binder而使用管道？

Handler是用作**同进程中不同线程间的通信**，所以线程不需要将内容从一个线程拷贝到另一个线程。
pipe的主要作用就是为了当一个线程A准备好message放入消息队列中时，需要通知另一个线程B去处理这个消息，线程A就会向pipe中写入数据1（老版本Android是w），pipe有数据就会唤醒线程B去处理消息。管道的主要工作就是**通知另一个线程**。
所以为什么不用binder？因为不是binder不能实现而是太浪费cpu和内存资源了，binder采用c/s架构，常用于不同进程间通信，从内存角度设计一次数据拷贝，handler同进程无需数据拷贝。而且从cpu角度来看，binder通信底层驱动还要维护一个binder线程池，每次设计binder线程的创建和内存分配比较浪费cpu资源。

[为什么Android的Handler采用管道而不使用Binder？ - 知乎](https://www.zhihu.com/question/44329366)

### runWithScissors方法是干嘛的？
**在一个线程中通过Handler向另外一个线程发送一个任务，并等待另一个线程处理此任务后再继续执行。**
其实就是实现了一个BlockingRunnable，使用synchronized保证临界区，使用wait阻塞。如果设置timeout还会被唤醒，直接返回false，标识执行失败。
缺点：
1. 如果超时了，无取消逻辑。即使超时了，runnable依然在目标线程的mq中没有被移除掉，最终还是会被handler线程调度并执行；
2. 造成死锁。阻塞时如果持有别的锁就会造成死锁。
所以说如果要使用尽量保证handler的looper不允许退出，主线程就不会退出。若要退出使用quitSafely方式。

[Handler(五)-runWithScissors - 简书](https://www.jianshu.com/p/5fbed33377e8)

[Android Framework 笔记 - 掘金](https://juejin.cn/post/7028895615304073223)



## LiveData


```java
protected void postValue(T value) {  
    boolean postTask;  
    synchronized (mDataLock) {  
        postTask = mPendingData == NOT_SET;  
        mPendingData = value;  
    }  
    if (!postTask) {  
        return;  
    }  
    //主线程使用handler、子线程使用线程池
    ArchTaskExecutor.getInstance().postToMainThread(mPostValueRunnable);  
}

@MainThread  
protected void setValue(T value) {  
    assertMainThread("setValue");  
    mVersion++;  
    mData = value;  
    dispatchingValue(null);  
}
```