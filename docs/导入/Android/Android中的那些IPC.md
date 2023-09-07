## IPC
| IPC方式  | 数据拷贝次数 |
| -------- | ------------ |
| 共享内存 | 0            |
| Pipe     | 2            |
| Socket   | 2            |
| 消息队列 | 2            |
| Binder   | 1             |

那为什么在有这么多Linux自带的IPC机制的情况下，还需要添加Binder呢？
## Binder
### 优势
1. **安全性**。传统IPC只能由用户在数据包中填入UID/PID，接收方无法获得对方可靠的进程UID/PID从而无法鉴别身份，容易被恶意程序利用。可靠的身份标识只有由 IPC 机制在内核中添加。Binder 既支持实名 Binder，又支持匿名 Binder，安全性高。
2. **稳定性**。基于C/S架构，架构清晰、职责明确又相互独立，这一点上远优于共享内存。
3. **性能**。Socket作为通用接口，传输效率低，开销大，一般用于跨网络进程间的通信。消息队列和管道采用存储-转发方式，所以数据需要先从发送方缓冲区拷贝到内核开辟的缓冲区，再从内核缓冲区拷贝到接收方缓冲区，有两次数据拷贝。共享内存无需拷贝，但需要信号量控制，考虑并发问题，控制复杂难以使用。Binder需要一次内存拷贝仅次于共享内存。
在细说Binder之前，先来了解下一些基础知识。
### 基础知识
现在操作系统是采用虚拟存储器，对于32位操作系统，寻址空间（虚拟存储空间）就是4GB。操作系统的核心是内核，他主要是可以访问受保护的内存空间，也有访问底层硬件设备的权限，也就是可以和底层硬件设备交互。所以，为了保护操作系统的稳定性与安全性，操作系统从逻辑上将虚拟空间分为用户空间（低3GB）和内核空间（高1GB）。

那么问题来了，如果用户空间想访问内核资源怎么办呢？**系统调用**来了。

#### 系统调用
当用户空间调用例如文件操作、网络访问等时，为了突破隔离限制就需要系统调用了。系统调用是用户空间访问内核空间的**唯一方式**。

当一个进程处于内核态时，执行的内核代码会使用当前进程的内核栈（每个进程都有自己的内核栈）。

### 具体
传统IPC跨进程通信是需要内核空间做支持的。那Binder并不是Linux系统内核的一部分，怎么办？Linux的**动态内核加载模块（LKM）** 机制，他在运行时被链接到内核作为内核的一部分运行，这样Android系统可以通过动态添加一个内核模块运行在内核空间来作为进程之间通信的桥梁，显而易见，在Android中这个内核模块就是我们常说的**Binder驱动**。

Binder数据传输的目的可以概括成：**一个进程可以通过自己用户空间的虚拟地址访问另一个进程的数据**。

#### mmap的意义

所有的数据都是存储在物理内存上的，但进程访问内存只能通过虚拟地址。所以能访问必须有个前提：**虚拟地址和物理内存之间建立映射关系**。
> 若是映射关系不建立，访问会出错。信号11（SIGSEGV）的MAPPER就是描述这个的。

***虚拟地址和物理地址建立映射关系就是通过mmap完成***。当mmap被调用(flag=MAP_ANONYMOUS)时，会做两件事：
1. 分配一块连续的虚拟地址空间；
2. 更新这些虚拟地址空间对应的page table entity(PTE)；

mmap做完这两件事儿后，就会返回虚拟地址空间的起始地址。但**mmap调用结束后，其实并不会分配物理页**（lazy allocation），那就会有两个问题：

**没有新的物理页分配，那PTE更新了什么内容呢？**
PTE反映了一个虚拟地址和物理地址间的映射关系。如果没有新的物理页分配那所有的虚拟地址都喝同一个zero page(页内容全为0)建立了映射关系。

**如果后续使用mmap返回的虚拟地址访问内存会怎么样？**
拿到mmap返回的虚拟地址后，并不会产生屋里也分配，此时如果直接读取虚拟地址的值，会通过PTE访问到zero page，读出0。如果此时写数据，则会触发一个正常的copy-on-write（COW）机制，需要多少页，就分配多少物理页，也就极大的保证了物理资源的合理分配和使用。


#### 进程间用户空间、内核空间是否隔离？
**不同进程间的用户空间是完全隔离的，内核空间是共享的。**

**隔离指的是什么呢？共享又指的是什么？**
**隔离**是指不同进程的页表不同，**共享**是不同进程的页表相同。
在Linux中虚拟地址空间氛围用户空间+内核空间，管理用户空间时，页表隔离，管理内核空间时，页表共享，使用同一份页表。
![[Pasted image 20230226121817.png|500]]

