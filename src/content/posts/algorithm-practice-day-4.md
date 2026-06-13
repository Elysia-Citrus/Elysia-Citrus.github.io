---
title: <题外话>算法练习 Day 4：链表
published: 2026-06-13
description: 两两交换节点、删除倒数第N个节点、链表相交、环形链表
image: ""
tags: [数据结构]
category: 算法练习
draft: false
lang: zh-CN
---

## 今日头条

依旧学习链表的相关操作，今天主要做了四道题：两两交换链表中的节点、删除链表的倒数第N个节点、链表相交、环形链表。

链表的增删改查本质都是基于修改 `next` 指针。其中有一些巧思需要注意，例如交换节点时指针更新的顺序，以及快慢指针在环形链表和倒数节点问题中的妙用。

虚拟头节点（dummy）是链表题中的一个技巧，它可以统一处理头节点的特殊情况，避免额外的边界判断。在执行增删改操作时很好用。

总的来说，抛开环形链表的那个相遇问题的证明，有关链表的主要操作增删改查都是基于next指针，其中有一些巧思例如交换节点，顺序需要留意，其他倒是跟日常数据结构的考点没区别。

## 链表语法回顾

链表节点的基本结构：

```cpp
struct ListNode {
    int val;
    ListNode* next;

    ListNode(): val(0), next(nullptr){}
    ListNode(int x): val(x), next(nullptr){}
    ListNode(int x, ListNode* p): val(x), next(p){}
};
```

构建链表和打印链表的辅助函数：

```cpp
ListNode* buildList(const vector<int>& nums){
    if (nums.empty()) {
        return nullptr;
    }

    ListNode* head = new ListNode(nums[0]);
    ListNode* cur = head;

    for (int i = 1; i < nums.size(); i++) {
        cur->next = new ListNode(nums[i]);
        cur = cur->next;
    }

    return head;
}

void printList(ListNode* head){
    ListNode* cur = head;

    while (cur != nullptr) {
        cout << cur->val << " ";
        cur = cur->next;
    }

    cout << endl;
}
```

几个关于指针使用的经验：

1. **何时创建 `temp`**：当修改 `cur->next` 后会丢失对原节点的引用时，需要先用 `temp` 保存，以便后续释放内存。
2. **何时创建 `cur`**：遍历链表时需要游标指针 `cur` 指向当前节点。通常初始指向虚拟头节点，这样删除操作可以直接修改 `cur->next`。
3. **何时创建 `dummy`**：虚拟头节点可以简化插入和删除操作，避免处理头节点的特殊情况（如删除头节点或插入到空链表）。

## 两两交换链表中的节点

题目要求将链表中相邻的两个节点两两交换，例如 `1→2→3→4→5` 变成 `2→1→4→3→5`。

这道题的难点在于交换节点时指针更新的顺序。如果顺序不对，会丢失节点引用导致链表断裂。

设 `cur` 为当前指针（初始指向 dummy），`node1 = cur->next`，`node2 = cur->next->next`：

```
交换前: cur → node1 → node2 → node2->next
交换后: cur → node2 → node1 → node2->next
```

更新 `next` 指针的顺序为：

1. `cur->next = node2` —— 当前指针指向 node2
2. `node1->next = node2->next` —— node1 跳到 node2 的下一个节点
3. `node2->next = node1` —— node2 指向 node1，完成交换
4. `cur = node1` —— 更新 cur 到下一对的前一个位置（此时 node1 已经在 node2 原来的 `next` 位置了）

最后的 `cur = node1` 很有意思：因为交换后 node1 已经移到了 node2 之后，所以它正好位于下一对待交换节点的前驱位置。

Belike：

![一、二，！](/illustrations/a_partice_day4.png)


循环条件 `while(cur->next && cur->next->next)` 必须**先判断 `cur->next`**，否则当链表长度为奇数时，`cur->next->next` 会访问空指针导致崩溃。

代码：

```cpp
ListNode* swapPairs(ListNode* head){
    ListNode* dummy = new ListNode(0, head);
    ListNode* cur = dummy;

    while(cur->next && cur->next->next){
        ListNode* node1 = cur->next;
        ListNode* node2 = cur->next->next;

        cur->next = node2;
        node1->next = node2->next;
        node2->next = node1;

        cur = node1;
    }

    return dummy->next;
}
```

## 删除链表的倒数第N个节点

题目要求在单向链表中删除倒数第 n 个节点，并返回链表头。

单向链表无法从尾部往回走，所以核心思路是使用**快慢指针**：

