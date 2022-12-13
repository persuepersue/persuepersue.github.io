---
title: KBASE_IOCTL_MEM_ALLOC_EX 解析
date: 2022-12-04 15:06:30
updated: 2022-12-04 15:06:30
tags:
- mali valhall gpu
- arm64
- gpu
categories:
- mali gpu kernel driver
---

源码来源：https://armkeil.blob.core.windows.net/developer/Files/downloads/mali-drivers/kernel/mali-valhall-gpu/VX504X08X-SW-99002-r41p0-01eac0.tar

这篇文章详解 `kbase_ioctl` 中的 `KBASE_IOCTL_MEM_ALLOC_EX` 命令。最新的 mali GPU 支持了 CSF，因此这篇的讲解基于 `MALI_USE_CSF` 使能的配置。

```c
static long kbase_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
    void __user *uarg = (void __user *)arg;
    ...;
    switch (cmd) {
#if MALI_USE_CSF
    case KBASE_IOCTL_MEM_ALLOC_EX:
    		KBASE_HANDLE_IOCTL_INOUT(KBASE_IOCTL_MEM_ALLOC_EX, kbase_api_mem_alloc_ex,
                                     union kbase_ioctl_mem_alloc_ex, kctx);
    		break;
#endif
     ...;
    }
    ...;
}
```

`KBASE_HANDLE_IOCTL_INOUT` 表示 `KBASE_IOCTL_MEM_ALLOC_EX` 是有输入并且有输出的 cmd，输入输出都是 `uarg`。在 `KBASE_HANDLE_IOCTL_INOUT` 中先将 `uarg` 通过 `copy_from_user(&param, uarg, sizeof(param));` 拷贝到一个临时变量 `union kbase_ioctl_mem_alloc_ex param` 中，然后调用 `ret = kbase_api_mem_alloc_ex(kctx, param);`，再调用 `copy_to_user(uarg, &param, sizeof(param));` 将 `param` 拷贝到 `uarg` 中，最终返回 `ret`（如果在两处 copy 中出错，则返回 `-EFAULT`）。

<!-- more -->

以上是 `KBASE_IOCTL_MEM_ALLOC_EX`  的粗略流程，具体内容在 `kbase_api_mem_alloc_ex` 函数中，接下来详解这个函数。

# union kbase_ioctl_mem_alloc_ex

上面说到 `KBASE_IOCTL_MEM_ALLOC_EX` 用户态传进来的参数和内核返回用户的数据都是 `union kbase_ioctl_mem_alloc_ex` 类型表示的，那么首先来看下这个类型：

```c
/**
 * union kbase_ioctl_mem_alloc_ex - Allocate memory on the GPU
 * @in: Input parameters
 * @in.va_pages: The number of pages of virtual address space to reserve
 * @in.commit_pages: The number of physical pages to allocate
 * @in.extension: The number of extra pages to allocate on each GPU fault which grows the region
 * @in.flags: Flags
 * @in.fixed_address: The GPU virtual address requested for the allocation,
 *                    if the allocation is using the BASE_MEM_FIXED flag.
 * @in.extra: Space for extra parameters that may be added in the future.
 * @out: Output parameters
 * @out.flags: Flags
 * @out.gpu_va: The GPU virtual address which is allocated
 */
union kbase_ioctl_mem_alloc_ex {
        struct {
                __u64 va_pages;
                __u64 commit_pages;
                __u64 extension;
                __u64 flags;
                __u64 fixed_address;
                __u64 extra[3];
        } in; 
        struct {
                __u64 flags;
                __u64 gpu_va;
        } out;
};
```

从联合体结合注释很简便看出，`in` 部分对应入参，`out` 部分对应出参。

1. `in`
   1. `va_pages`: 虚拟空间大小
   2. `commit_pages`: 需要分配的物理页
   3. `extension`: 每次 GPU fault 需要额外分配的页面数
   4. `flags`
   5. `fixed_address`: 如果 `flags` 中带了 `BASE_MEM_FIXED` 标志，则考虑用 `fixed_address` 作为 GPU 虚拟空间地址分配
   6. `extras`: 保留
