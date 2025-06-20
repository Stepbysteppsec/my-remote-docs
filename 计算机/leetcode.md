### 链表类型
单链表
双链表
循环链表

### 链表存储方式

#### 单链表节点定义
```cpp
struct ListNode{
int data;
ListNode* next;
ListNode(int x):data(x),next(null){} //链表构造函数
};
```

通过自定义构造函数初始化节点
```cpp
ListNode head = new ListNode(0);
```

#### 链表的增删改查

#####  增

```cpp
ListNode node = new ListNode(5);
head->next = node;
```

##### 删
```cpp
// dummy head
	//只要检查cur->next 就行了，保证cur->next->data 不会引起未定义行为。
    while(cur->next != nullptr ){
	    
	    //需要记住要被删除的节点，否则内存泄露。
		    ListNode tmp = cur->next;
		    cur = cur->next->next;
		    delete tmp;
		
    //有可能头节点已经删除，直接返回会引起未定义行为
    head = dummy->next;

```

#### 递归删

        // 递归处理，如果当前节点是待删除点，直接使用新节点指向下一个节点。
        if (head->val == val) {
            ListNode* newHead = removeElements(head->next, val);
            delete head;
            return newHead;
        }
### 在链表最前面插入一个节点
```cpp
//因为使用MyLinkedList 然后插入节点 并不需要返回头节点，头节点应该被隐藏只能通过接口访问。
void adHead(int val){
//这里只是插入一个新节点，所以命名时符合语义 新节点即可
	ListNode newnode = new ListNode(val);
	
}
```
### 链表长度
如果index 为链表长度，那index == _size

```cpp
if(index > size) return ;
if(index < 0 ) return;
while(index--){

}
```

### 链表删除
```cpp
delete tmp;
tmp = nullptr;
_size--;
```
### ListNode* 和 ListNode
1. **内存分配位置不同**：
    - `LinkedNode* newNode = new LinkedNode(val)`：在堆(heap)上分配内存，这块内存在函数结束后仍然存在
    - `LinkedNode newNode(val)`：在栈(stack)上分配内存，函数结束后自动销毁
2. **生命周期差异**：
    - 使用指针创建的节点可以在链表中持久存在，即使创建它的函数已经返回
    - 栈上创建的对象会在离开作用域时自动销毁，这对链表是致命的
3. **链表结构的需要**：
    - 链表是一个动态数据结构，需要在运行时动态添加/删除节点
    - 每个节点的`next`指针需要指向另一个真实存在的节点，如果节点是栈上的局部变量，函数返回后指针会变成悬空指针

如果你使用`LinkedNode newNode(val)`：

1. 这个`newNode`只在当前函数作用域内有效
2. 函数结束后，这个节点会被销毁，但链表仍然持有指向这块已被释放内存的指针
3. 当后续代码试图访问这个节点时，会导致未定义行为(undefined behavior)，很可能引起程序崩溃

这就是为什么在实现持久性数据结构（如链表）时，必须使用动态内存分配（通过`new`关键字）和指针来操作节点的原因。

好的，我再用更详细的方式解释**传统递归法**，并对比你的**尾递归法**，帮助你彻底理解。

---

### 链表反转的两种递归方式
#### 示例链表：
```
1 → 2 → 3 → 4 → NULL
```
目标：反转为 `4 → 3 → 2 → 1 → NULL`。

---

### 方法 1：尾递归（你的方法）
**逻辑：** 像迭代法一样，一步步修改指针方向，用递归代替循环。
```cpp
ListNode* reverse(ListNode* pre, ListNode* cur) {
    if (cur == NULL) return pre;      // 终止条件
    ListNode* temp = cur->next;       // 暂存下一个节点
    cur->next = pre;                  // 反转当前节点
    return reverse(cur, temp);        // 递归处理下一个节点
}
```
**执行过程：**
1. `reverse(NULL, 1)` → 1.next = NULL，递归 `reverse(1, 2)`
2. `reverse(1, 2)` → 2.next = 1，递归 `reverse(2, 3)`
3. `reverse(2, 3)` → 3.next = 2，递归 `reverse(3, 4)`
4. `reverse(3, 4)` → 4.next = 3，递归 `reverse(4, NULL)`
5. `reverse(4, NULL)` → 返回 `4`（新头节点）

**结果：**  
`4 → 3 → 2 → 1 → NULL`

---

### 方法 2：传统递归（分治思想）
**逻辑：** 先让子问题解决剩余部分，再处理当前节点。
```cpp
ListNode* reverseList(ListNode* head) {
    if (head == NULL || head->next == NULL) {
        return head;  // 终止条件：空链表或单节点
    }
    ListNode* newHead = reverseList(head->next); // 先递归反转剩余部分
    head->next->next = head; // 关键：让剩余部分的末尾指向自己
    head->next = NULL;       // 断开自己的原指针
    return newHead;          // 返回新头节点
}
```
**执行过程：**
1. `reverseList(1)`：
   - 递归调用 `reverseList(2)` →
     - 递归调用 `reverseList(3)` →
       - 递归调用 `reverseList(4)` →
         - 4.next == NULL，返回 `4`（终止）
       - 现在：`3.next = 4`，执行 `4.next = 3` 和 `3.next = NULL`，返回 `4`
     - 现在：`2.next = 3`，执行 `3.next = 2` 和 `2.next = NULL`，返回 `4`
   - 现在：`1.next = 2`，执行 `2.next = 1` 和 `1.next = NULL`，返回 `4`

**结果：**  
`4 → 3 → 2 → 1 → NULL`

---

### 关键区别：
| 操作步骤       | 尾递归（你的方法）                          | 传统递归                                   |
|----------------|---------------------------------------------|--------------------------------------------|
| **指针修改时机** | 递归前直接修改（类似迭代）                  | 递归返回后回溯时修改                       |
| **执行顺序**    | 从前往后处理节点                            | 从后往前处理节点                           |
| **代码逻辑**    | `cur->next = pre;` 然后递归                 | 先递归到末尾，再让 `head->next->next = head` |
| **空间复杂度**  | O(n)（栈空间，但可优化为迭代）              | O(n)（必须使用栈）                         |

---

### 为什么传统递归更难理解？
因为它依赖**递归栈的回溯**：
1. 递归会一直深入到链表末尾（`4`），然后逐层回溯。
2. 回溯时，通过 `head->next->next = head` 让后一个节点指向前一个节点（比如让 `4` 指向 `3`）。
3. 最后要断开原指针（`head->next = NULL`），否则会成环。

---

### 建议：
1. **画图辅助**：对每个递归步骤画图，观察指针如何变化。
2. **调试代码**：在递归调用前后打印 `head` 和 `newHead` 的值。
3. **对比学习**：先彻底理解尾递归（你的方法），再过渡到传统递归。

如果还有疑问，可以具体指出哪一步不清楚，我会进一步解释！

