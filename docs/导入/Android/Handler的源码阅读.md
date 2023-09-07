>本文源码基于Android 30版本。


Handler在Android中的使用非常广泛，今天从使用到原理来撸一遍源码，带着问题去看代码可能更容易理解，在工作中经常遇到的问题有：
1. 为什么主线程中不需要`Looper.prepare()`？
    在`ActivityThread`的`main`方法中，也就是java的启动类方法中调用了`Looper.prepareMainLooper()`和`Looper.loop()`。
    ```java
    class ActivityThread{
    public static void main(String[] args) {

        Looper.prepareMainLooper();
        
        long startSeq = 0;
        if (args != null) {
            for (int i = args.length - 1; i >= 0; --i) {
                if (args[i] != null && args[i].startsWith(PROC_START_SEQ_IDENT)) {
                    startSeq = Long.parseLong(
                            args[i].substring(PROC_START_SEQ_IDENT.length()));
                }
            }
        }
        ActivityThread thread = new ActivityThread();
        thread.attach(false, startSeq);

        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }

        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
        Looper.loop();

        throw new RuntimeException("Main thread loop unexpectedly exited");
        }
    }
    ```

2. Handler的多种使用方式的异同处有哪些？
3. Handler的组成？Looper、MessageQueue、ThreadLocal？
4. Handler的IdleHandler有用过吗？哪里有用过？
5. Handler是死循环，主线程Handler为什么不会造成阻塞？



##  相关文章
1. https://blog.csdn.net/start_mao/article/details/98963744
2. [Handler后传篇一: 为什么Looper中的Loop()方法不能导致主线程卡死? - 掘金](https://juejin.cn/post/6844903774096457736)
3. [都 2021 年了，还有人在研究 Handler？ - 掘金](https://juejin.cn/post/7020060105773154312#comment)
4. [Android组件系列：再谈Handler机制（Native篇） - 掘金](https://juejin.cn/post/7146239048191836190/)