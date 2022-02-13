# 09题：用两个栈实现队列

**难度**：简单
**关键词**：栈、队列、设计

用两个栈实现一个队列。队列的声明如下，请实现它的两个函数 appendTail 和 deleteHead ，分别完成在队列尾部插入整数和在队列头部删除整数的功能。(若队列中没有元素，deleteHead 操作返回 -1 )

示例 1：
输入：
["CQueue","appendTail","deleteHead","deleteHead"]
[[],[3],[],[]]
输出：[null,null,3,-1]

示例 2：
输入：
["CQueue","deleteHead","appendTail","appendTail","deleteHead","deleteHead"]
[[],[],[5],[2],[],[]]
输出：[null,-1,null,null,5,2]

提示：
1 <= values <= 10000
最多会对 appendTail、deleteHead 进行 10000 次调用

### 解题思路

栈实现队列的出队操作效率低下：栈底元素（对应队首元素）无法直接删除，需要将上方所有元素出栈。

列表倒序操作可使用双栈实现：设有含三个元素的栈 A = [1,2,3] 和空栈 B = [] 。若循环执行 A 元素出栈并添加入栈 B ，直到栈 A 为空，则 A = [] , B = [3,2,1] ，即栈 B 元素为栈 A 元素倒序。

利用栈 B 删除队首元素：倒序后，B 执行出栈则相当于删除了 A 的栈底元素，即对应队首元素。

题目要求实现 加入队尾appendTail() 和 删除队首deleteHead() 两个函数的正常工作。因此，可以设计栈 A 用于加入队尾操作，栈 B 用于将元素倒序，从而实现删除队首元素。

> 来源：https://leetcode-cn.com/leetbook/read/illustration-of-algorithm/5dq0t5/

### 代码

```java
class CQueue {

        private Stack<Integer> firstStack, secondStack;

        public CQueue() {
            firstStack = new Stack<Integer>();
            secondStack = new Stack<Integer>();
        }

        public void appendTail(int value) {
            firstStack.push(value);
        }

        public int deleteHead() {

            while (!firstStack.empty()) {
                secondStack.push(firstStack.pop());
            }
            int result = secondStack.empty() ? -1 : secondStack.pop();
            while (!secondStack.empty()) {
                firstStack.push(secondStack.pop());
            }
            return result;
        }
}
```