1. 快指针 `fast` 先走 n 步
2. 然后快慢指针一起走，直到 `fast` 到达链表末尾
3. 此时慢指针正好在待删除节点的前驱位置

为什么这可行？因为快指针先走了 n 步，当它走到尾部时，慢指针落后 n 步，正好在倒数第 n 个节点的前一个位置。

代码：

```cpp
ListNode* removeNthFromEnd(ListNode* head, int n){
    ListNode dummy(0, head);
    ListNode* cur = &dummy;
    ListNode* behind = &dummy;

    for(int i = 0; i < n; i++){
        cur = cur->next;
    }

    while(cur->next){
        cur = cur->next;
        behind = behind->next;
    }

    ListNode* temp = behind->next;
    behind->next = temp->next;
    delete temp;

    return dummy.next;
}
```

注意：快指针先走 n 步后，循环条件用的是 `while(cur->next)` 而非 `while(cur)`，这样 `behind` 停在待删除节点的前驱，方便执行删除操作。

## 链表相交

题目要求找到两个单链表的相交起始节点（如果相交的话）。

一个容易混淆的点：判断相交的标准是两个指针是否**指向同一个节点地址**，而不是节点的值是否相同。

解法思路：让两个指针走**相同的总路程**。

- A指针先遍历A链表，走完后再遍历B链表，总路程 = `len(A) + len(B)`
- B指针先遍历B链表，走完后再遍历A链表，总路程 = `len(B) + len(A)`

如果两个链表相交，两指针走到相交点之前走过的路程相同，因此在相交点必然相遇。如果两指针同时走到 `nullptr`，则说明两链表不相交。

代码：

```cpp
ListNode* getIntersectionNode(ListNode* headA, ListNode* headB){
    ListNode* A = headA;
    ListNode* B = headB;

    while(A != B){
        if(A){
            A = A->next;
        }else{
            A = headB;
        }

        if(B){
            B = B->next;
        }else{
            B = headA;
        }
    }
    return A;
}
```

这个方法的巧妙之处在于：不需要事先计算两链表的长度差，通过让两指针走相同的总路程自然对齐。

## 环形链表

题目要求判断链表中是否存在环，若存在则返回环的入口节点。

### 方法一：哈希表

最直观的做法，遍历每个节点并用哈希表记录已访问的节点地址，第一个重复访问的节点就是环的入口。

```cpp
ListNode* detectCycle(ListNode* head){
    ListNode* cur = head;
    unordered_set<ListNode*> visited;

    while(cur != nullptr){
        if(visited.count(cur)){
            return cur;
        }
        visited.insert(cur);
        cur = cur->next;
    }
    return nullptr;
}
```

虽然能通过，但内存和时间表现都不是最优。

### 方法二：快慢指针（Floyd 判圈法）

这一题一开始没什么思路，索性用哈希表做的。虽然力扣通过了，但是内存和用时都排得很靠后。

但是这一题有一个数学联立所产生的小妙招。用快慢指针做，先在环内让二者相遇，然后在环外开辟一个新指针，可以证明的是，新指针走到环的所需步数就是慢指针从相遇点走回入口的步数。

设 ：
- x = 从head 到环入口的距离；
- y = 从环入口到第一次相遇点的距离；
- z = 从第一次相遇点再走回环的距离；

环的总长度 = y + z

快慢指针第一次相遇时，慢指针走了x+y步，快指针走了x + y + n * (y + z)，n为快指针比慢指针多绕的圈数。

因为快指针的速度是慢指针的2倍，所以快指针走的距离 = 2 * 慢指针走的距离

即: x + y + n(y + z) = 2(x + y)

化简得：x = n(y + z) - y = (n - 1)(y + z) + z

因此，从head到入口的距离x = 从相遇点走到环入口的距离z + 若干整圈环

也是说，如果一个指针从head出发，另一个指针从第一次相遇点出发，它们相遇的点正好是环入口。

代码：

```cpp
ListNode* detectCycle(ListNode* head){
    ListNode* fast = head;
    ListNode* slow = head;

    while(fast && fast->next){
        slow = slow->next;
        fast = fast->next->next;

        if(slow == fast){
            ListNode* ptr = head;

            while(ptr != slow){
                ptr = ptr->next;
                slow = slow->next;
            }

            return ptr;
        }
    }
    return nullptr;
}
```

注意循环条件 `while(fast && fast->next)`：因为快指针每次走两步，必须同时确保 `fast` 和 `fast->next` 都不为空，否则会访问空指针。