#### 共享内存
虚拟地址只是为了内存访问封装的一层接口，数据总归是在物理内存上的。所以，若是想让A进程通过用户空间的虚拟地址访问到B进程中的数据，**最高效的办法就是修改A/B进程中某些虚拟地址的PTE，让虚拟地址映射到同一片物理区域上**，这样就不用拷贝了，数据在物理空间中也只有一份。但缺点就是要考虑各种同步机制。

#### 内存拷贝
共享内存的同步机制复杂的情况下，取而代之就是内存拷贝，让不同进程都拥有一块属于自己的内存区域，就不用数据考虑同步问题。由于不同进程的内核是共享的，所以使用内核作为数据中转站。

**传统IPC一次通信流程（两次拷贝）**
通常是发送方（进程）将待发送数据放在内存缓冲区，通过系统调用进入内核态，然后内核程序在内核空间分配内存，开辟一块新的内核缓冲区，调用`copy_from_user()`将数据从用户空间的内存缓冲区拷贝到内核空间的内核数据缓冲区。同样，在接收方（进程）的用户空间开辟一块内存缓冲区，然后内核程序调用`copy_to_user()`将数据从内核空间的内核数据缓冲区拷贝到接收进程用户空间的内存缓冲区。

既然两次拷贝都发生在一个进程的用户空间+内核空间，说明：**用户空间和内核空间的虚拟地址指向不同的物理页**，正因为此，所以才需要再拷贝，那我们让用户空间和内核空间指向同一个物理页不就ok了？这就是binder的实现原理。

### 早期版本（<= Android P）
为了减少一次拷贝，接收数据的进程必须同时满足下面三个条件：
1. 在用户空间分配一块连续区域A（仅虚拟地址分配）；
2. 在内核空间分配一块同样大小的连续区域B（仅虚拟地址分配）；
3. 在每次数据通信时，根据实际需求分配物理页，并将物理页同时映射到A、B中偏移相同的位置。

1和2 是在进程调用Binder的mmap时已经完成，3在每次数据通信时进行。

下面假设进程1发送数据，进程2接收数据，分析下内存拷贝发生在何时（以下发生在进程1，不过正在执行驱动代码陷入内核态）：
1. 由于进程2之前调用过mmap（只调用一次），因此它拥有用户空间的区域A和内核空间的区域B；
2. 得到即将发送的数据大小，根据数据大小分配实际的物理页；
3. 将刚刚分配出来的物理页映射到进程2的A、B区域；
4. 将用户空间的发送数据通过copy_from_user拷贝到内核区域B中；
5. 由于A、B映射了同样的物理页，因此B中的数据也可以通过A的地址读出来。

在整个过程中，只有步骤4发生了一次数据拷贝。但在这个过程中可能会有一个疑问，当前不是在进程1中么，步骤3怎么做到的？前文说过，因为内核空间共享，所以陷入内核态系统调用的进程1可以操作进程2的PTE。

