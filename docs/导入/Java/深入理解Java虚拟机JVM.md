# 深入理解Java虚拟机JVM

## 第2章 Java内存区域与内存溢出异常

### 程序计数器

是一块较小的内存空间，它可以看作是当前线程所执行的字节码的行号指示器。为了线程切换后能恢复到正确的执行位置，每条线程都需要有一个独立的程序计数器，各条线程之间计数器互不影响，独立存储，我们称这类内存区域为“**线程私有**”的内存。

如果线程正在执行的是一个Java方法，这个计数器记录的是正在执行的虚拟机字节码指令的地 址;如果正在执行的是本地(Native)方法，这个计数器值则应为空(U ndefined)。此内存区域是唯一一个在《Java虚拟机规范》中没有规定任何OutOfMemoryError情况的区域。

### Java虚拟机栈

**线程私有。生命周期与线程同步。**

虚拟机栈描述的是Java方法执行的线程内存模型:每个方法被执行的时候，Java虚拟机都会同步创建一个栈帧[1](Stack Frame)用于**存储局部变量表、操作数栈、动态连接、方法出口**等信息。每一个方法被调用直至执行完毕的过程，就对应着一个栈帧在虚拟机栈中从入栈到出栈的过程。

局部变量表存放了编译期可知的各种Java虚拟机基本数据类型、对象引用（reference类型，它并不等同于对象本身，可能是一个指向对象起始 地址的引用指针，也可能是指向一个代表对象的句柄或者其他与此对象相关的位置）和returnAddress类型（指向了一条字节码指令的地址）。

这些数据类型在局部变量表中的存储空间以局部变量槽（slot）来表示，64位长度的`long`和`double`类型的数据会占用两个变量槽，其余数据只占用一个。局部变量表所需的内存空间在编 译期间完成分配，当进入一个方法时，这个方法需要在栈帧中分配多大的局部变量空间是完全确定 的，在方法运行期间不会改变局部变量表的大小。这里说的“大小”是指变量槽的数量， 虚拟机真正使用多大的内存空间(譬如按照1个变量槽占用32个比特、64个比特，或者更多)来实现一 个变量槽，这是完全由具体的虚拟机实现自行决定的事情。

> 在《Java虚拟机规范》中，对这个内存区域规定了两类异常：程请求的栈深度大于虚 拟机所允许的深度，将抛出StackOverflowError异常;如果Java虚拟机栈容量可以动态扩展[2]，当栈扩展时无法申请到足够的内存会抛出OutOfMemoryError异常。

### 本地方法栈

和虚拟机栈发挥的作用是非常相似的，其区域只是虚拟机栈执行Java方法（字节码）服务，而本地方法栈则是为虚拟机使用到的本地（Native）方法服务。

### Java堆

Java堆是虚拟机管理内存中最大的一块，是被所有线程共享的一块内存区域，在虚拟机启动时创建。此内存区域的唯一目的是存放对象实例，“几乎所有的对象实例都在这里分配内存”。

### 方法区

方法区和Java堆一样，是各个线程共享的内存区域，它用于存储已经被虚拟机加载的类型信息、常量、静态变量、即时编译器编译后的代码缓存等数据。

> 在JDK 1.7 的HotSpot，已经把原来放在永久代中的字符串常量池、静态变量等移出，到了JDK 1.8 完全放弃了永久代的概念，用本地内存中实现的元空间来替代，把JDK中永久代还剩余的其他内容（主要是类型信息）全部移到元空间中。


## 垃圾收集器与内存分配策略

### 引用计数算法

有引用则+1，GC时引用数为0则回收。若有循环引用，则会导致无法释放内存泄漏。

### 可达性分析算法

通过一系列称为“GC Roots”的根对象作为起始节点集，从这些节点开始，根据引用关系向下搜索，搜索过程所走过的路径称为“引用链”(Reference Chain)，如果某个对象到GC Roots间没有任何引用链相连，或者用图论的话来说就是从GC Roots到这个对象不可达时，则证明此对象是不可能再被使用的。

