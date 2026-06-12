---
title: <题外话>算法练习 Day 3：链表基础
published: 2026-06-12
description: 链表基础语法、移除元素、设计链表、反转链表
image: ""
tags: [数据结构]
category: 算法练习
draft: false
lang: zh-CN
---

## 今日复盘

复习了链表的基础操作，包括链表的构建与遍历，以及三道经典题目：移除链表元素、设计链表、反转链表。

链表与数组最大的不同在于内存不连续，每个节点通过指针串联。这意味着链表无法像数组一样随机访问，但插入和删除操作只需要修改指针，时间复杂度为 O(1)（不考虑查找成本）。

总的来说还是一般数据结构课程会接触到的内容，很简单，主要对cpp一些语法还不够熟练。

## 链表基础语法

链表节点的定义使用结构体，包含数据域 `val` 和指针域 `next`。为了方便创建节点，通常会提供多个构造函数：

```cpp
struct ListNode {
    int val;
    ListNode* next;

    ListNode(): val(0), next(nullptr){}
    ListNode(int x): val(x), next(nullptr){}
    ListNode(int x, ListNode* p): val(x), next(p){}
};
```

构建链表有两种常见方式：一种是使用虚拟头节点的尾插法，另一种是直接从一个数组元素开始逐个追加。遍历链表则是用一个 `cur` 指针从头走到尾，直到 `cur == nullptr`。

```cpp
ListNode* buildList(const vector<int>& nums){
    ListNode dummy;
    ListNode* tail = &dummy;

    for(int x: nums){
        tail->next = new ListNode(x);
        tail = tail->next;
    }

    return dummy.next;
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

构建链表时使用虚拟头节点 `dummy` 的好处是：不需要判断链表是否为空，也不需要单独处理第一个节点的插入，所有节点的插入逻辑都是统一的——追加到 `tail` 后面即可。

## 移除链表元素

给定一个链表和一个值 `val`，删除链表中所有值等于 `val` 的节点。

核心思路是使用虚拟头节点指向原链表头，然后遍历链表，每当发现 `cur->next->val == val` 时，将 `cur->next` 指向 `cur->next->next`，从而跳过被删除的节点。

```cpp
class Solution{
public:
    ListNode* removeElements(ListNode* head, int val){
        struct ListNode* dummy = new ListNode(0, head);
        struct ListNode* cur = dummy;

        while(cur->next != nullptr){
            if(cur->next->val == val){
                cur->next = cur->next->next;
            }else{
                cur = cur->next;
            }
        }
        return dummy->next;
    }
};
```

这里虚拟头节点 `dummy` 的作用是：如果头节点本身需要被删除，`cur` 从 `dummy` 开始就能统一处理，不需要单独判断 `head` 是否为空或 `head->val == val`。

## 设计链表

实现一个链表类，支持以下操作：获取第 index 个节点的值、在头部插入、在尾部插入、删除第 index 个节点。

主要注意点：
- 虚拟头节点 `dummy` 让头部插入和空链表插入不再特殊。
- 删除节点时，需要先用临时指针 `temp` 保存被删节点的地址，否则修改 `cur->next` 后就无法访问它来释放内存。
- 遍历时用 `cur` 指针从 `dummy` 或 `dummy->next` 开始，具体取决于是否需要访问目标节点的前驱。

```cpp
class LinkedList{
public:
    LinkedList(){
        dummy = new ListNode(0);
        size = 0;
    }

    int get(int index){
        ListNode* cur = dummy->next;
        int i = 0;
        while(cur != nullptr){
            if(i == index) return cur->val;
            cur = cur->next;
            i++;
        }
        return -1;
    }

    int addAtHead(int val){
        ListNode* newNode = new ListNode(val, dummy->next);
        dummy->next = newNode;
        size++;
        return 0;
    }

    int addAtTail(int val){
        ListNode* newNode = new ListNode(val);
        ListNode* cur = dummy;
        while(cur->next != nullptr){
            cur = cur->next;
        }
        cur->next = newNode;
        size++;
        return 0;
    }

    int deleteAtIndex(int index){
        if(index < 0 || index >= size) return -1;

        ListNode* cur = dummy;
        int i = 0;
        while(cur->next != nullptr){
            if(i == index){
                ListNode* temp = cur->next;
                cur->next = cur->next->next;
                delete temp;
                size--;
                return 0;
            }
            cur = cur->next;
            i++;
        }
        return -1;
    }

private:
    ListNode* dummy;
    ListNode* tail;
    int size;
};
```

## 反转链表

给定一个单链表的头节点，将其反转，返回反转后的头节点。

使用迭代法，需要三个指针：`p` 指向前一个节点（初始为 `nullptr`），`cur` 指向当前节点，`nxt` 暂存下一个节点。每一步将 `cur->next` 指回 `p`，然后三个指针整体前移。

```cpp
class Solution{
public:
    ListNode* reverseList(ListNode* head){
        ListNode* p = nullptr;
        ListNode* cur = head;

        while(cur){
            ListNode* nxt = cur->next;
            cur->next = p;
            p = cur;
            cur = nxt;
        }
        return p;
    }
};
```

循环结束时 `cur == nullptr`，`p` 指向原链表的最后一个节点，也就是反转后的新头节点。反转操作的实质是把每个节点的 `next` 指针从指向后继改为指向前驱，因此必须用 `nxt` 暂存后继，避免断链后找不到下一个节点。
