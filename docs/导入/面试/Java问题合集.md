### 静态变量存在方法区还是堆里？
局部变量
方法中的局部变量存在于栈内存。每当程序调用一个方法时，系统都会为该方法建立一个方法栈，其所在方法中声明的变量就放在方法栈中，当方法结束系统会释放方法栈，其对应在该方法中声明的变量随着栈的销毁而结束，这就局部变量只能在方法中有效的原因。

成员变量
对象实例的引用存储在栈内存中，对象实例存储在堆内存中。所以，对象中声明的成员变量存储在堆中。（成员变量不会随着某个方法执行结束而销毁）`isDeveloperPlugin()`

```java
public abstract class Animal {  
  
  
    abstract void speak();  
  
    public void say() {  
        System.out.println("Animal is say");  
        hello();  
    }  
  
    public void hello(){  
        System.out.println("hello");  
    }  
}
```
静态变量
类中的静态变量（被static关键字修饰）存放在 Java 内存区域的方法区。方法区与 Java 堆一样，是各个线程共享的内存区域，它用于存储已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据。虽然Java 虚拟机规范把方法区描述为堆的一个逻辑部分，但是它却有一个别名叫做 Non-Heap（非堆），目的应该是与 Java 堆区分开来。

[Java 变量存储的位置 - 百里浅暮 - 博客园](https://www.cnblogs.com/xfeiyun/p/16863033.html)

### 为什么成员变量可以不赋初始值但局部变量不可以？
因为java在类加载过程中，会对成员变量进行两次初始化，第一次是在准备阶段默认0值，第二次是在初始化阶段，默认为程序员自己设置的值。但局部变量在方法中，方法是基于栈执行的，要进行进栈操作，所以没有初始值编译器就会无法通过，但成员变量有准备阶段的0值兜底。

[java关于局部变量必须初始化赋初值及成员变量不必须该操作的原理浅解析_wjwisme的博客-CSDN博客](https://blog.csdn.net/wjw521wjw521/article/details/79243596)

### init和clinit的区别？

**init和clinit方法执行时机不同**
init是对象构造器方法，也就是说程序在执行new 一个对象的时候会调用这个对象类的constructor方法时才会执行init方法，而clinit是类构造器方法，就是jvm在进行类加载的时候初始化阶段会调用clinit方法。

**init和clinit方法执行目的不同**
init是instance实例构造器，对非静态变量解析初始化，而clinit是class类构造器是对静态变量、静态代码快进行初始化的。

[深入理解jvm--Java中init和clinit区别完全解析_HankingHu的博客-CSDN博客_clinit](https://blog.csdn.net/u013309870/article/details/72975536)
[关于jvm-java中init与clint区别_qq_24904257的博客-CSDN博客](https://blog.csdn.net/qq_24904257/article/details/91607263)

### 基于栈和基于寄存器的区别？
### String、StringBuffer 和 StringBuilder之间的区别？
1. String长度不可变，StringBuilder和StringBuffer长度可变（源码中String使用final char[]，其他俩继承子AbstractStringBuilder使用char[]）；
2. 运行速度上：StringBuilder > StringBuffer > String（StringBuffer有线程安全锁导致的速度低于StringBuilder，String是因为每次"+"是在创建新String对象，创建完后再回收前面无用的String对象）；
3. StringBuilder 线程不安全 和 StringBuffer线程安全；（有`toStringCache`锁字段）

[String、StringBuffer和StringBuilder的区别 - java面试题 - SegmentFault 思否](https://segmentfault.com/a/1190000022038238)

### 什么是双亲委派机制？
### lambda是什么，和匿名内部类有啥区别？

##### 在java中，有两种情况：

> lambda是java8推出的一种语法糖，取代函数接口的一种简写。

1. 当是一个无状态的lambda（当前块作用域内），就能直接生成一个static的方法；
2. 当引用了一个成员变量时，就会生成一个类的实例方法，同时，这种方式会引入ALOAD 0，也就是this指针，如果外层类和lambda生命周期不同步，就会导致内存泄漏；

```java
//状态1
void myFunc(View view){ 
	int a = 1; 
	view.setOnClickListener(v -> { 
	Log.e("hello","123" +a ); 
	}); 
}

//状态2
public String s;
void myFunc(View view){ 
	view.setOnClickListener(v -> { 
	Log.e("hello","123" +s); 
	});
}
```
匿名内部类中，字节码会生成新对象，其实还是新建对象调用对应方法；lambda会在类中生成一个静态方法，使用`invokedynamic`调用。

##### kotlin中
在kotlin中，不管是无状态还是引用了类的成员变量，都会生成一个静态方法，区别是在于当引用了类的成员变量时，会直接在生成方法参数中传入引用的外部变量。

[Lambda - 认识java lambda与kotlin lambda的细微差异 - 掘金](https://juejin.cn/post/7206576861052420157)

## 并发

在先调用unpark，再调用park时，仍能够正确实现同步，不会造成由wait/notify调用顺序不当所引起的阻塞。因此park/unpark相比wait/notify更加的灵活。

### Thread.sleep()和Object.wait()的区别

-   Thread.sleep()不会释放占有的锁，Object.wait()会释放占有的锁；
-   Thread.sleep()必须传入时间，Object.wait()可传可不传，不传表示一直阻塞下去；
-   Thread.sleep()到时间了会自动唤醒，然后继续执行；
-   Object.wait()不带时间的，需要另一个线程使用Object.notify()唤醒；
-   Object.wait()带时间的，假如没有被notify，到时间了会自动唤醒，这时又分好两种情况，一是立即获取到了锁，线程自然会继续执行；二是没有立即获取锁，线程进入同步队列等待获取锁；

其实，他们俩最大的区别就是Thread.sleep()**不会释放锁资源**，Object.wait()**会释放锁资源**。