GCRoots的对象包括以下几种：
- 在虚拟机栈（栈帧中的本地变量表）中引用的对象，比如各个线程中杯调用的方法堆栈中使用到的参数、局部变量、临时变量等；
- 在方法去中类静态属性引用的对象，比如Java类的引用类型静态变量；
- 在方法区中常量引用的对象，比如字符串常量池里面的引用；
- 在本地方法栈中JNI引用的对象；
- Java虚拟机内部的引用，如基本数据类型对应的Class对象，一些常驻的异常对象等，还有系统类加载器；
- 所有被同步锁持有的对象；

### 引用？
在JDK1.2 之后，Java对引用的概念进行了扩充，分为四种：**强引用、软引用、弱引用、虚引用**。

**强引用**：必须的对象。`Object obj = new Object()`这种普遍存在的赋值关系。只要强引用关系还存在，无论任何情况下，GC永远不会回收掉被引用的对象；
**软引用**：有用但非必须的对象。软引用海关连着的对象，只有在系统要发生内存溢出异常前，才会把这些对象列入回收范围之中进行二次回收，如果还是没有足够的内存，才会抛出内存溢出异常；
**弱引用**：非必须的对象。强度比软引用更弱一些，被弱引用关联的对象只能生存到下一次GC发生时为止；
**虚引用**：最弱的一种关系，一个对象是否有虚引用的 存在，完全不会对其生存时间构成影响，也无法通过虚引用来取得一个对象实例。

### 回收方法区

方法区的垃圾回收主要回收两部分内容：**废弃的常量和不再使用的类型**。

回收废弃常量与回收Java堆中的对象非常类似。一个曾进入常量池的常量，在当前系统中已经没有任何字符串对象引用，且在虚拟机中也没有其他地方引用这个字面量，如果这时发生GC，且垃圾收集器判定有必要的话，这个常量就会被清理出常量池。
判定一个类型是否属于「不再被使用的类」要满足三个条件才可达到“被允许”（并不是和对象一样，没有引用了发生GC就必然回收）：
1. 该类的所有实例都已经被回收，就是说Java堆中不存在该类及其任何派生子类的实例；
2. 加载该类的类加载器已经被回收（很难达成）；
3. 该类对应的`java.lang.Class`对象没有在任何地方被引用，无法在任何地方通过反射访问该类的方法。

## 第六章 类文件结构

每个Class文件的头4个字节被称为**魔数**。它的唯一作用是确定这个文件是否为一个能被虚拟机接受的Class文件。

紧接着魔数的四个字节存储的是Class文件的版本号：第5和第6个字节是**次版本号**，第7和第8个字节是**主版本号**。

接下来是常量池入口，常量池可以比喻为Class文件里的资源仓库，是占用Class文件空间最大的项目之一，常量池中常量的数量是不固定的。所以常量池入口需要放置一项u2类型的数据，代表常量池容量计数器（constant_pool_count），这个容量计数器从1而不是0开始。

> 为什么从1开始而不是从0开始？
> 如果后面某些指向常量池的索引值的数据在特定情况下需要表达“不引用任何一个常量池项目”的意义，可以把索引值设置为0来表示。

常量池中主要存放两大类常量：**字面量**和**符号引用**。

字面量指比较接近Java语言的常量概念，如文本字符串、被final声明的常量值等。
符号引用属于编译原理方面的概念，主要包括一下几类常量：
1. 被模块导出或者开放的包；
2. 类和接口的全限定名；
3. 字段的名称和描述符；
4. 方法的名称和描述符；
5. 方法句柄和方法类型；
6. 动态调用点和动态常量；

在常量池结束之后，紧接着的2个字节代表访问标志，这个标志用于识别一些类或者接口层次的访问信息（这个Class是类还是接口？是否定义为public类型？是否定义为abstract类型？如果是累的话，是否final？等等）。

