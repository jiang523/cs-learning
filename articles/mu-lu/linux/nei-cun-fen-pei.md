# 内存分配

## 前言

分析kernel内存管理模块之前，先提出两个问题:

1. 在0.11版本的内核中，每个进程所分配到的内存是64M，而内核支持最多同时运行64个进程，即需要64\*64M = 4G内存。然而在那个时代，计算机远远没有4G的物理内存配置，所以内核是如何分配4G内存的呢？
2. 在内核fork进程时，子进程会复制父进程的信息，那子进程是否会将父进程的内存完全拷贝一份？如果完全拷贝一份会不会让进程的复制开销很大？

## 内存分类

Linux将内存分为物理内存和虚拟内存，物理内存可以理解，就是我们计算机的硬件内存，如磁盘硬盘等，而虚拟内存，则是操作系统暴露给用户使用的地址，我们使用计算机或写程序时看到的内存概念都指的是虚拟内存。

虚拟内存是基于物理内存的映射，操作系统为我们完成了这个映射的工作。在我们操作虚拟内存的时候，操作系统会自动为我们找到这块虚拟内存对应的物理内存，这个找的动作也就是寻址的过程。

Linux三个地址的概念:

逻辑地址 : 用户看到的地址，Linux操作系统分配给每一个进程的独立地址

线性地址 : 总线地址，ARM使用分段机制（线性地址 = 逻辑地址 + 段基址）

物理地址 : CPU总线的直接地址

根据这三个地址的概念，我们大体可以知道内存的映射过程是:

逻辑地址 —> 线性地址 —> 物理地址

## 分段机制

一个Linux进程由以下构成:

数据段: 存放程序静态变量和全局变量

代码段：存放程序可执行代码指令

BSS段： 存放程序中未初始化的全局变量

堆： 存放进程运行中被动态分配的内存段

栈： 存放程序临时创建的局部变量

Linux为每个进程都分配了一个独立的虚拟内存空间，也就是一段空间，这张图描述了Linux的分段机制:

<div align="left">

<figure><img src="../../.gitbook/assets/image-20220628185741850.png" alt=""><figcaption></figcaption></figure>

</div>

根据这张图可以看出，进程虚拟内存的分段机制，让所有进程的内存形成了线性的结构，因此可以先来看看逻辑地址 —>线性地址是如何映射的。

逻辑地址可以理解为一个内存段中的偏移地址，要转化为线性地址，需要段基址+偏移地址，而linux0.11内核给每个进程分配的虚拟内存是64M，因此一个进程内存段的基地址是pid \* 64m, 线性地址为: pid\* \* 64m + 逻辑地址。

## 分页机制

上面分析了linux如何将逻辑地址和线性地址对应的，线性地址始终还是虚拟内存中的地址概念，那么linux是如何将虚拟内存和物理内存做映射的呢？答案是分页机制。

<div align="left">

<figure><img src="../../.gitbook/assets/image-20220119155919281-2579162.png" alt=""><figcaption></figcaption></figure>

</div>

上面这张图描述了虚拟内存到物理内存的映射过程。linux将物理内存以4k为单位分页，内存的读取可以连续也可以分散。

<div align="left">

<figure><img src="../../.gitbook/assets/image-20220127150603075.png" alt=""><figcaption></figcaption></figure>

</div>

这张图描述了虚拟内存到物理内存的寻址过程:CR3寄存器 —> 页目录 —> 页表 —> 物理页。

页表项的结构：

<div align="left">

<figure><img src="../../.gitbook/assets/image-20220127162905082.png" alt=""><figcaption></figcaption></figure>

</div>

由于物理地址是20位表示的，所以页表项剩下12位可以另作他用，而12位中有3位为os专用，1位空白，剩下8位中，主要有这几个:

| 位号 | 位名  | 含义                                             |
| -- | --- | ---------------------------------------------- |
| 0  | P   | 表示是否有指向的物理内存页，如果没有P位为0，如果有则为1                  |
| 1  | R/W | 表示该页表项的读写权限                                    |
| 2  | U/S | 表示允许访问的特权级别                                    |
| 3  | PWT | 表示是否采用写穿式(写穿式:既往内存写也往高速缓存里写 回写式:只往缓存写，定时同步到内存) |
| 4  | PCD | 表示是否开启高速缓存                                     |
| 5  | A   | Access标志，表示页表项是否可以被操作系统使用                      |
| 6  | D   | dirty位，如果页表被修改了，则为1                            |
| 7  | PSE | Page size，不适合于页表项                              |

