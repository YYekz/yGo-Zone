# 30题：包含 min 函数的栈

> 注意：本题与主站 155 题相同：https://leetcode-cn.com/problems/min-stack/

**难度**：简单

**关键词**：栈、设计

定义栈的数据结构，请在该类型中实现一个能够得到栈的最小元素的 min 函数在该栈中，调用 min、push 及 pop 的时间复杂度都是 O(1)。

示例:
MinStack minStack = new MinStack();
minStack.push(-2);
minStack.push(0);
minStack.push(-3);
minStack.min();   --> 返回 -3.
minStack.pop();
minStack.top();      --> 返回 0.
minStack.min();   --> 返回 -2.

提示：
各函数的调用总次数不超过 20000 次

### 解题思路

普通栈的 push() 和 pop() 函数的复杂度为 ``O(1)``；而获取栈最小值 min() 函数需要遍历整个栈，复杂度为 ``O(N)``。

本题难点： 将 min() 函数复杂度降为 ``O(1)`` 。可借助辅助栈实现：

数据栈 A ： 栈 A 用于存储所有元素，保证入栈 push() 函数、出栈 pop() 函数、获取栈顶 top() 函数的正常逻辑。
辅助栈 B ： 栈 B 中存储栈 A 中所有 非严格降序 元素的子序列，则栈 A 中的最小元素始终对应栈 B 的栈顶元素。此时， min() 函数只需返回栈 B 的栈顶元素即可。

因此，只需设法维护好 栈 B 的元素，使其保持是栈 A 的非严格降序元素的子序列，即可实现 min() 函数的 O(1)O(1) 复杂度。
![min算法](https://pic.leetcode-cn.com/1599880866-aLaPYz-Picture1.png)

> 来源：https://leetcode-cn.com/leetbook/read/illustration-of-algorithm/50bp33/

### 代码

```java
class MinStack {

        private Stack<Integer> A, B;

        /**
         * initialize your data structure here.
         */
        public MinStack() {
            A = new Stack<Integer>();
            B = new Stack<Integer>();
        }

        public void push(int x) {
            if (B.isEmpty() || x <= B.peek()) B.push(x);
            A.push(x);
        }

        public void pop() {
        //这里需要注意使用equals方法，java内部做优化，有做int型常量池，所以 == 会出错
            if (A.pop().equals(B.peek())) {
                B.pop();
            }
        }

        public int top() {
            return A.peek();
        }

        public int min() {
            return B.peek();
        }
    }

 ```