接下来的是**类索引**、**父类索引**和**接口索引集合**依次排在访问标志之后，类索引和父类索引分别为一个u2类型的数据，接口索引是一组u2类型的数据集合。

> 类索引表示当前类的全限定名为u2类型。
> 父类索引表示该类的父类，因为java单继承所以除了java.lang.Object之外，所有的类都有且仅有一个父类为u2类型。
> 接口索引集合表示该类实现的所有接口，从左到右依次排列，因为java多实现，所以是一组u2类型。接口索引结合入口的第一项u2类型的数据为接口计数器，表示索引表的容量。如果该类没有实现任何接口，则计数器值为0，后面接口索引表不再占用任何字节。

接下来就是**字段表集合**，用于描述接口或者类中声明的变量。Java语言中的「字段（Field）」包括类级变量以及实例级变量，但不包括在方法内部声明的局部变量。字段修饰符放在`access_flags`中是一个u2类型，跟随着的是`name_index`和`descriptor_index`，他们都是对常量池项的引用，分别代表着字段的简单名称以及字段和方法的描述符。

> 什么是简单名称、描述符和全限定名？
> **全限定名**就是将类的全路径展示出来“.”替换为“/”，例如“com/suda/simple/Test”，在使用时一般最后会加一个“；”表示全限定名结束。
> **简单名称**就是指没有类型和参数修饰的方法或者字段名称。
> **描述符**是用来描述字段的数据类型、方法的参数列表（包括数量、类型和顺序）以及返回值。

字段表集合中不会列出从父类或者父接口中继承而来的字段，但有可能出现原本Java代码之中不存在的字段(比如在内部类中为了保持对外部类的访问，编译器会自动添加对外部类实例的字段)。*在Java语言中字段是无法重载的，两个字段的数据类型、修饰符不管是否相同，都必须使用不一样的名称，但是对于Class文件格式来讲，只要两个字段的描述符不是完全相同，那字段重名就是合法的。*

字段表集合之后就是方发表集合，Class文件存储格式中对方法的描述和对字段的描述采用几乎完全一致的方式。方法里的代码经过Javac编译器编译成字节码指令之后，存放在方法属性表集合中一个名为“Code”的属性里面。与字段表集合相对应地，如果父类方法在子类中没有被重写(Override)，方法表集合中就不会出现来自父类的方法信息。但同样地，有可能会出现由编译器自动添加的方法，最常见的便是类构造器“<clinit>()”方法和实例构造器“<init>()”方法。

**Code属性**：Java程序方法体里面的代码经过Javac编译器处理之后，最终变为字节码指令存储在Code属性中。Code属性出现在方法表的属性集合之中，但并非所有的方法都必须存在这个属性。（接口或者抽象类中的方法就不存在Code属性）

> 一个无参的方法为什么`Args_size`会是1？而且无论是在参数列表里还是方法体里，都没有定义任何局部变量，那`Locals`为什么会是1？
> Java语言里面的潜规则：在任何实例方法里面，都可以通过`this`关键字访问到此方法所属的对象。是通过Javac编译器编译时把对this关键字的防蚊转变为对一个普通方法参数的防蚊，然后在虚拟机调用此实例方法时自动传入次参数而已。因此在实力方法的局部变量表中至少会存在一个指向当前对象实例的局部变量，局部变量表也会预留出第一个变量槽位来存放对象实例的引用。（方法声明为`static`，`Args_size`就为0了）

`ConstantValue`属性的作用是通知虚拟机自动为静态变量赋值。只有被static关键字修饰的类变量才能使用这项属性。虚拟机对这两种变量赋值的方式和时刻都有所不同。
- 对非`static`的实例变量的赋值就是在实例构造器`<init>()`方法中进行的；
- 对于`static`的类变量的赋值有两种方式选择：在类构造器`<clinit>()`方法中或者使用`ConstantValue`属性。
> 目前Oracle公司Javac编译器选择是，如果使用`final static`来修是一个变量（常量），并且这个变量的数据类型是基本类型或者`java.lang.String`的话，就会生成`ConstantValue`属性来进行初始化，如果变量没有`final`修饰，或者并非基本类型及字符串，则会选择在`<clinit>()`方法中进行初始化。


