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

备左则右寡，备右则左寡，无所不备则无所不寡。
学习也要2，8分。然后白板回忆时要内容全，这时候直接对照书就行了。


#### 递归删

        // 递归处理，如果当前节点是待删除点，直接使用新节点指向下一个节点。
        if (head->val == val) {
            ListNode* newHead = removeElements(head->next, val);
            delete head;
            return newHead;
        }
