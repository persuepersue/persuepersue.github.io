---
title: plist 源码解析
date: 2023-01-05 00:44:25
updated: 2023-01-10 01:24
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
1. 初始化
2. 增加删除
3. 遍历

### 初始化
初始化有头结点初始化和普通结点初始化两种 API，这两种又分为静态初始化和动态初始化。
#### 头结点初始化
头结点静态初始化有两个 API: `PLIST_HEAD_INIT` 和 `PLIST_HEAD`，而 `PLIST_HEAD` 是通过 `PLIST_HEAD_INIT` 实现的。一般使用 `PLIST_HEAD` 初始化一个 `struct plist_head` 结构体：
```c
#define PLIST_HEAD_INIT(head)				\
{							\
	.node_list = LIST_HEAD_INIT((head).node_list)	\
}

#define PLIST_HEAD(head) \
	struct plist_head head = PLIST_HEAD_INIT(head)
```
具体用法如 `mm/swapfile.c` 中 `static PLIST_HEAD(swap_active_head);`。当然也可借助 `PLIST_HEAD_INIT` 宏来完成 `plist_head` 的初始化了，如 `kernel/power/qos.c` 中这样使用：
```c
static struct pm_qos_constraints cpu_latency_constraints = {
	.list = PLIST_HEAD_INIT(cpu_latency_constraints.list),
	.target_value = PM_QOS_CPU_LATENCY_DEFAULT_VALUE,
	.default_value = PM_QOS_CPU_LATENCY_DEFAULT_VALUE,
	.no_constraint_value = PM_QOS_CPU_LATENCY_DEFAULT_VALUE,
	.type = PM_QOS_MIN,
};
```
头结点的动态初始化 API 为 `plist_head_init`：
```c
static inline void
plist_head_init(struct plist_head *head)
{
	INIT_LIST_HEAD(&head->node_list);
}
```
一般 `head` 不能或者不方便静态初始化时，譬如 `mm/swapfile.c` 中 `plist_head_init(&swap_avail_heads[nid]);`。

### 普通结点初始化
普通结点通过 `PLIST_NODE_INIT` 静态初始化：
```c
#define PLIST_NODE_INIT(node, __prio)			\
{							\
	.prio  = (__prio),				\
	.prio_list = LIST_HEAD_INIT((node).prio_list),	\
	.node_list = LIST_HEAD_INIT((node).node_list),	\
}
```
如 `init/init_task.c` 中：
```c
struct task_struct init_task
= {
    ...;
#ifdef CONFIG_SMP
	.pushable_tasks	= PLIST_NODE_INIT(init_task.pushable_tasks, MAX_PRIO),
#endif
    ...;
};
```
通过 `plist_node_init` 来动态初始化：
```c
static inline void plist_node_init(struct plist_node *node, int prio)
{
	node->prio = prio;
	INIT_LIST_HEAD(&node->prio_list);
	INIT_LIST_HEAD(&node->node_list);
}
```
如 `mm/swapfile.c` 中 `plist_node_init(&p->avail_lists[i], 0);`。

### 增加删除
增加的 API 是 `plist_add` 函数，删除是 `plist_del` 函数。还有一个特殊的函数，相当于优化版本的先删除再加入：`plist_requeue`，这个函数的用途是譬如系统中有很多个 swapfile，可能有几个 swapfile 的优先级是相同的，那么我们希望可以轮询使用相同优先级的 swapfile，具体操作就是遍历这个 swapfile 时，调用 `plist_requeue` 函数将当前 swapfile 置于相同优先级的最后一个，这样就可以达到轮询相同优先级的效果了。