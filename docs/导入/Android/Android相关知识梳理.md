# Android文件格式
## .dex文件
**.dex文件**是android虚拟机可执行字节码文件。我们知道java文件通过`javac`编译生成class文件，dx工具会将class文件合并处理最终生成dex文件。

dex文件分为四大部分：DEX文件头、索引结构区、data数据区、静态链接数据区。在文件中所有的代码和数据都是放在data数据区的，索引结构区中存放的是data中各种数据对应的偏移量和索引。
![[Pasted image 20221029113316.png|500]]
## odex文件
ODEX是优化过后的DEX文件。
Android5.0之前，Android采用的是JIT（just-in-time）编译，边执行边编译。为了增加程序执行的效率，在apk第一次安装的时候将程序的dex文件优化生成odex文件，存放在`/data/dalvik-cache/`目录下，下次apk运行时直接加载这个目录下的odex文件（优化基于系统dalvik虚拟机版本，不同版本odex文件不兼容），避免重复验证以提高执行效率，加速apk启动时间。

## oat文件
Android5.0之后，Android采用的是AOT（ahead-of-time）事前编译，就是在程序运行前编译。oat是ART虚拟机的运行文件，是ELF格式的二进制文件，包含**DEX文件和编译的本地机器指令**，oat文件包含dex文件，因此比odex文件占用空间大。程序在首次安装的时候，dex2oat会把.dex文件翻译成本地机器指令，生成ELF格式的oat文件，并存放在`/data/dalvik-cache/`或者`/data/app/package-name`/目录下，ART加载oat文件不需要经过处理就可以直接运行，在编译时就从字节码转换成机器码，因此运行速度比odex更快。

在Android5.0之后，oat文件的后缀还是odex，但是已经不是Android5.0之前的文件格式，而是ELF格式封装的本地机器码（可以认为是oat在dex上加了一层壳），可以从oat文件中提取出dex文件。
![[elf格式的oat文件示意.png|500]]
## vdex文件
Android8.0之前，dex文件嵌入到oat文件本身中，在8.0之后，dex2oat将.dex文件优化生成两个文件：oat文件(.odex)和vdex文件(.vdex)，其中包含apk的未压缩dex代码，以及一些旨在加快验证速度的元数据。odex文件中包含了本机代码的oat，vex文件包含了原始的dex文件副本。

## art文件
ART虚拟机在执行dex文件时，需要将dex文件中使用的类、字符串等信息转换为自定义的结构。art文件就是保存apk中使用的一些类、字符串信息等信息的ART内部表示，以加快apk程序启动速度。

## ELF文件
上问提到的ELF格式的二进制文件，那ELF文件是什么呢？
ELF是Executable and Linking Format 的缩写，是android平台上通用的二进制文件格式。在Android的NDK开发中，几乎都是和ELF打交道。
比如：
- c/c++文件编译得到的.o(.obj)文件就是ELF格式的文件；
- 动态库(.so)文件、可执行文件也是ELF格式的文件；
从ELF的全程中可以看出有executable和linking两种重要特性。
1. executable：表示可执行的。ELF文件将参与程序的执行过程，包括二进制程序的运行以及动态库.so文件加载。
2. linking：表示可连接的。ELF文件参与编译链接过程。

## 文件加载
在Android中我们想加在一个so库会使用，`System.load`或者`System.loadLibrary`。两个方法的区别其实就是`loadLibrary`只能加载jniLibs目录下的so，`load`可以加载任意路径下的so，两种方式最终都是通过调用Android底层的dlopen来打开so。
执行的大致流程为：
1. 先读取so文件的.init_array段；
2. 执行JNI_OnLoad函数（JNI_OnLoad是.so文件的初始化函数）；
3. 最后调用具体的native方法；

## D8编译器
AS 3.1版本引入D8编译器为默认的DEX文件字节码编译器。可以通过在`gradle.properties`中添加`android.enableD8=true`开启。
**特点**：
1. 编译更快、时间更短；
2. DEX编译时占用内容更小；
3. .dex文件更小；
4. D8编译的.dex文件拥有相同或更好的运行时性能；

## R8工具
AS3.2 引入R8 作为替代ProGuard的工具，用于代码压缩和混淆，通过在`gradle.properties`中增加`android.enableR8=true`开启。
### 变更
在AS3.4 ，R8把脱糖、压缩、混淆、优化和dex都合成了一步执行。（如果再`build.gradle`中配置了`useProguard=false`则不管是否开启R8编译都会使用R8压缩代码）

## 脱糖
在上面我们说的文件转化的过程中，有一步叫做脱糖，目的就是将Java 8 中的特性在Android平台上适配，即在编译阶段将在语法层面一些底层字节码不支持的特性转换为基础的字节码结构，(比如 `List` 上的泛型脱糖后在字节码层面实际为 `Object`，让java8中的lambda表达式转化为Java7)； `Android` 工具链对 `Java8` 语法特性脱糖的过程可谓丰富多彩，当然他们的最终目的是一致的：使新的语法可以在所有的设备上运行。
如果想关闭/禁止脱糖则可以在`gradle.properties`中加入。
```groovy
android.enableIncrementalDesugaring=false
android.enableDesugar=false
```


> [R8详解](https://mp.weixin.qq.com/s/54Gyq6g4X1NAcmNsZtJpeA)
## 打包

## aab


## 相关文章
[android逆向基础之文件格式](https://mp.weixin.qq.com/s/3-GSGGtJJsO9dPhFDx7_yQ)
[Android D8 编译器 和 R8 工具 - 掘金](https://juejin.cn/post/6973089862278725640)
[为方法数超过 64K 的应用启用 MultiDex  |  Android 开发者  |  Android Developers](https://developer.android.com/studio/build/multidex.html)