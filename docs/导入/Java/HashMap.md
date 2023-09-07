HashMap中存放一个数组，数组中每个元素是一个**单项链表**。
每个Entry包含四个属性：key、value、hash和单向链表的next。
capacity：当前数组容量，始终保持2^n。每次扩容为当前2倍，默认为16。
loadFactor：负载因子，默认为0.75。
threshold：扩容阈值，也就是loadFactor * capacity。

