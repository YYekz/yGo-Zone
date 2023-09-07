# Java集合
从这几个方面入手：
1. 底层实现数据结构；
2. 基础容量与扩容机制；
3. CRUD操作的实现原理，以及复杂度；


## ArrayList

### 底层实现数据结构
ArrayList底层基于数组实现，所以支持随机访问，复杂度为O(1)，但非线程安全。

### 基础容量与扩容机制
如果没有传入size，则默认为空，在第一次`add`类型操作的时候，对数组扩容为10（若`add`元素>10则为对应size）。

```java
//有些虚拟机会在数组中保留一些头字段
private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        int newCapacity = oldCapacity + (oldCapacity >> 1); //一次扩容1.5倍
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0) 
            newCapacity = hugeCapacity(minCapacity); 
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
    
    private static int hugeCapacity(int minCapacity) {
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError();
        return (minCapacity > MAX_ARRAY_SIZE) ? //如果指定为比MAX_ARRAY_SIZE要大，则为对应size
            Integer.MAX_VALUE :
            MAX_ARRAY_SIZE;
    }
```
总结一下，ArrayList的扩容机制为：当前size为A，
- 一次扩容为之前的1.5倍大小为 1.5A；
- 若传入的size大于扩容1.5A则扩容容量为传入的size；
- 若传入的size 大于 MAX_ARRAY_SIZE但小于Integer.MAX_VALUE，则为MAX_VALUE；
- 若传入的size 大于Integer.MAX_VALUE，则报错；
- 若传入的size 大于小与MAX_ARRAY_SIZE，则为MAX_ARRAY_SIZE；

### CRUD操作的实现原理，以及复杂度


### 其他
Fail-Fast机制: ArrayList也采用了快速失败的机制，通过记录modCount参数来实现。在面对并发的修改时，迭代器很快就会完全失败，而不是冒着在将来某个不确定时间发生任意不确定行为的风险。 


## HashMap

HashMap就是使用哈希表来存储的。哈希表为解决冲突，可以采用开放地址法和链地址法等来解决问题，Java中HashMap采用了链地址法。

### tableSIzeFor 方法
```java
    static final int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
```
该方法的目的就是将传入的容量大小设置为2的指数倍。
