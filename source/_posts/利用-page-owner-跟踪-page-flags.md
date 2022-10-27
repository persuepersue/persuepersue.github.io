---
title: 利用 page_owner 跟踪 page flags
date: 2022-10-27 23:23:43
updated: 2022-10-27 23:23:43
tags:
- kernel debug
- page_owner
categories:
- kernel debug
---

在 kernel 的开发调试过程中，经常需要知道这个 page 的某个 page flag 在哪条路径被 Set、被 Clear 了。甚至是在项目死机问题 debug 上，由于 page 不知道在哪里**不合时宜**地被 mlock 了，被 lock 住了，单纯地看代码可能比较难看出设置 page flag 的路径在哪，除非是对代码很熟悉，甚至需要对同事的代码比较熟悉，知道哪位同事之前有某个需求用了一种“奇怪”的方式来实现，然后你可以灵光一闪想到这个死机问题没准和这个同事的修改有关。不过大部分情况我们是没有这个灵光一闪的，那就需要特别的 debug 手段了。page_owner 很适合这种场景。

<!-- more -->

# 0x00 例子：

假设有个死机问题，通过初步分析发现是因为有个 page 带了 PG_mlocked 标志，正好遇到处理这个页面的内核路径不允许带有 PG_mlocked 标志的页面，所以就死机了。那到底为什么这个 page 会带有 PG_mlocked 标志呢？我们直接从代码层面翻来覆去地查看，**都没有找到 PG_mlocked 从哪里来的**。这个时候 page_owner 这个利器就派上用场了，我们在 `page-flags.h` 文件中重新实现了 SetPageMlocked 和 TestSetPageMlocked，在这两个函数实现中调用 `__set_page_owner` 函数，如 `__set_page_owner(page, 0, 0x1);`。因为在内核模块中可能会调用 SetPageMlocked 和 TestSetPageMlocked，所以还需要在 `mm/page_owner.c`中 `EXPORT_SYMBOL(__set_page_owner);`。并且在死机时 `dump_page(page, "page set mlocker");`。这样在死机 dump 中就可以看到这个 page 被设置 PG_mlock 的路径了。

当然，像 PG_lock 这种标志还要考虑在 trylock_page 中加 `__set_page_owner`。



# 0x01 如何开启 page_owner

```sh
CONFIG_PAGE_OWNER=y

cmdline 中：
page_owner=on stack_depot_disable=off
```

