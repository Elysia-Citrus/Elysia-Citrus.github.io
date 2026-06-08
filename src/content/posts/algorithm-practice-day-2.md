---
title: 算法练习Day 2：链表与哈希表
published: 2026-06-03
description: 记录链表操作与哈希表查找的学习过程。
image: ""
tags: [数据结构, 算法, 链表, 哈希表]
category: 算法练习
draft: false
lang: zh-CN
---

## 今日目标

今天练习链表的基础操作，并理解哈希表如何使用额外空间换取更快的查找速度。

链表中的节点不需要连续存储，插入和删除节点时通常只需要修改指针；哈希表则通过键快速定位对应的值，适合处理查找、计数和去重问题。

## 示例：判断链表是否存在环

使用快慢指针遍历链表。如果链表存在环，移动速度更快的指针最终会追上慢指针。

```ts
interface ListNode {
	value: number;
	next: ListNode | null;
}

function hasCycle(head: ListNode | null): boolean {
	let slow = head;
	let fast = head;

	while (fast?.next) {
		slow = slow?.next ?? null;
		fast = fast.next.next;

		if (slow === fast) {
			return true;
		}
	}

	return false;
}
```

该方法只遍历链表，时间复杂度为 `O(n)`，并且不需要额外集合，空间复杂度为 `O(1)`。

## 哈希表的常见用途

- 记录元素是否出现过。
- 统计每个元素的出现次数。
- 建立键与数据之间的快速映射。
- 使用空间换取更低的查找时间复杂度。

## 学习总结

链表问题需要重点关注节点之间的连接关系，而哈希表适合快速回答“是否存在”和“出现多少次”。面对一道题时，可以先判断是否需要保留遍历历史，再决定是否引入哈希表。