## 内存数据结构

我们都知道，内核是基于task\_struct结构体来管理进程的，而在task\_struct中，是通过mm\_struct来管理内存的。

1. task\_struct:

```c
struct task_struct {
  struct mm_struct *mm;
}
```

1. mm\_struct:

```c
<mm_types.h>
struct mm_struct {
  struct vm_area_struct * mmap;        //vma链表
  struct rb_root mm_rb;                //vma红黑树根节点 
  struct vm_area_struct * mmap_cache;  //vma缓存
}
```

Linux内核中，关于**虚存管理的最基本的管理单元**应该是struct vm\_area\_struct了，它描述的是一**段连续的、具有相同访问属性的虚存空间，该虚存空间的大小为物理内存页面的整数倍**。一个进程只有一个mm\_struct，而一个mm\_struct可以对应多个vm\_area\_struct，mm\_struct中组织了链表和红黑树两种结构来保存vm\_area\_struct，当vma结点较多时，红黑树可以加速检索。

1. vm\_area\_struct:

```c
struct vm_area_struct {
  struct mm_struct * vm_mm;  
  unsigned long vm_start; 
  unsigned long vm_end;
  struct vm_area_struct *vm_next;
  pgprot_t vm_page_prot;
  unsigned long vm_flags;
  struct rb_node vm_rb;
  shared address_space->i_mmap address_space->i_mmap_nonlinear
    struct {
      struct list_head list;
      void *parent; 
    } vm_set;
  };
   struct raw_prio_tree_node prio_tree_node;
  } shared;
​
  struct list_head anon_vma_node; 
  struct vm_operations_struct * vm_ops;
​
  unsigned long vm_pgoff; 
  struct file * vm_file; 
  void * vm_private_data; 
}
```

* [ ] vm\_mm是一个反向指针，指向vma所属的mm\_struct
* [ ] vm\_start和vm\_end指向线性区第一个线性地址和线性区后的第一个线性地址
* [ ] vm\_next指向链表的下一个节点
* [ ] vm\_page\_prot代表访问特权级别
* [ ] vm\_flags描述该区域的一组标志
* [ ] vm\_rb是红黑树节点
* [ ] 给出一个文件内存区间，内核有时候需要知道该区间映射到的所有进程，这种映射叫做共享映射。一个程序可以选择**MAP\_SHARED或MAP\_PRIVATE共享模式**将一个文件的某部分数据映射到自己的虚存空间里面。这两种映射方式的区别在于：MAP\_SHARED映射后在内存中对该虚存空间的数据进行修改会影响到其他以同样方式映射该部分数据的进程，并且该修改还会被写回文件里面去，也就是这些进程实际上是在共用这些数据。而MAP\_PRIVATE映射后对该虚存空间的数据进行修改不会影响到其他进程，也不会被写入文件中。
* [ ] vm\_ops指向一些要用到的函数
* [ ] vm\_file指向映射的文件

目前介绍了三种虚拟内存相关的数据结构，下图表示了这三种数据结构所构成的结构:

<div align="left">

<figure><img src="../../.gitbook/assets/image-20220628154603670.png" alt=""><figcaption></figcaption></figure>

</div>

1.  优先查找树:

    内核中，使用优先查找树建立一个文件的一个区域与这个区域所映射到的虚拟内存的关联。一个文件用struct file表示，而这个结构体中包含一个指向地址空间的指针struct address\_space

    ```
    struct address_space {
      struct inode *host; 
      struct prio_tree_root i_mmap;
      struct list_head i_mmap_nonlinear;
    }
    struct file { 
      struct address_space *f_mapping;
    }
    ```

映射的过程如下图所示:

<div align="left">

<figure><img src="../../.gitbook/assets/image-20220628155843930.png" alt=""><figcaption></figcaption></figure>

</div>

## 虚拟内存映射

