## OkHttp
1. 最大连接数64，同一个host最大连接数为5，重定向次数最大为21次；
2. 支持http1.0、http1.1、http2.0、明文http2.0（不支持升级）、spdy、quic；
3. 同步线程在当前线程block等待结果，异步线程会使用线程池执行请求但结束时会在对应子线程；
4. 连接会使用socket实现，会根据当前是否传入了`sslSocketFactory`以及是否使用了HTTP代理判断是否建立tunnel（返回200说明建立成功返回null，若返回407则说明服务器需要代理认证，需要调用sslSocketFactory）；
5. 使用EventListener可以同步监测网络请求的执行步骤；

### tunnel是什么？
使用隧道传递的数据可以使不同协议的数据帧或包，简单来说就是使用一种网络协议传输另一种网络协议的数据。
**打开隧道**：http提供了一个connect方法，客户端发送一个connect请求给隧道网关请求打开一条tcp链接。

**不使用connect的隧道**：代理收到客户端的请求后，会重新创建请求并发送到目标服务器。当目标服务器返回数据后，代理会对response解析，重新组装然后发给客户端。所以这种方式的隧道，**代理可以对客户端和目标服务器之间的通信数据进行窥探和篡改**。
**使用connect的隧道**：和不使用connect的代理的唯一区别就是**不会对response做窥探和篡改，只转发**。

所以为什么HTTPS在有http代理的情况下要通过connect来建立ssl隧道，因为https的数据是加密后数据，代理在正常情况下无法对数据解密，保证了它的安全性。


SSL隧道：初衷是为了通过防火墙来传输加密的ssl数据，此时就是为了将非http的流量（ssl流量）传过防火墙到达指定的服务器。


### Okio的优点
增强了流与流之间的互动，使得当数据从一个缓冲区移动到另一个缓冲区时，可以不经过copy达到。okio优化了缓冲策略以减轻内存压力和性能消耗，并对于部分io场景，提供更友好的api。

[okhttp3源码分析之拦截器 - 掘金](https://juejin.cn/post/6844903945828040711)
[Okio好在哪 - 简书](https://www.jianshu.com/p/2fff6fe403dd)
[Android DiskLruCache完全解析，硬盘缓存的最佳方案\_guolin的博客-CSDN博客](https://blog.csdn.net/guolin_blog/article/details/28863651)
[浏览器 HTTP 协议缓存机制详解 - leejun2005的个人页面 - OSCHINA - 中文开源技术交流社区](https://my.oschina.net/leejun2005/blog/369148)
[面试官：听说你熟悉OkHttp原理？ - 掘金](https://juejin.cn/post/6844904087788453896#heading-1)
[OkHttp的完整指南 - 掘金](https://juejin.cn/post/7068162792154464264)

## Retrofit
Retrofit通过Java动态代理技术生成我们请求接口的对象，而我们调用对象的任何函数时，其实都转到`InvocationHandler`对象的`invoke`函数。而`invoke`函数的主要工作就是解析我们在接口中定义函数，包括解析注解，参数注解及值，以及返回类型，然后封装到`RequestFactory`，然后通过`HttpServiceMethod.parseAnnotations`函数，查找合适的请求适配器和响应转换器，最后将`RequestFactory`和适配器、转换器封装成`HttpServiceMethod`的子类，并调用其`invoke`函数，通过`OkHttp`发起网络请求。

## Glide

[Glide源码难看懂？用这个角度让你事半功倍！ - 掘金](https://juejin.cn/post/6994669144490639368)
[Android Fresco源码分析 | 自律使人自由](https://hningoba.github.io/2020/03/12/Android%20Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/)

## MMKV
缺点：Linux采用了分页来管理内存，存入数据先要创建一个文件，并要给这个文件分配一个固定的大小。如果存入了一个很小的数据，那么这个文件其余的内存就会被浪费。相反如果存入的数据比文件大，就需要动态扩容。

## LeakCanary
leakcanary 比较核心的一个原理就是利用了弱引用的一个特性，这个特性就是：

> 在创建弱引用的时候，可以指定一个 RefrenceQueue ，当弱引用引用的对象的可达性发生变化的时候，系统会把这个弱引用引用的对象放到之前指定的 RefrenceQueue 中等待处理。

所以 GC 后，引用对象仍然没有出现在 RefrenceQueue 的时候，说明可能发生了内存泄漏，这个时候 leakcanary 就会 dump 应用的 heap ，然后用 shark 库分析 heap ，找出一个到 GC ROOT 的最短引用链并提示。

### 一些常见的内存泄漏场景   
1. 静态变量持有 Context 等。
2.  单例实例持有 Context 等。
3.  一些回调没有反注册，比如广播的注册和反注册等，有时候一些第三方库也需要注意。
4. 一些 Listener 没有手动断开连接。
5.  匿名内部类持有外部类的实例。比如 Handler , Runnable 等常见的用匿名内部类的实现，常常会不小心持有 Context 等外部类实例。


## Dagger2 
