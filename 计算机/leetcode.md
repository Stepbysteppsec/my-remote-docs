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
ListNode * remove(int x,ListNode *head){
	ListNode dummy_head = new ListNode(0);
	dummy_head->next = head;
    ListNode* cur = dummy_head; 
    while(cur->next != nullptr && cur->next->next != nullptr){
	    if(cur->next->data == x){
		    cur = cur->next->next;
		}
	    else cur = cur->next;
    }
    return head;
}
```

