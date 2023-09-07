## HashMap
hash表有两种实现方式：**开放地址法** 和 **冲突链表法**。java7 hashmap采用冲突链表法。
### 版本差异
1. 1.7中采用数组+链表，1.8中升级为数组+链表/红黑树（链表长度超过8升级）；
2. 1.7采用的是头插法，1.8采用的是尾插法；（头插法在扩容时会改变链表中元素的原本顺序，在并发场景下导致链表成环的问题，尾插法在扩容时会保持链表元素原本顺序，就不会出现链表成环的情况）
3. 1.7使用Entry来代表每一个hashmap的数据节点，1.8使用node，基本没有区别都是key、value、hash 和 next四个属性（红黑树是TreeNode）；
4. 1.7扩容时需要重新计算hash和索引位置，1.8不需要重新计算hash，使用扩容后容量的&操作来重新计算索引位置；
5. 1.7是先扩容再插入新值，1.8是先插入新值再扩容；
### 实现细节
1. 默认大小为：16，负载因子为0.75，阈值为 负载因子* 大小；
2. size会默认扩充到2的倍数，如传入15，则size为16；若传入17则size为32；
3. entry计算位置：通过key值的hash计算在桶中的位置(hash是通过key的hash值**高16位&低16位**得来)；如果key为空，则在0位；
4. 如果发生扩容，在桶上的元素，新位置通过hash & (newSize-1)得出，（因为size是2的倍数，所以可以保证将hash的高位抹掉，均匀散列）；如果是冲突链表，则 `hash & newSize ==0` 将链表拆分成在新旧两个位置的两个链表；
5. **Fail-Fast机制**：因为hashmap非线程安全，所以也会记录modCount在对hashmap进行操作时，其他线程对结构有修改，会抛出异常`ConcurrentModificationException`；

### 缺点
1. hash冲突，浪费内存；（极端情况hash完全相同，则在一条链上）
2. 并发问题；

### ArrayMap
为了节约HashMap对内存的浪费，基于数组实现，可以存储**任何数据类型**。底层使用**两个数组**：一个存储key的hashcode（**升序**），一个存储key/value。hashCode计算出数组下标index，value数组中2* index为key，2* index+1 是对应value。
查找时使用**二分查找**。删除时如果数据个数为1， 则置为空数组，原数组容量大小为4或者8则会被缓存；如果容量大于8且存储数据个数小于1/3，则需要重新计算容量：数量小于8个则新容量为8，如果大于8，则新容量为新容量为现有数据个数的1.5倍。

### SpaseArray
Android特有的数据结构，存储int为key，object为value的键值对。key使用二分查找法快速定位key所在的下标。

因为int作为key，所以不会出现hash冲突，但依然会有扩容问题。删除元素并不会直接移动数组，而是会将对应值设为DELETE（一个特殊的object），在合适的时机统一处理，减少数组的移动优化性能。

在put时可以明白所有逻辑：
1. 二分查找对应位置；
2. 如果已存在则替换掉旧值；
3. 如果不存在则取反得出要插入的位置；
4. 如果对应的位置已经被删除（值的元素为DELETE），则直接替换；
5. 如果size大于key的长度，且mGarbage为true，则gc一次，且重新计算要插入的下标（因为gc时，数组发生了变化）；
6. 将对应元素插入对应位置，size++；

### 并发对比ConcurrentHashMap
#### 版本差异
1.7之前使用分段锁机制实现（Segment），最大并发度取决于Segment的个数；1.8使用HashMap类似的**数组+链表+红黑树**的方式实现，加锁通过CAS和synchronized 来实现。
#### 实现细节
1. 扩容为：1.5* initialCapacity + 1，然后向上取2的整数倍；
2. put在链上的元素、树化以及数据迁移（transfer）时会使用到synchronized；
3. 数组为空插入在桶上的位置、initTable、数据迁移和扩容时会使用CAS；

### LinkedHashMap
1. LinkedHashMap是HashMap的子类，只是在HashMap的基础上使用双向链表的形式将所有entry连接起来，
	1. 为了保证元素的迭代顺序和插入顺序相同；
	2. 迭代LinkedHashMap不需要像HashMap遍历整个table，只需要遍历header指向的双向链表即可，也就是说LHM只跟entry的个数有关，与table的大小无关；