2. `out`
   1. `flags`
   2. `gpu_va`: 返回的 GPU 虚拟空间地址



# kbase_mem_allow_alloc

由于 mali 的 user space 驱动是闭源的，所以我也只能靠猜测 mali 驱动是如何使用的。

`kbase_api_mem_alloc_ex` 一开始就调用了 `kbase_mem_allow_alloc(kctx)` 来判断这个 `kctx` 可不可以分配空间：

```c
/**
 * kbase_mem_allow_alloc - Check if allocation of GPU memory is allowed
 * @kctx: Pointer to kbase context
 *
 * Don't allow the allocation of GPU memory until user space has set up the
 * tracking page (which sets kctx->process_mm) or if the ioctl has been issued
 * from the forked child process using the mali device file fd inherited from
 * the parent process.
 *
 * Return: true if allocation is allowed.
 */
static inline bool kbase_mem_allow_alloc(struct kbase_context *kctx)
{
        bool allow_alloc = true;

        rcu_read_lock();
        allow_alloc = (rcu_dereference(kctx->process_mm) == current->mm);
        rcu_read_unlock();

        return allow_alloc;
}
```

粗看这个函数，只是判断了 `kctx->process_mm`，那这个 `process_mm` 是什么呢？

```c
/*
 * @process_mm:           Pointer to the memory descriptor of the process which
 *                        created the context. Used for accounting the physical
 *                        pages used for GPU allocations, done for the context,
 *                        to the memory consumed by the process.
 */
struct kbase_context {
    ...;
    struct mm_struct __rcu *process_mm;
    ...;
};
```

根据注释看，`process_mm` 是哪个进程创建了这个 `kctx`，它的值就是这个进程的 `mm` 成员。用处是用来跟踪这个进程在 GPU 中分配的内存。

再看 `kbase_mem_allow_alloc` 注释，如果 `process_mm` 还未设置或者发出 `ioctl` 命令的进程是设置 `process_mm` 的进程 `fork` 出来的子进程，分配是不允许的。换句话来说，只允许已经设置了 `process_mm` 的进程来分配。注释中还说到， `process_mm` 是在设置 `tracking page` 时设置的，这个 `tracking page` 是什么呢？按照 `process_mm` 的含义，估计就是用来跟踪进程 GPU 分配情况的。来搜索下 `process_mm` 具体是怎么设置的：

```c
static int kbase_tracking_page_setup(struct kbase_context *kctx, struct vm_area_struct *vma)
{
        /* check that this is the only tracking page */
        spin_lock(&kctx->mm_update_lock);
        if (rcu_dereference_protected(kctx->process_mm, lockdep_is_held(&kctx->mm_update_lock))) {
                spin_unlock(&kctx->mm_update_lock);
                return -EFAULT;
        }

        rcu_assign_pointer(kctx->process_mm, current->mm);

        spin_unlock(&kctx->mm_update_lock);

        /* no real access */
        vma->vm_flags &= ~(VM_READ | VM_MAYREAD | VM_WRITE | VM_MAYWRITE | VM_EXEC | VM_MAYEXEC);
        vma->vm_flags |= VM_DONTCOPY | VM_DONTEXPAND | VM_DONTDUMP | VM_IO;
        vma->vm_ops = &kbase_vm_special_ops;
        vma->vm_private_data = kctx;

        return 0;
}
```

这个函数的调用流程如下：

{% mermaid graph %}
A("mmap=>operation: mmap 时调用 .mmap 钩子 kbase_mmap(filp, vma);") -->B("kbase_context_mmap(kctx, vma);")
    B --> C{根据 vm_pgoff 的值来做不同的操作}
    C -->|"vma->vm_pgoff == PFN_DOWN(BASE_MEM_MAP_TRACKING_HANDLE)"| D["kbase_tracking_page_setup(kctx, vma);"]
{% endmermaid %}
