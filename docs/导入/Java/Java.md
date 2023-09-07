# Java

## 基础

### String
StringBuilder线程不安全；StringBuffer线程安全（使用`synchronized`进行同步）。

#### inter()
使用 String.intern() 可以保证相同内容的字符串变量引用同一的内存对象。
inter()是先将引用对象放到字符串常量池中，然后返回这个对象引用。
```
        String s1 = new String("aaa");
        String s2 = new String("aaa");
        System.out.println(s1 == s2);  //false  两个不同对象
        String s3 = s1.intern();
        System.out.println(s3 == s1);  //false  两个不同对象
        System.out.println(s3 == s1.intern()); //true 
        String s4 = "aaa";
        String s5 = "aaa";
        System.out.println(s4 == s5);  //true 这种新建方式会自动放入常量池中
        System.out.println(s4 == s2);  //false 
        System.out.println(s4 == s3);  //true 
```

### 重写与重载 
1. 重写(Override) 存在于继承体系中，指子类实现了一个与父类在方法声明上完全相同的一个方法。 为了满足里式替换原则，重写有以下两个限制: 子类方法的访问权限必须大于等于父类方法； 子类方法的返回类型必须是父类方法返回类型或为其子类型。 使用 `@Override` 注解，可以让编译器帮忙检查是否满足上面的两个限制条件。
2. 重载(Overload) 存在于同一个类中，指一个方法与已经存在的方法名称上相同，但是参数类型、个数、顺序至少有一个不同。 应该注意的是，返回值不同，其它都相同不算是重载。 

### clone()
clone() 是 Object 的 protected 方法，它不是 public，一个类不显式去重写 clone()，其它类就不能直接去调用该类实例的 clone() 方法。

应该注意的是，clone() 方法并不是 Cloneable 接口的方法，而是 Object 的一个 protected 方法。Cloneable 接口只是规定，如果一个类没有实现 Cloneable 接口又调用了 clone() 方法，就会抛出 CloneNotSupportedException。
```java
    protected Object clone() throws CloneNotSupportedException {
        if (!(this instanceof Cloneable)) {
            throw new CloneNotSupportedException("Class " + getClass().getName() +
                                                 " doesn't implement Cloneable");
        }

        return internalClone();
    }
```

### 静态方法
静态方法在类加载的时候就存在了，它不依赖于任何实例。所以静态方法必须有实现，也就是说它不能是抽象方法(abstract)。

### 泛型

泛型的类型擦除**原则**： 
1. 消除类型参数声明，即删除<>及其包围的部分。 
2. 根据类型参数的上下界推断并替换所有的类型参数为原生态类型：如果类型参数是无限制通配符或没有上下界限定则替换为Object，如果存在上下界限定则根据子类替换原则取类型参数的最左边限定类型（即父类）。
3. 为了保证类型安全，必要时插入强制类型转换代码。
4. 自动产生“桥接方法”以保证擦除类型后的代码仍然具有泛型的“多态性”。

在调用**泛型方法**时，可以指定泛型，也可以不指定泛型: 
1. 在不指定泛型的情况下，泛型变量的类型为该方法中的几种类型的同一父类的最小级，直到Object ;
2. 在指定泛型的情况下，该方法的几种类型必须是该泛型的实例的类型或者其子类 。

Java编译器是通过先检查代码中泛型的类型，然后在进行类型擦除，再进行编译。
类型检查就是针对引用的，谁是一个引用，用这个引用调用泛型方法，就会对这个引用调用的方法进行类型检测，而无关它真正引用的对象。

泛型的出现就是为了消灭 ClassCastException。
**协变**： `< ? extends T>实现泛型协变`
**逆变**：`<? super T>实现泛型逆变`
**PECS原则**：（Producer extends ,consumers super）
- ``<? extends E>` 子类限定通配符：从泛型类读取类型 T 的数据，并且不能写入，用于生产者环境（Producer）；
- ``<? super E>`父类限定通配符： 从集合中写入类型 T 的数据，并且不需要读取，用于消费者场景（Consumer）；
- 如果既要存数据又要取数据，那么通配符无法满足需求。

> 推荐文章：[Java泛型的协变与逆变](https://blog.csdn.net/m0_37796683/article/details/108584499)


泛型类中的静态方法和静态变量不可以使用泛型类所声明的泛型类型参数。
因为泛型类中的泛型参数的实例化是在定义对象的时候指定的，而静态变量和静态方法不需要使用对象来调用。对象都没有创建，如何确定这个泛型参数是何种类型，所以当然是错误的。

通过反射获取泛型的具体参数类型。

### 注解

注解@interface 是一个实现了Annotation接口的 接口， 然后在调用getDeclaredAnnotations()方法的时候，返回一个代理$Proxy对象，这个是使用jdk动态代理创建，使用Proxy的newProxyInstance方法时候，传入接口 和InvocationHandler的一个实例(也就是 AnotationInvocationHandler ) ，最后返回一个代理实例。

期间，在创建代理对象之前，解析注解时候 从该注解类的常量池中取出注解的信息，包括之前写到注解中的参数，然后将这些信息在创建 AnnotationInvocationHandler时候 ，传入进去 作为构造函数的参数。

> 推荐文章：[java注解原理](https://blog.csdn.net/qq_20009015/article/details/106038023)

#### AOP

### 反射

Class类，Class类也是一个实实在在的类，存在于JDK的java.lang包中。Class类的实例表示java应用运行时的类(class ans enum)或接口(interface and annotation)（每个java类运行时都在JVM里表现为一个class对象，可通过类名.class、类型.getClass()、Class.forName("类名")等方法获取class对象）。数组同样也被映射为class 对象的一个类，所有具有相同元素类型和维数的数组都共享该 Class 对象。基本类型boolean，byte，char，short，int，long，float，double和关键字void同样表现为 class  对象。£

## jvm

### 指令

`invokestatic`：调用静态方法
`invokespecial`：调用实例构造方法，私有方法和父类方法
`invokevirtual`：调用所有的虚方法（抽象类）
`invokeinterface`：调用接口方法，会在运行时确定一个实现该接口的对象

### 异常表
那么异常表用在什么时候呢?
答案是异常发生的时候，当一个异常发生时 
1. JVM会在当前出现异常的方法中，查找异常表，是否有合适的处理者来处理
2. 如果当前方法异常表不为空，并且异常符合处理者的from和to节点，并且type也匹配，则JVM调用位于target的调用者来处理。 
3. 如果上一条未找到合理的处理者，则继续查找异常表中的剩余条目 
4. 如果当前方法的异常表无法处理，则向上查找（弹栈处理）刚刚调用该方法的调用处，并重复上面的操作。
5. 如果所有的栈帧被弹出，仍然没有处理，则抛给当前的Thread，Thread则会终止。 
6. 如果当前Thread为最后一个非守护线程，且未处理异常，则会导致JVM终止运行。 