## 第七章 虚拟机类加载机制

### 类加载的时机

一个类从被夹在到虚拟机内存中开始，到卸载出内存位置，他的整个生命周期会经历：**加载、验证、准备、解析、初始化、使用和卸载**，一种**验证、准备和解析**三个部分统称为**连接**。
![](16465516373187.jpg)
如图中所示，加载、验证、准备、初始化和卸载这五个阶段的顺序是确定的，类型加载过程必须按照顺序进行，而解析阶段则不一定：有些情况下可以再初始化阶段之后再进行（Java的动态绑定）。

什么时候会开始进行类加载过程中的第一个阶段“加载”呢？有且只有六种情况必须立即对类进行“初始化”：
1. 遇到new、getstatic、putstatic或invokestatic这四条字节码指令时，如果类型没有进行过舒适化，则需要先出法其初始化阶段。能够生成这四条指令的典型场景有：
    1. 使用`new`关键字实例化对象时；
    2. 读取或设置一个类型的静态字段（被final修饰、已在编译器把结果放入常量池的静态字段除外）时；
    3. 调用一个类型的静态方法时；
2. 使用`java.lang.reflect`的方法对类型进行反射的时候，如果类型没有进行过初始化，则需要先出发其初始化；
3. 当初始化类的时候，如果父类还没有进行过初始化，则需要先出法其父类的初始化；
4. 当虚拟机启动时，用户需要制定一个要执行的主类（也就是包含`main()`的类），虚拟机会先初始化这个主类；
5. 当使用JDK 7新加入的动态语言支持时，如果一个java.lang.invoke.MethodHandle实例最后的解析结果为REF_getStatic、REF_putStatic、REF_invokeStatic、REF_newInvokeSpecial四种类型的方法句柄，并且这个方法句柄对应的类没有进行过初始化，则需要先触发其初始化；
6. 当一个借口中定义了JDK 8 新加入的默认方法（被`defalut`关键字修饰的接口方法）时，如果这个接口的实现类发生了初始化，则该接口要在其之前被初始化。

> 接口的加载过程与类加载过程稍有不同，针对接口需要做一些特殊说明:接口也有初始化过程， 这点与类是一致的，上面的代码都是用静态语句块“static{}”来输出初始化信息的，而接口中不能使用“static{}”语句块，但编译器仍然会为接口生成“<clinit>()”类构造器[2]，用于初始化接口中所定义的成员变量。接口与类真正有所区别的是前面讲述的六种“有且仅有”需要触发初始化场景中的第三种: 当一个类在初始化时，要求其父类全部都已经初始化过了，但是一个接口在初始化时，并不要求其父接口全部都完成了初始化，只有在真正使用到父接口的时候(如引用接口中定义的常量)才会初始化。

#### 加载
是整个类加载过程中的一个阶段。在这个阶段，虚拟机需要完成以下三件事儿：
1. 通过一个类的全限定名来获取定义此类的二进制字节流；
2. 将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构；
3. 在内存中生成一个代表这个类的java.lang.Class对象，作为方法区这个类的各种数据的访问入口；

> 数组比较特殊，情况有所不同，数组类本身不通过类加载器创建，它是由Java虚拟机直接在内存中动态构建出来的。

#### 验证
验证是连接的第一步。是为了确保Class文件的字节流中包含的信息符合《Java虚拟机规范》的全部约束要求。大概会完成下面四个阶段的检验动作：**文件格式验证、元数据验证、字节码验证和符号引用验证**。

**文件格式验证**：第一阶段要验证字节码流是否符合Class文件格式的规范，并且被当前版本的虚拟机处理。该验证阶段的主要目的是保证输入的字节流能正确地解析并存储在方法区内。这个阶段的验证是基于二进制字节流进行的，通过了这一阶段，字节流才会被允许进入Java虚拟机内存的方法区中进行存储，后面三个阶段都是基于方法区的存储结构进行的，不会再直接读取、操作字节流了。

