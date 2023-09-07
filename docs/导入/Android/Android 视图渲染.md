## Window

### WindowManagerService


Android中的窗口主要分为三种：系统窗口、应用窗口、子窗口。

Toast就属于系统窗口，而Dialog、Activity属于应用窗口，不过Dialog必须依附Activity才能存在。PopupWindow算是子窗口，必须依附到其他窗口，依附的窗口可以使应用窗口也可以是系统窗口，但是不能是子窗口。



## 参考文章

[Android窗口管理分析（1）：View如何绘制到屏幕上的主观理解 - 掘金](https://juejin.cn/post/6844903492318920718)
[Android窗口管理分析（2）：WindowManagerService窗口管理之Window添加流程 - 掘金](https://juejin.cn/post/6844903492792877069)
[Android图形显示流程简介](https://mp.weixin.qq.com/s/LOa6u051OPPEPUASjvIbPg)

[「Android渲染」图像是怎样显示到屏幕上的？\_Android渲染\_李小四\_InfoQ写作社区](https://xie.infoq.cn/article/174897b20b1c230506eba4124)
