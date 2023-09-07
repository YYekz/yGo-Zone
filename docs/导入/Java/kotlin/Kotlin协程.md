```plantuml

interface CoroutineScope
interface Continuation
interface CoroutineStackFrame

abstract class BaseContinuationImpl

Continuation <-- BaseContinuationImpl
CoroutineStackFrame <-- BaseContinuationImpl

```


[协程中的取消和异常 | 异常处理详解\_谷歌开发者的博客-CSDN博客](https://blog.csdn.net/googledevs/article/details/107437548)


### CoroutineStart - 协程启动模式

-   DEFAULT 立即调度，可以在执行前被取消
-   LAZY 需要时才启动，需要start、join等函数触发才可进行调度
-   ATOMIC 立即调度，协程肯定会执行，执行前不可以被取消
-   UNDISPATCHED 立即在当前线程执行，直到遇到第一个挂起点（可能切线程）

### coroutineScope & supervisorScope

首先这两个函数都是挂起函数，需要运行在协程内或挂起函数内。`supervisorScope`属于主从作用域，会继承父协程的上下文，它的特点就是子协程的异常不会影响父协程，它的设计应用场景多用于子协程为独立对等的任务实体的时候，比如一个下载器，每一个子协程都是一个下载任务，当一个下载任务异常时，它不应该影响其他的下载任务。`coroutineScope`和`supervisorScope`都会返回一个作用域，它俩的差别就是异常传播：`coroutineScope` 内部的异常会向上传播，子协程未捕获的异常会向上传递给父协程，任何一个子协程异常退出，会导致整体的退出；`supervisorScope` 内部的异常不会向上传播，一个子协程异常退出，不会影响父协程和兄弟协程的运行。

### 相关文章
[万字长文 - Kotlin 协程进阶 - 掘金](https://juejin.cn/post/6950616789390721037#heading-18)
[硬核万字解读：Kotlin 协程原理解析 - 开发者头条](https://toutiao.io/posts/vtq5kjj/preview)

[协程执行顺序](https://mp.weixin.qq.com/s/KLBXqE95yWJLZ-rjdnlWHQ)