**元数据验证**：主要目的是对类的元数据信息进行语义校验，保证不存在与《Java语言规范》定义相悖的元数据信息。

**字节码验证**：第三阶段是整个验证过程中最复杂的一个阶段，主要目的是通过数据流分析和控制流分析，确定程序语义是合法的、符合逻辑的。在第二阶段对元数据信息中的数据类型校验完毕以后，这阶段就要对类的方法体(Class文件中的Code属性)进行校验分析，保证被校验类的方法在运行时不会做出危害虚拟机安全的行为。

**符号引用验证**：最后一个阶段的校验行为发生在虚拟机将符号引用转化为直接引用的时候，这个转化动作将在连接的第三阶段——解析阶段中发生。符号引用验证可以看作是对类自身以外(常量池中的各种符号引用)的各类信息进行匹配性校验，通俗来说就是，该类是否缺少或者被禁止访问它依赖的某些外部类、方法、字段等资源。

> 验证阶段对于虚拟机的类加载机制来说，是一个非常重要的、但却不是必须要执行的阶段，因为验证阶段只有通过或者不通过的差别，只要通过了验证，其后就对程序运行期没有任何影响了。

#### 准备

准备阶段是正式为类中定义的变量（被`static`修饰的静态变量）分配内存并设置类变量初始值的阶段，这时进行内存分配的仅包括类变量，而不包括实例变量，实例变量将会随着对象实例化时随着对象一起分配在Java堆中。从概念上理解这些类变量所使用的内存应当在方法区中分配（**JDK8之前，HotSpot用永久代实现方法区，而JDK8及以后类变量会随着Class对象一起放在Java堆中**，所以类变量放在方法区中完全是一种对逻辑概念的表述了）。且这时进行的初始化通常情况下是数据的零值而不是代码设置的默认初始值，因为此时还未执行任何Java方法，赋值操作是在类构造器执行`<clinit>()`也就是类的初始化阶段才会被执行。

#### 解析

解析阶段是Java虚拟机将常量池内的符号引用替换为直接引用的过程。

**符号引用**：符号引用以一组符号来描述所引用的目标，符号可以是任何形式的字面量，只要使用时能无歧义地定位到目标即可。符号引用与虚拟机实现的内存布局无关，引用的目标并不一定是已经加载到虚拟机内存当中的内容。
**直接引用**：符号引用以一组符号来描述所引用的目标，符号可以是任何形式的字面量，只要使用时能无歧义地定位到目标即可。符号引用与虚拟机实现的内存布局无关，引用的目标并不一定是已经加载到虚拟机内存当中的内容。

#### 初始化

类的初始化阶段是类加载过程的最后一个步骤。知道初始化阶段，Java虚拟机才真正开始执行类中编写的Java代码，将主导权移交给应用程序。初始化阶段就是执行类构造器`<clinit>()`方法的过程。

> <clinit>()方法干嘛的？
> <clinit>() 是由编译器自动手记类中的所有类变量的赋值动作和静态语句块（static{}）中的语句合并产生的，代码顺序按照原文件中出现的顺序决定，静态语句块只能访问到定义在静态语句块之前的变量，定义在它之后的变量，在前面的静态语句块可以复制，但不能访问。<clinit>() 与 类的构造函数（在虚拟机视角中的实例构造器<init>()）不同，他不需要显式地调用父类构造器，Java虚拟机会保证在子类的<clinit>()方法执行前，父类<clinit>()方法已经执行完毕。因此在Java虚拟机中第一个被执行的<clinit>()方法的类型肯定是java.lang.Object。
> <clinit>()方法对于类或接口来说并不是必需的，如果一个类中没有静态语句块，也没有对变量的 赋值操作，那么编译器可以不为这个类生成<clinit>()方法。
> 接口中不能使用静态语句块，但仍然有变量初始化的赋值操作，因此接口与类一样都会生成 <clinit>()方法。但接口与类不同的是，执行接口的<clinit>()方法不需要先执行父接口的<clinit>()方法， 因为只有当父接口中定义的变量被使用时，父接口才会被初始化。
> Java虚拟机必须保证一个类的<clinit>()方法在多线程环境中被正确地加锁同步，如果多个线程同时去初始化一个类，那么只会有其中一个线程去执行这个类的<clinit>()方法，其他线程都需要阻塞等待，直到活动线程执行完毕<clinit>()方法。