### **虚拟内存分配**

程序调用malloc函数申请内存，最终会调用brk系统调用来从堆空间中分配内存。我们来分析一下brk系统调用的实现：

```c
unsigned long sys_brk(unsigned long brk)
{
  unsigned long rlim, retval;
  unsigned long newbrk, oldbrk;
  struct mm_struct *mm = current->mm;
​
  down_write(&mm->mmap_sem);
​
  /**
   * 首先验证addr参数是否位于进程代码所在的线性区。
   * 如果是，则立即返回。因为堆不能与进程代码所在的线性区重叠。
   */
  if (brk < mm->end_code)
    goto out;
  /**
   * 由于brk系统调用作用于某一个线性区，它分配和释放完整的页。
   * 因此，把addr高速为PAGE_SIZE的整数倍。然后把调整的结束与内存描述符的brk字段的值进行比较。
   */
  newbrk = PAGE_ALIGN(brk);
  oldbrk = PAGE_ALIGN(mm->brk);
  if (oldbrk == newbrk)
    goto set_brk;
  /**
   * 如果是请求缩小堆，则调用do_munmap函数完成任务。并返回。
   */
  if (brk <= mm->brk) {
    if (!do_munmap(mm, newbrk, oldbrk-newbrk))
      goto set_brk;
    goto out;
  }
  /**
   * 如果是请求扩大堆，则首先检查是否允许进程这么做。
   */
  rlim = current->signal->rlim[RLIMIT_DATA].rlim_cur;
  if (rlim < RLIM_INFINITY && brk - mm->start_data > rlim)
    goto out;
  /**
   * 然后检查扩大后的堆是否与进程其他线性区重叠。如果是，不做任何事情就返回。
   */
  if (find_vma_intersection(mm, oldbrk, newbrk+PAGE_SIZE))
    goto out;
  /**
   * 如果一切顺利，则调用do_brk函数。该函数其实是do_mmap的简化版。
   * 如果它返回oldbrk，则分配成功且sys_brk返回addr的值，否则返回mm->brk值。
   */
  if (do_brk(oldbrk, newbrk-oldbrk) != oldbrk)
    goto out;
set_brk:
  mm->brk = brk;
out:
  retval = mm->brk;
  up_write(&mm->mmap_sem);
  return retval;
}
```

总结上面的代码，主要有以下几个步骤：

* 1、判断堆空间的大小是否超出限制，如果超出限制，就不作任何处理，直接返回旧的 `brk` 值。
* 2、如果新的 `brk` 值跟旧的 `brk` 值一致，那么也不用作任何处理。
* 3、如果新的 `brk` 值发生变化，那么就调用 `do_brk` 函数进行下一步处理。
* 4、设置进程的 `brk` 指针（堆空间顶部）为新的 `brk` 的值。

我们看到第 3 步调用了 `do_brk` 函数来处理，`do_brk` 函数的实现有点小复杂，所以这里介绍一下大概处理流程：

* 通过堆空间的起始地址 `start_brk` 从进程内存分区红黑树中找到其对应的内存分区对象（也就是 `vm_area_struct`）。
* 把堆空间的内存分区对象的 `vm_end` 字段设置为新的 `brk` 值。

至此，`brk` 系统调用的工作就完成了（上面没有分析释放内存的情况），总结来说，`brk` 系统调用的工作主要有两部分：

1. 把进程的 `brk` 指针设置为新的 `brk` 值。
2. 把堆空间的内存分区对象的 `vm_end` 字段设置为新的 `brk` 值。

### **物理内存分配**

虚拟内存分配好以后，程序就可以通过分配好的虚拟内存地址去寻找对应的物理内存。根据前面的介绍，内核会根据虚拟内存的线性地址，先从cr3中读取页目录项，然后读取页表项，最后根据页表项，去查找对应的物理内存页。

问题来了，前面我们只分配了虚拟内存，此时内核并没有建立虚拟内存和物理内存的映射，也就是说页表项根本没有对应的物理内存页。

一旦内核找不到对应的物理内存页，就会触发一种中断:**缺页异常中断**，该中断的中断服务函数如下:

