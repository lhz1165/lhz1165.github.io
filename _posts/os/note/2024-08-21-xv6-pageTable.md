---
layout: post
title: "xv6-PageTable虚拟内存"
category: os-6s081-source
---

6.s081 pageTable

# 虚拟内存-页表机制
虚拟地址：分为两部分，前一个部分用来作为页表的索引(数组下标)；后面部分作为页内偏移量(OFFSET)

操作系统把物理内存分成一个一个的页，一个页大小由人工设定，这影响到虚拟内存的OFFSET长度







### 物理内存管理

将**空闲物理内存**划分物理页每页是4096byte大小，**空闲物理内存**是链表结构

```c
struct {
  struct spinlock lock;
  struct run *freelist;
} kmem;

struct run {
  struct run *next;
};
```

**kalloc(void)** ：从空闲物理页的分配一页，从链表上摘下来。

**kfree(void *pa)** ：把pa处往后的(4096b)的物理页，从新添加进空闲链表。

void freerange(void *pa_start, void *pa_end)：释放地址间的所有物理内存（加入到空闲链表）。

**kinit()**： 初始化物理内存

```c
{
  initlock(&kmem.lock, "kmem");
  freerange(end, (void*)PHYSTOP);
}
```

### 内核态虚拟内存

**walk(pagetable_t pagetable, uint64 va, int alloc)**:用软件来模拟硬件MMU查找页表。找到虚拟地址对应的最后一级页表的PTE的指针。

**walkaddr(pagetable_t pagetable, uint64 va)**：调用walk，找到pte，然后取出pte的地址，<<10,再>>12得到物理页号的地址（开始地址，再用va的偏移量拼接才能得到va对应的物理地址）。

**freewalk(pagetable_t pagetable)**：递归回收页表每一项，pagetable[i] = 0，并回收pte所在的每个物理页;

**walkaddr（pagetable_t pagetable, uint64 va）**：先通过walk找到pte，再通过pte找到物理页的地址。

**int mappages（pagetable_t pagetable, uint64 va, uint64 size, uint64 pa, int perm）**：把va和pa在页表中建立联系，

先根据va取得pte地址，pa取得物理页地址，把物理页地址写入pte指针所指的位置

**kvmmap()** ：等于mappages（），额外做了异常处理

**void proc_mapstacks(pagetable_t kpgtbl)**：为每个进程初始化内核栈（64个），因为每个进程的内核栈不一样

**copyin(pagetable_t pagetable, char *dst, uint64 srcva, uint64 len)**：把用户态va作为src，拷贝数据到内核的页表

### 用户态虚拟内存

**proc_pagetable()**： 为进程分配物理页，建立页表，TRAMPOLINE和TRAPFRAME的映射

**proc_freepagetable()**：回收TRAPFRAME的(pte=0，因为TRAPFRAME是多个进程映射的所以不能回收物理内存)，回收自身所有的pte和物理内存,主要调用uvmunmap和uvmfree

**uvmunmap**：从va开始往后npages个虚拟页设为无效（令叶子节点的每个pte=0）, 如果do_free=1， **kfree回收pte所指向的的物理页**

proc_freepagetable包含以下两个

1. **uvmunmap**：回收进程页表TRAPFRAME的虚拟页（pte=0）

2. **uvmfree**：回收进程所有的虚拟页（pte=0）和物理页（kfree），回收进程的页表的虚拟页和物理页（uvmunmap和freewalk）



