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