#### 类加载器

类加载器只用于实现类的加载动作，但作用却不仅于此，对于任意一个类，都必须由加载它的类加载器和这个类本身一起共同确立其在Java虚拟机中的唯一性，每一个类加载器都拥有一个独立的类名称空间。

比较两个类是否“相等”，只有在这两个类是由同一个类加载器加载的前提下才有意义。否则，即使两个类是来源于同一个Class文件，被同一个Java虚拟机加载，只要加载它们的类加载器不同，那这两个类就必不相等。

站在Java虚拟机的角度来看，只存在两种不同的类加载器：一种是启动类加载器，这个类加载器由C++语言实现，是虚拟机自身的一部分；另外一种是其他所有的类加载器，这些类加载器都由Java语言实现，独立存在于虚拟机外部，并且全都继承自抽象类`java.lang.ClassLoader`。

自JDK1.2开始，Java一直保持三层类加载器、双亲委派机制的类加载架构。

**三层类加载器**：
1. 启动类加载器：这个类加载器负责加载存放在`<JAVA_HOME>\lib`目录，或者被`-Xbootclasspath`参数所指定的路径中存放的，而且是Java虚拟机能够识别的(按照文件名识别，如rt .jar、t ools.jar，名字不符合的类库即使放在lib目录中也不会被加载)类库加载到虚拟机的内存中。启动类加载器无法被Java程序直接引用，用户在编写自定义类加载器时，如果需要把加载请求委派给引导类加载器去处理，那直接使用null代替即可。
2. 扩展类加载器：这个类加载器是在类sun.misc.Launcher$ExtClassLoader中以Java代码的形式实现的。它负责加载<JAVA_HOME>\lib\ext目录中，或者被java.ext.dirs系统变量所指定的路径中所有的类库。根据“扩展类加载器”这个名称，就可以推断出这是一种Java系统类库的扩展机制，JDK的开发团队允许用户将具有通用性的类库放置在ext目录里以扩展Java SE的功能，在JDK 9之后，这种扩展机制被模块化带来的天然的扩展能力所取代。由于扩展类加载器是由Java代码实现的，开发者可以直接在程序中使用扩展类加载器来加载Class文件。
3. 应用程序类加载器：这个类加载器由sun.misc.Launcher$AppClassLoader来实现。由于应用程序类加载器是ClassLoader类中的getSystem- ClassLoader()方法的返回值，所以有些场合中也称它为“系统类加载器”。它负责加载用户类路径上所有的类库，开发者同样可以直接在代码中使用这个类加载器。如果应用程序中没有自定义过自己的类加载器，一般情况下这个就是程序中默认的类加载器。

**双亲委派机制**

![](16465597719934.jpg)
双亲委派模型要求除了顶层的启动类加载器外，其余的类加载器都应有自己的父类加载器。不过这里类加载器之间的父子关系一般不是以继承的关系来实现的，而是通常使用组合关系来复用父加载器的代码。
双亲委派模型的工作过程是:如果一个类加载器收到了类加载的请求，它首先不会自己去尝试加载这个类，而是把这个请求委派给父类加载器去完成，每一个层次的类加载器都是如此，因此所有的加载请求最终都应该传送到最顶层的启动类加载器中，只有当父加载器反馈自己无法完成这个加载请求(它的搜索范围中没有找到所需的类)时，子加载器才会尝试自己去完成加载。


## 第八章 虚拟机字节码执行引擎

### 运行时栈帧结构


## FAQ

泛型擦除