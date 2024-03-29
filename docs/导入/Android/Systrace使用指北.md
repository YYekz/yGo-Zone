
## 背景
作为一个开发，有没有如下这种裂开的时刻？测试提了一个卡顿bug给你，跟你说某些机型偶现，UI显示不流畅、不连贯，动画效果卡卡的不如其他手机或者iOS的流畅自然？遇到这种问题时，也不能自信的说不是bug或者说这个手机是特殊手机我们不兼容，让客户换手机。而且更恐怖的是，这种问题不是流程逻辑bug，基本没有什么日志可以供我们参考，那我们该怎么办呢？

那就引入我们今天的主角Systrace，相信作为Android开发，或多或少都听过这个工具，没错这是Android sdk自带的工具。

## 主角登场

### Systrace到底是什么？
[Systrace](https://developer.android.google.cn/topic/performance/tracing?hl=zh-cn) 是一款用于将设备活动保存到跟踪文件的 Android 工具。在搭载较低版本 Android 系统的设备上，跟踪文件会以 Systrace 格式保存。
Systrace 是平台提供的旧版命令行工具，可记录短时间内的设备活动，并保存在压缩的文本文件中。该工具会生成一份报告，其中汇总了 Android 内核中的数据，例如 CPU 调度程序、磁盘活动和应用线程。

### Perfetto是什么？
[Perfetto](https://ui.perfetto.dev/)是 Android 10 中引入的全新平台级跟踪工具。这是适用于 Android、Linux 和 Chrome 的更加通用和复杂的开源跟踪项目。与 Systrace 不同，它提供数据源超集，可让您以 protobuf 编码的二进制流形式记录任意长度的跟踪记录。在搭载 Android 10（API 级别 29）或更高版本的设备上，跟踪文件会以 Perfetto 格式保存。

#### 优点
Systrace和Perfetto（以下简称`这俩工具`）可以帮助我们梳理出整个手机（不止当前应用）从统计开始，到统计结束的所有相关进程、线程、系统服务、CPU利用率等全局信息，我们可以从中排查关键点，方便我们查询问题。

#### 不足
但 `这俩工具` 不会收集有关应用进程中代码执行情况的详细信息。如需详细了解您的应用正在执行哪些方法及其占用了多少 CPU 资源，请使用 Android Studio 中的 CPU 性能剖析器（也就是Android Studio内部自带的Profiler工具）。

### 怎么用？

#### 生成文件
**那么一直说的Systrace到底在哪里呢？怎么使用呢？**
工具位置其实就在我们sdk的目录下，具体路径为：
```
/Users/xxx/Library/Android/sdk/platform-tools/systrace
```
[相关的命令参数配置](https://developer.android.google.cn/topic/performance/tracing/command-line?hl=zh-cn)

一般来说比较常用的参数为：

1. -o : 指示输出文件的路径和名字
2. -t : 抓取时间(最新版本可以不用指定, 按 Enter 即可结束)
3. -b : 指定 buffer 大小 (一般情况下,默认的 Buffer 是够用的,如果你要抓很长的 Trae , 那么建议调大 Buffer )
4. -a : 指定 app 包名 (如果要 Debug 自定义的 Trace 点, 记得要加这个)

类似命令为：
```
python systrace.py -o mynewtrace.html sched freq idle am wm gfx view \
        binder_driver hal dalvik camera input res
```

#### 基础操作
快捷键的使用可以加快查看 Systrace 的速度,下面是一些常用的快捷键

W : 放大 Systrace , 放大可以更好地看清局部细节
S : 缩小 Systrace, 缩小以查看整体
A : 左移
D : 右移
M : 高亮选中当前鼠标点击的段(这个比较常用,可以快速标识出这个方法的左右边界和执行时间,方便上下查看)

鼠标模式快捷切换 : 主要是针对鼠标的工作模式进行切换 , 默认是 1 ,也就是选择模式,查看 Systrace 的时候,需要经常在各个模式之间切换 , 所以点击切换模式效率比较低,直接用快捷键切换效率要高很多。

数字键1 : 切换到 Selection 模式 , 这个模式下鼠标可以点击某一个段查看其详细信息, 一般打开 Systrace 默认就是这个模式 , 也是最常用的一个模式 , 配合 M 和 ASDW 可以做基本的操作；
数字键2 : 切换到 Pan 模式 , 这个模式下长按鼠标可以左右拖动, 有时候会用到；
数字键3 : 切换到 Zoom 模式 , 这个模式下长按鼠标可以放大和缩小, 有时候会用到；
数字键4 : 切换到 Timing 模式 , 这个模式下主要是用来衡量时间的,比如选择一个起点, 选择一个终点, 查看起点和终点这中间的操作所花费的时间。

#### 基础信息
**Linux 常见的进程状态**

1. D 无法中断的休眠状态（通常 IO 的进程）；
2. R 正在可运行队列中等待被调度的；
3. S 处于休眠状态；
4. T 停止或被追踪；
5. W 进入内存交换 （从内核2.6开始无效）；
6. X 死掉的进程 （基本很少見）；
7. Z 僵尸进程；
8. < 优先级高的进程
9. N 优先级较低的进程
10. L 有些页被锁进内存；
11. s 进程的领导者（在它之下有子进程）；
12. l 多进程的（使用 CLONE_THREAD, 类似 NPTL pthreads）；

**线程状态**

1. 绿色：运行中
    处在正常运行的状态下，很好理解。
2. 蓝色：可运行
    线程处在可运行但未轮到的状态，等待CPU调度。
3. 白色：休眠中
    线程无任务可做，也可能是因为线程在互斥锁上被阻塞。
4. 橘色：不可中断的休眠态 IO Block
    线程在IO上被阻塞或等待磁盘操作完成，一般底部信息都会表示出当前的callsite。
5. 紫色：不可中断的休眠态
    线程在另一个内核操作（通常是内存管理）上被阻塞。（一般是指陷入了内核态，也可能有问题）
    
## 用给我看看

这个工具使用是因为在查找A，B包启动速度差异明显的问题时使用的。（A包慢）

### 大体排查逻辑
1. 首先看了app主进程中的每个关键方法的执行时长，比如
    1. app进程创建的`ZygoteInit`；
    2. 主线程创建`ActivityThreadMain`；
    3. app的`bindApplication`；
    4. activity的`activityStart`；
    5. 还有视图响应绘制的`Choreographer#doFrame`；
2. 查看了第一步相关的操作执行时长和UI绘制卡顿之后，发现卡顿开始的时间其实是在`bindApplication`开始之后，前两步甚至A还要比B更快！那到底为什么呢？看主线程中的任务其实相差不大，甚至有些A包主线程的任务执行时间要略快于B包的执行时间；
3. 结合内核CPU的高占用率来看，仔细思考后觉得可能是同时执行了其他任务导致了CPU占用率高，然后仔细查找了其他线程，发现了一个叫AsyncIOThread的IO子线程从`bindApplication`的中段开始，一直在执行IO操作，直到首页快绘制完成为止，感觉差不多了；
4. 结合第三步中查询的问题在代码中找了找看看有没有类似的线程，于是看到了项目代码，查看了调用的地方，发现有很多IO操作使用了该线程，导致该线程持续占用CPU执行了7873ms；
5. 与对应同学沟通确认A包和B包具体的区别后得知，A包的APP加密形式和B包是不同的，使用了一种新的加密方式，可能会导致IO问题，用同样配置的其他包同手机测试了启动速度发现有同样问题，确认不是我们业务侧代码改动引起的启动缓慢，现已交由其他组同学排查。
## 结束
在查找过程中，也走了很多弯路，为了排查是否是业务侧代码问题，也使用了AS内部的Profiler工具查看火焰图看方法耗时，查看新增代码查找加载缓慢的原因等等。

**这大体就是最基础的Systrace使用方法。DONE**


### 参考文档

[Quickstart: Record traces on Android - Perfetto Tracing Docs](https://perfetto.dev/docs/quickstart/android-tracing#recording-a-trace-through-the-cmdline)
[systrace卡顿分析神器 - 掘金](https://juejin.cn/post/7066400362273439751)