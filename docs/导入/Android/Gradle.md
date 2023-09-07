# Gradle

![](media/16383464200315/16383464251914.jpg)

## 编译速度

- **编译时间**。把 Java 或者 Kotlin 代码编译为“.class“文件，然后通过 dx 编译为 Dex 文件。对于增量编译，我们希望编译尽可能少的代码和资源，最理想情况是只编译变化的部分。但是由于代码之间的依赖，大部分情况这并不可行。这个时候我们只能退而求其次，希望编译更少的模块。Android Plugin 3.0使用 Implementation 代替 Compile，正是为了优化依赖关系。
- **安装时间**。我们要先经过签名校验，校验成功后会有一大堆的文件拷贝工作，例如 APK 文件、Library 文件、Dex 文件等。之后我们还需要编译 Odex 文件，这个过程特别是在 Android 5.0 和 6.0 会非常耗时。对于增量编译，最好的优化是直接应用新的代码，无需重新安装新的 APK。

### 执行流程
1. 解析settings.gradle来获取模块信息，这是初始化阶段；
2. 配置每个模块，生成build.gradle，配置的时候并不会执行task；
3. 配置完成后，有一个重要回调 project.afterEvaluate，表示所有模块都已经配置完成，可以准备执行task了；
4. 执行指定的task以及其依赖的task；

### app module下的dependencies 和 root buildscript的dependencies的区别？
不同于 根项目 buildscript 中的 dependencies 是用来配置我们 Gradle 工程的插件依赖的，而 app moudle 下的 dependencies 是用来为应用程序添加第三方依赖的。


### Task

每个 task 都会经历 初始化、配置、执行 这一套完整的生命周期流程。

## 组件依赖关键字

**implementation**：只在组件内部使用。举个例子，A implementation B，:app 依赖了 A，但:app无法使用B。如果想使用可以用api的方式去依赖。(implementation 编译速度快于 api，所以如果不需要依赖透传，则推荐使用implementation)

**api**：依赖组件透传。举个例子，A api B，:app依赖A，也可以使用B。

**compileOnly**：只在编译时有效，不会参与打包，使用该方式依赖自己用的组件，防止冲突。

**testImplementation**、**debugImplementation**、**releaseComplementation**：仅针对对应模式的编译和最终版本包生效。


## 好文
[从零开始打造一个，用gradle配置即可执行的Hook库 - 掘金](https://juejin.cn/post/7100086790639337508)