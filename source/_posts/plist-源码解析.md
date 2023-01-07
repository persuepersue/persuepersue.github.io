---
title: plist 源码解析
date: 2023-01-05 00:44:25
updated: 2023-01-08 02:33
tags:
- kernel
- plist
categories:
- kernel
- util
keywords:
- plist
---
plist 是内核中一种带优先级的双向链表数据结构，链表中每个元素的优先级按数值大小升序排序。plist 作为 pi-futex 的一部分补丁引入内核，初始提交链接为：[https://github.com/torvalds/linux/commit/77ba89c5cf28d5d98a3cae17f67a3e42b102cc25](https://github.com/torvalds/linux/commit/77ba89c5cf28d5d98a3cae17f67a3e42b102cc25)。本文基于 6.2-rc2 版本内核解析 plist 实现。

<!-- more -->

## 基本结构
plist 是基于 kernel 中的双向链表 list 实现的，其有两个重要结构体：`struct plist_head` 和 `struct plist_node`，分别表示 plist 的 head 和普通结点，它们的具体内容如下：

```C
struct plist_head {
	struct list_head node_list;
};

struct plist_node {
	int			prio;
	struct list_head	prio_list;
	struct list_head	node_list;
};

struct list_head {
	struct list_head *next, *prev;
};
```

`struct plist_head` 中只有一个 `struct list_head node_list;` 成员，其是 `struct plist_node` 中 `struct list_head node_list;` 的 head，这个链表包含了加入 plist 的所有结点。从 `struct plist_node` 结构体中看到还有另外一个会由 `struct list_head prio_list;` 构成的链表，这个链表可能没有包含全部的结点，因为相同优先级的结点只有第一个会加入 `prio_list` 中。这个链表用于加速找到要插入结点的正确位置，由于每个优先级只有一个结点会在链表上，所以这个链表的长度最长是优先级的范围。假设优先级范围为 K，在插入一个新元素时就会遍历 `prio_list` 链表来找到正确的位置，那么最多就只会遍历 K 个元素，这保证了插入元素算法的最坏复杂度为 `O(K)`。`prio` 成员自然就表示该结点的优先级了。

下面来个实例看看 plist 中具体结构是怎么样的。
1. 通过类似 `PLIST_HEAD(example_head)` 定义并初始化一个 `struct plist_head example_head;` 结点，我们将得到以下结构：

    ![plist 初始化](head.png "plist 初始化")

    上图中 `node_list` 表示 `struct plist_head` 结构体中的 `struct list_head node_list;` 成员，`prev` 和 `next` 表示 `node_list` 中的 `struct list_head *prev;` 和 `struct list_head *next;`。 

2. 往 `example_head` 中加入一个优先级为 20 的结点 `struct plist_node node1;`：
    ![往 example_head 中加入一个优先级为 20 的结点](node1.png)

3. 往 `example_head` 中加入一个优先级为 19 的结点 `struct plist_node node2;`:
    ![往 example_head 中加入一个优先级为 19 的结点](node2.png)

4. 再次往 `example_head` 中加入一个优先级为 20 的结点 `struct plist_node node3;`:
    ![再次往 example_head 中加入一个优先级为 20 的结点](node3.png)

## API
根据操作方法可以分为以下几种 API：
1. 初始化。
2. 增加删除。
3. 遍历。