```c
void do_page_fault(struct pt_regs *regs, unsigned long error_code)
 {
    struct vm_area_struct *vma;
    struct task_struct *tsk;
    unsigned long address;
    struct mm_struct *mm;
    int write;
    int fault;
 
    tsk = current;
    mm = tsk->mm;
​
    address = read_cr2();        // 获取导致页缺失异常的虚拟内存地址
    ...
    vma = find_vma(mm, address); // 通过虚拟内存地址从进程内存分区中查找对应的内存分区对象
    ...
    if (likely(vma->vm_start <= address)) // 如果找到内存分区对象
        goto good_area;
    ...
​
   good_area:
     write = error_code & PF_WRITE;
    ...
     // 调用 handle_mm_fault 函数对虚拟内存地址进行映射操作
     fault = handle_mm_fault(mm, vma, address, write ? FAULT_FLAG_WRITE : 0);
    ...
}
```

该函数做了这些工作:

1. 从cr2寄存器读取引发缺页异常的虚拟内存地址
2. 通过虚拟内存地址找到vm\_area\_struct
3. 调用handle\_mm\_fault函数，将物理内存页映射到虚拟内存页

handle\_mm\_fault:

```c
int handle_mm_fault(struct mm_struct *mm, struct vm_area_struct *vma,
                    unsigned long address, unsigned int flags)
 {
    pgd_t *pgd;  // 页全局目录项
    pud_t *pud;  // 页上级目录项
    pmd_t *pmd;  // 页中间目录项
    pte_t *pte;  // 页表项
    ...
    pgd = pgd_offset(mm, address);         // 获取虚拟内存地址对应的页全局目录项
    pud = pud_alloc(mm, pgd, address);     // 获取虚拟内存地址对应的页上级目录项
    ...
    pmd = pmd_alloc(mm, pud, address);     // 获取虚拟内存地址对应的页中间目录项
    ...
    pte = pte_alloc_map(mm, pmd, address); // 获取虚拟内存地址对应的页表项
    ...
    // 对页表项进行映射
    return handle_pte_fault(mm, vma, address, pte, pmd, flags);
}
```

handle\_mm\_fault获取了页目录项和页表项，最终调用了handle\_pte\_fault函数对页表项目进行了映射，继续来看看handle\_pte\_fault函数:

```c
static inline int handle_pte_fault(struct mm_struct *mm,
  struct vm_area_struct * vma, unsigned long address,
  int write_access, pte_t *pte, pmd_t *pmd)
{
  pte_t entry;
  entry = *pte;
  /**
   * 页还不存在，内核分配一个新的页表并适当的初始化。
   * 即请求调页。
   */
  if (!pte_present(entry)) {
    /**
     * pte_none返回1，说明页从未被进程访问且没有映射磁盘文件。
     */
    if (pte_none(entry))
      return do_no_page(mm, vma, address, write_access, pte, pmd);
    /**
     * pte_file返回1，说明页属于非线性磁盘文件的映射。
     */
    if (pte_file(entry))
      return do_file_page(mm, vma, address, write_access, pte, pmd);
​
    /**
     * 这个页曾经被访问过，但是其内容被临时保存在磁盘上。
     */
    return do_swap_page(mm, vma, address, pte, pmd, entry, write_access);
  }
​
  /**
   * 页存在，但是为只读。就分配一个新的页框，并把旧页框的数据复制到新页框。即COW
   */
  if (write_access) {
    if (!pte_write(entry))
      return do_wp_page(mm, vma, address, pte, pmd, entry);
    entry = pte_mkdirty(entry);
  }
  entry = pte_mkyoung(entry);
  ptep_set_access_flags(vma, address, pte, entry, write_access);
  update_mmu_cache(vma, address, entry);
  pte_unmap(pte);
  spin_unlock(&mm->page_table_lock);
  return VM_FAULT_MINOR;
}
```

根据不同的类型，会进行不同的处理，以do\_no\_page为例:

```c
static int
do_anonymous_page(struct mm_struct *mm, struct vm_area_struct *vma,
    pte_t *page_table, pmd_t *pmd, int write_access,
    unsigned long addr)
{
  pte_t entry;
  struct page * page = ZERO_PAGE(addr);
​
  /* Read-only mapping of ZERO_PAGE. */
  /**
   * 对读访问时，页的内容是无关紧要的。
   * 但是，第一次分给进程的页，最好还是填上0。以免旧页的信息被黑客利用。
   * 没有必要立即分配这样的页框。只需要把empty_zero_page页映射给进程就行了。
   * 并且将页标记为只读，当进程试图写这个页时，再启动写时复制机制。
   */
  entry = pte_wrprotect(mk_pte(ZERO_PAGE(addr), vma->vm_page_prot));
​
  /* ..except if it's a write access */
  if (write_access) {
    /**
     * 释放临时内核映射。
     * 在调用handle_pte_fault函数之前，由pte_offset_map所建立页表项的高端内存物理地址。
     * pte_offset_map宏是和pte_unmap配对使用的。
     * pte_unmap必须在alloc_page前释放。因为alloc_page可能阻塞当前进程？？
     */
    pte_unmap(page_table);
    spin_unlock(&mm->page_table_lock);
​
    if (unlikely(anon_vma_prepare(vma)))
      goto no_mem;
    page = alloc_zeroed_user_highpage(vma, addr);
    if (!page)
      goto no_mem;
​
    spin_lock(&mm->page_table_lock);
    page_table = pte_offset_map(pmd, addr);
​
    if (!pte_none(*page_table)) {
      pte_unmap(page_table);
      page_cache_release(page);
      spin_unlock(&mm->page_table_lock);
      goto out;
    }
    /**
     * 递增rss字段，它记录了分配给进程的页框总数。
     */
    mm->rss++;
    acct_update_integrals();
    update_mem_hiwater();
    /**
     * 标记页框为既脏又可写。
     * 如果调试程序试图往被跟踪进程只读线性区中的页中写数据。内核不会设置相关标志。
     * maybe_mkwrite会处理这种特殊情况。
     */
    entry = maybe_mkwrite(pte_mkdirty(mk_pte(page,
               vma->vm_page_prot)),
              vma);
    /**
     * lru_cache_add_active把新页框插入与交换相关的数据结构中。
     */
    lru_cache_add_active(page);
    SetPageReferenced(page);
    page_add_anon_rmap(page, vma, addr);
  }
​
  set_pte(page_table, entry);
  pte_unmap(page_table);
​
  /* No need to invalidate - it was non-present before */
  update_mmu_cache(vma, addr, entry);
  spin_unlock(&mm->page_table_lock);
out:
  return VM_FAULT_MINOR;
no_mem:
  return VM_FAULT_OOM;
}
```

`do_anonymous_page` 函数的实现比较有趣，它会根据 `缺页异常` 是由读操作还是写操作导致的，分为两个不同的处理逻辑，如下：

* 如果是读操作导致的，那么将会使用 `零页` 进行映射（`零页` 是 Linux 内核中一个比较特殊的内存页，所有读操作引起的 `缺页异常` 都会指向此页，从而可以减少物理内存的消耗），并且设置其为只读（因为 `零页` 是不能进行写操作）。如果下次对此页进行写操作，将会触发写操作的 `缺页异常`，从而进入下面步骤。
* 如果是写操作导致的，就申请一块新的物理内存页，然后根据物理内存页的地址生成映射关系，再对页表项进行填充（映射）。

`do_anonymous_page`函数将真实的物理内存页和虚拟内存地址做了映射，这样一来缺页异常中断服务函数执行完毕，内存分配完毕。

## 总结

回到开头的两个问题，可以给出回答了:

1. Linux采用虚拟内存管理技术，每个进程都有各自独立的进程地址空间(即4G的线性虚拟空间)，无法直接访问物理内存。这样起到保护操作系统，并且让用户程序可使用比实际物理内存更大的地址空间。
2. fork()时，父子进程共享页表，只有当子进程或父进程试图修改特定页表项时，内核才创建该页表项的新拷贝，之后父子进程不再共享该页表项。可见，利用共享页表可以消除fork()操作中页表拷贝所带来的消耗。这种机制叫做copy on write

本文介绍了linux内存管理，主要介绍了虚拟内存的数据结构和虚拟内存和物理内存的映射过程，重点是缺页异常机制及其中断服务函数的处理。

\