从性能角度而言，几乎已经无可挑剔了，但有一个致命的稳定性缺点，导致**从Q版本开始，Binder内存拷贝的实现有了改变**。上文已知，从驱动执行完mmap后，进程会在内核空间分配一块虚拟地址区域B。对Android应用进程而言，B的默认大小是**1M-8k**([[#Binder Buffer为什么会是(1M - 8K)，而不是整数值1M？]])。
只要进程没有退出，这1M-8K的虚拟地址就会一直分配给他。

通常对于虚拟地址长时间占用是不会有问题的，但32为机器内存寻址为4GB，高1GB用作内核地址空间，但1GB又划分为：直接映射、vmalloc区、临时映射区和固定映射区。随着Android Treble项目（Android O引入）的启动，hardware binder进入大众视野，越来越多的HAL Service使用hw binder进行跨进程通信，所以原来用作分配的内核空间vmalloc区域占用成倍上升。当应用启动过多时，可能会导致vmalloc的虚拟地址被耗尽，导致报错，为了解决这个问题，简单的增大vmalloc区域并不是好的解决方案。即引入了新的实现。

### 最新版本
[Binder | 内存拷贝的本质和变迁 - 掘金](https://juejin.cn/post/6844904113046568973)，
简而言之，在新版本实现中，每拷贝一页的内容就调用一次kunmap将分配的内核空间虚拟地址释放掉，这样就不会发生长时间占用内核空间虚拟地址的情况。其实对于接收数据的进程而言，在用户空间和内核空间都拥有映射同一块物理区域的虚拟地址，kunmap只是放了内核空间的虚拟地址，接收端返回到上层后依然可以通过在用户空间的虚拟地址读取数据。



## Android应用层面
可以看到一次完整的进程间通信必须包含两个进程，通常称为：客户端进程+服务端进程，双方通过binder来实现通信。

### Client、Server、ServiceManager、驱动
从运行空间上来看，Client、Server、ServiceManager运行在用户空间，驱动运行在内核空间。其中ServiceManager和Binder驱动由系统提供，Service和Client通过应用程序来实现。Client、Server以及SM均是通过系统调用 open 、 mmap、ioctl来访问设备文件 /dev/binder来实现跨进程通信。
![[Pasted image 20230226170147.png|500]]
**ServiceManager**：将字符形式的Binder名字转化为Client中对该Binder的引用，其实就是提供获取Binder实体引用的管理类。Server创建了Binder实体之后，会起一个名字，将这个Binder实体连同名字以数据包的形式通过Binder驱动发送给SM，通知SM注册一个名为xxx的Binder，位于某个Server中。驱动为这个穿过进程边界的Binder创建位于内核中的实体节点以及SM对应实体的引用，将名字以及新建的引用打包传递给SM，SM收到数据包后，从中取出名字和引用填入查找表中。

其实在这里，Client、Server以及SM都是不同的进程，那我们Client要找到Server要通过SM，但SM也是独立进程，和SM通信都要使用跨进程通信，这不是死循环了么？
其实Client和Service对应SM来说，都算是Client，怎么理解呢？因为SM比较特殊，他没有名字也不需要注册，当一个进程使用`BINDER_SET_CONTEXT_MGR`命令将自己注册成SM时，binder驱动会自动为他创建Binder实体，这个binder的引用在所有Client中都固定位0而无需通过其他手段获得。所以一个Server要向SM中注册自己的Binder就必须通过0这个引用和SM进行跨进程通信。这样以来，Client可以通过0号引用像SM请求对应名字获取Binder引用了（其实到这时，这个Binder有两个引用，一个位于SM中，一个位于发起请求的Client中，如果接下来有新的请求该Binder则引用数继续叠加，而且这些引用属于强类型）。

![[Pasted image 20230226165913.png|600]]

#### 匿名binder
并不是所有binder都需要注册给SM广而告之的。Server可以通过已经建立的Binder连接将创建的Binder实体直接传递给Client，当然这条已经建立的连接是通过实名Binder建立的。由于这个匿名Binder没有向SM注册名字，所以Client通过这个匿名Binder的引用向Server中实体发送请求，是一条私密通道。

那有一个新的问题，就是**在Client进程使用Server进程的某个对象时，是同个对象吗？**

其实不是，在数据流经过Binder驱动时，驱动会对数据做一层转换。当A想要获取B的obj时，驱动并不会真的把object返回给A，而是返回一个跟obj看起来一模一样的代理对象objProxy，具备和obj一样的方法，这些方法只需要把请求参数交给驱动即可，驱动收到请求后，发现是objProxy就查找是对应哪个进程的代理对象，然后通知对应进程调用obj的对应方法，要求对应进程把结果返回给自己，当驱动拿到B进程的结果后转发给A进程，通信完成。

#### 具体代码
我们平时使用AIDL时，会生成如下类。

-   **IBinder** : IBinder 是一个接口，代表了一种跨进程通信的能力。只要实现了这个接口，这个对象就能跨进程传输。
-   **IInterface** : IInterface 代表的就是 Server 进程对象具备什么样的能力（能提供哪些方法，其实对应的就是 AIDL 文件中定义的接口）
-   **Binder** : Java 层的 Binder 类，代表的其实就是 Binder 本地对象。BinderProxy 类是 Binder 类的一个内部类，它代表远程进程的 Binder 对象的本地代理；这两个类都继承自 IBinder, 因而都具有跨进程传输的能力；实际上，在跨越进程的时候，Binder 驱动会自动完成这两个对象的转换。
-   **Stub** : AIDL 的时候，编译工具会给我们生成一个名为 Stub 的静态内部类；这个类继承了 Binder, 说明它是一个 Binder 本地对象，它实现了 IInterface 接口，表明它具有 Server 承诺给 Client 的能力；Stub 是一个抽象类，具体的 IInterface 的相关实现需要开发者自己实现。

在Stub中有两个方法需要重点说明：`asInterface`和`onTransact`。
```java
    public static BookManager asInterface(IBinder binder) {
        if (binder == null)
            return null;
        IInterface iin = binder.queryLocalInterface(DESCRIPTOR);
        if (iin != null && iin instanceof BookManager)
            return (BookManager) iin;
        return new Proxy(binder);
    }
    
    @Override
    protected boolean onTransact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
        switch (code) {

            case INTERFACE_TRANSACTION:
                reply.writeString(DESCRIPTOR);
                return true;

            case TRANSAVTION_addBook:
                data.enforceInterface(DESCRIPTOR);
                Book arg0 = null;
                if (data.readInt() != 0) {
                    arg0 = Book.CREATOR.createFromParcel(data);
                }
                this.addBook(arg0);
                reply.writeNoException();
                return true;

        }
        return super.onTransact(code, data, reply, flags);
    }
```

先说说 `asInterface`，当 Client 端在创建和服务端的连接，调用 bindService 时需要创建一个 ServiceConnection 对象作为入参。在 ServiceConnection 的回调方法 onServiceConnected 中 会通过这个 asInterface(IBinder binder) 拿到 BookManager 对象，这个 IBinder 类型的入参 binder 是驱动传给我们的，正如你在代码中看到的一样，方法中会去调用 binder.queryLocalInterface() 去查找 Binder 本地对象，如果找到了就说明 Client 和 Server 在同一进程，那么这个 binder 本身就是 Binder 本地对象，可以直接使用。否则说明是 binder 是个远程对象，也就是 BinderProxy。因此需要我们创建一个代理对象 Proxy，通过这个代理对象来是实现远程访问。

最后就是Proxy。
```java
public class Proxy implements BookManager {

    ...

    public Proxy(IBinder remote) {
        this.remote = remote;
    }

    @Override
    public void addBook(Book book) throws RemoteException {

        Parcel data = Parcel.obtain();
        Parcel replay = Parcel.obtain();
        try {
            data.writeInterfaceToken(DESCRIPTOR);
            if (book != null) {
                data.writeInt(1);
                book.writeToParcel(data, 0);
            } else {
                data.writeInt(0);
            }
            remote.transact(Stub.TRANSAVTION_addBook, data, replay, 0);
            replay.readException();
        } finally {
            replay.recycle();
            data.recycle();
        }
    }

    ...
}
```

在 Proxy 中的 addBook() 方法中首先通过 Parcel 将数据序列化，然后调用 remote.transact()。正如前文所述 Proxy 是在 Stub 的 asInterface 中创建，能走到创建 Proxy 这一步就说明 Proxy 构造函数的入参是 BinderProxy，即这里的 remote 是个 BinderProxy 对象。最终通过一系列的函数调用，Client 进程通过系统调用陷入内核态，Client 进程中执行 addBook() 的线程挂起等待返回；驱动完成一系列的操作之后唤醒 Server 进程，调用 Server 进程本地对象的 onTransact()。最终又走到了 Stub 中的 onTransact() 中，onTransact() 根据**函数编号**调用相关函数（在 Stub 类中为 BookManager 接口中的每个函数中定义了一个编号，只不过上面的源码中我们简化掉了；在跨进程调用的时候，不会传递函数而是传递编号来指明要调用哪个函数）；我们这个例子里面，调用了 Binder 本地对象的 addBook() 并将结果返回给驱动，驱动唤醒 Client 进程里刚刚挂起的线程并将结果返回。
## 其他的IPC方式在Android中的体现

View绘制的内存共享
Handler epoll的 pipe
Socket
[Android匿名共享内存（Ashmem）原理 - 掘金](https://juejin.cn/post/6844903504679534606)

## Q&A
### Binder Buffer为什么会是(1M - 8K)，而不是整数值1M？

如果一个进程使用ProcessState这个类来初始化Binder服务，这个进程的Binder内核内存上限就是BINDER_VM_SIZE，也就是1MB-8KB。
> [Cross Reference: /frameworks/native/libs/binder/ProcessState.cpp](http://androidxref.com/9.0.0_r3/xref/frameworks/native/libs/binder/ProcessState.cpp)

从GIT提交的记录可以看到下面的commits：kernel的“backing store”需要一个保护页，这使得1M用来分配碎片内存时变得很差，所以这里减去两页来提高效率，因为减去一页就变成了奇数。


## 相关文章
[Binder | 内存拷贝的本质和变迁 - 掘金](https://juejin.cn/post/6844904113046568973)
[Android Bander设计与实现 - 设计篇\_universus的博客-CSDN博客\_android binder 数组](https://blog.csdn.net/universus/article/details/6211589)
[写给 Android 应用工程师的 Binder 原理剖析 - 掘金](https://juejin.cn/post/6844903589635162126)
[3分钟带你看懂android的Binder机制 - 掘金](https://juejin.cn/post/6844903764986462221#comment)
[Android中mmap原理及应用简析 - 掘金](https://juejin.cn/post/6844903762146885640)
[图解Android中的binder机制 - 掘金](https://juejin.cn/post/6844904115777044488)
[MMKV 原理分析 一. 流程分析 - 掘金](https://juejin.cn/post/7078640657807441934)