2. 非线程安全，可以使用Collections.synchronizedMap(LinkedHashMap)包装成同步的；
3. 经典用法就是：可以轻松实现一个采用了**FIFO替换策略**的缓存。LHM中有一个子类方法`removeEldestEntry`，就是告诉map是否删除最早插入的entry，如果方法返回true，元素就会被删除。在每次插入新元素后，LMH会自动询问`removeEldestEntry`，这样只需要在子类重写该方法，当元素超过某个size时，`removeEldestEntry`返回true即可实现FIFO。
## ArrayList

#### 实现细节
1. 传入的只是java泛型提供的语法糖，其实里面存储的都是object，是一个object数组；
2. **自动扩容**：默认大小为10，每次add元素，会先检测长度再添加，超出则扩容，扩容为1.5倍（grow 和 newCapacity）；也可以手动扩容通过(ensureCapacity)。数组扩容的代价是很高的，因此在实际使用时尽量避免数组扩容；
3. **Fail-Fast机制**：通过modCount实现，在面对并发修改操作时，迭代器会很快失败防止；
4. 允许null元素，底层通过数组实现，与Vector的区别是未实现同步；

### CopyOnWriteArrayList
1. 默认为空大小；
2. 采用了写时复制，读写分离；`add`、`remove`、`set`（增、删、改）都先加锁，再操作。

## LinkedList
1. 同时实现了List和Deque接口，既可以当做队列，也可以当做栈（现在首先是ArrayDeque，有着比LinkedList更好的性能）；
2. 通过双向链表实现，每个节点用node实现。通过first 和 last引用分别指向链表的第一个和最后一个元素（当链表为空时，fist=last=null）；

### ArrayDeque
1. 通过Array实现，为了满足在两端插入和删除，该数组是**循环数组**，任何一个点都可以被看做起点或者终点；
2. 非线程安全，不允许放入null,size扩容为2的倍数；
> 循环数组：head指向首端的第一个有效元素，tail指向尾端第一个可以插入元素的空位，因为是循环数组，所以head不一定总等于0，tail也不一定总比head大。

## String

### 实现原理
String为final类。内部使用char数组存储，该数组为final，意味着value数组初始化后不能再引用其他数组，内部也无改变value数组的方法，因此可以确保String不可变。不可变的好处：
1. **可以缓存hash值**。因为String经常作为例如HashMap的key，不可变的特性使得hash值也不可变因此只需要以此计算；
2. **StringPool的需要**。因为只有不可变才能使用StringPool；
3. **安全性**。String作为参数，String的不可变性可以保证参数不变；
4. **线程安全**。

### java8 使用char[] java9使用byte[] 为什么？
最主要的目的是为了**解决字符串的内存占用**，从而减少GC。char类型在JVM中占两个字节使用UTF-8编码也就是说本一个字节可以表示的字符也得占用两个字节。而实际开发中单字节使用率高于双字节。但还是有双字节的，所以从char变为byte不够，还需要一个编码格式，如果是单字节的使用Latin-1编码这样一个基础的jack单词，只用四字节即可以表示。但如果对于中文，则需要使用UTF-16来表示。
那为什么不是UTF-8呢？因为UTF-8使用1-4字节表示字符，那在随机访问的时候，需要从头读取每个字符的长度，才能读取到你想要的字符。那UTF-16使用2-4字节来存储字符，但在java中，String各种操作都是以char为单位的，char是两字节，所以UTF-16在Java世界中，可以视为一个定长编码。

> [Java9为何要将String的底层实现由char[]改成了byte[]?](https://blog.csdn.net/androidstarjack/article/details/124976993)

### intern
intern方法可以保证相同内容的字符串变量引用的是同一内存对象。

### StringBuilder 和 StringBuffer
StringBuilder线程不安全，StringBuffer线程安全（synchronized）。

## 线程池
Worker继承AQS，**为什么使用AQS但没使用重入ReentrantLock？** 为的就是实现不可重入的特性去反映线程当前的执行状态（lock一旦获取了独占锁，表示当前线程正在执行任务）。

[Java线程池实现原理及其在美团业务中的实践 - 美团技术团队](https://tech.meituan.com/2020/04/02/java-pooling-pratice-in-meituan.html)
[Fetching Title#zqpj](https://www.javadoop.com/post/java-thread-pool)
