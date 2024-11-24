---
layout: post
title: "xv6-源码解析-PageTable"
category: os-6s081-source
---

xv6 虚拟内存实现

# 虚拟内存-页表机制
### 虚拟内存概念

虚拟地址：分为两部分，前一个部分用来作为页表的索引(数组下标)；后面部分作为页内偏移量(OFFSET)。

操作系统把物理内存分成一个一个的页，一个页大小预先定好(2^n)，这决定了虚拟内存的OFFSET长度n。

操作系统把空的页用链表保存起来，已使用的页移除链表，通过页表记录。



页表有是一个PTE数组(page table entry)

每个PTE的内容是PPN（Physical Page Number）和FLAG

PPN 指向了实际存储数据的物理页帧

![image-20240427143035383](/assets/os/note-page/1.png)

FLAG告诉硬件这个物理页的权限

![image-20240427143035383](/assets/os/note-page/2.png)



操作系统的内存有内核地址空间和用户进程地址空间如下

**内核地址空间**

![image-20240427143035383](/assets/os/note-page/kernel_space.png)

- The trampoline page: 用来保存用户态和内核态的上下文信息的页面
- The kernel stack pages：每个进程都有一个内存栈，刚开始初始化使用临时栈(间entry.S)，有了进程使用这些内核栈
- Guard page：防止栈溢出影响到别的栈，用来作为保护区域，访问这一片会异常

**用户进程地址空间**

![image-20240427143035383](/assets/os/note-page/user_space.png)

### 初始化地址空间

参见 vm.c 代码

#### 物理内存管理

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



**页表的指针**

```
typedef uint64 *pagetable_t; // 512 PTEs
```



#### **查找虚拟地址的pte**

为了软件模拟查找虚拟地址的物理地址首先要找到PTE

```c
pte_t *
walk(pagetable_t pagetable, uint64 va, int alloc)
{
  if(va >= MAXVA)
    panic("walk");

  for(int level = 2; level > 0; level--) {
    pte_t *pte = &pagetable[PX(level, va)];// 三级页表，从第一级开始，找到index(L2,L1)的pet
    if(*pte & PTE_V) {
      pagetable = (pagetable_t)PTE2PA(*pte); //pet=（44(ppn)+10（flag）），所以找到下一级的pa（ppn）的pagetable
    } else {
      if(!alloc || (pagetable = (pde_t*)kalloc()) == 0)
        return 0;
      memset(pagetable, 0, PGSIZE);
      *pte = PA2PTE(pagetable) | PTE_V;
    }
  }
  return &pagetable[PX(0, va)];//找到了最后一级pagetable，只需要右移12位L0
}
```

**把虚拟地址安装到 pageTable**

一半在分配新对象，获取一个未使用物理地址，然后再根据已使用的虚拟地址分配一个未使用虚拟地址，把他们在页表建立映射，kvmmap 封装了 mappages ，他们等价

```c
int mappages(pagetable_t pagetable, uint64 va, uint64 size, uint64 pa, int perm)
{
  uint64 a, last;
  pte_t *pte;

  if(size == 0)
    panic("mappages: size");
  
  a = PGROUNDDOWN(va);
  last = PGROUNDDOWN(va + size - 1);
  for(;;){
    if((pte = walk(pagetable, a, 1)) == 0) //L0的pet找到了，PPN+offset就是PA
      return -1;
    if(*pte & PTE_V)
      panic("mappages: remap");
    *pte = PA2PTE(pa) | perm | PTE_V; //(PPN(44)+FLAG(10)) | perm | PTE_V 初始化这页的权限
    if(a == last)
      break;
    a += PGSIZE;
    pa += PGSIZE;
  }
  return 0;
}

```



#### 完全释放用户进程页表

uvmfree（）：完全释放用户的地址空间

uvmunmap() ：清除最后一级页表PTE。同时回收PTE指向的物理页内存

 freewalk()：清空每一级页表的所有PTE(必须保证最后一级PTE已经被清理了)。回收页表的物理内存

freewalk必须和uvmunmap配合使用，先uvmunmap才能调用freewalk

```c
//两者一般配合使用
void
uvmfree(pagetable_t pagetable, uint64 sz)
{
  if(sz > 0)
    uvmunmap(pagetable, 0, PGROUNDUP(sz)/PGSIZE, 1);
  freewalk(pagetable);
}


//pagetab进程页表，va虚拟地址开始，npages 往后n个页面，do_free 是否清空物理内存
void uvmunmap(pagetable_t pagetable, uint64 va, uint64 npages, int do_free)
{
  uint64 a;
  pte_t *pte;
	
  if((va % PGSIZE) != 0)
    panic("uvmunmap: not aligned");

  for(a = va; a < va + npages*PGSIZE; a += PGSIZE){
    if((pte = walk(pagetable, a, 0)) == 0)
      panic("uvmunmap: walk");
    if((*pte & PTE_V) == 0)
      panic("uvmunmap: not mapped");
    if(PTE_FLAGS(*pte) == PTE_V)
      panic("uvmunmap: not a leaf");
    if(do_free){
      uint64 pa = PTE2PA(*pte);
        //2.物理页加入空闲链表，回收物理内存
      kfree((void*)pa);
    }
     //1.核心点清空叶子节点(最后一级页表的)的pte
    *pte = 0;
  }
}

void
freewalk(pagetable_t pagetable)
{
  // there are 2^9 = 512 PTEs in a page table.
  for(int i = 0; i < 512; i++){
    pte_t pte = pagetable[i];
      
    //1. 如果 PTE_R、PTE_W 或 PTE_X 中的任何一个位被设置，则说明该条目指向一个物理页，代表一个叶子节点
    //2. 如果这些位均未设置，但 PTE_V 被设置，则说明该条目不是叶子节点，而是指向下一级页表。
    if((pte & PTE_V) && (pte & (PTE_R|PTE_W|PTE_X)) == 0){
      // this PTE points to a lower-level page table.
      uint64 child = PTE2PA(pte);
      freewalk((pagetable_t)child);
      pagetable[i] = 0;
    } else if(pte & PTE_V){
       //3. 调用freewalk之前，必须保证叶子pte全部被清理完毕，防止内存泄漏
      panic("freewalk: leaf");
    }
  }
  kfree((void*)pagetable);
}

```

#### 分配和释放用户进程内存

sbrk 用来增加或减少内存底层函数

uvmalloc：获取一页新的内存，并在页表项中建立起来到新分配页表的映射

uvmdealloc：调用uvmunmap来释放内存页

#### 父进程的整个地址空间全部复制到子进程中

uvmcopy：获取新的物理页，复制父进程物理页内容到新页，在新的子进程页表中建立映射关系



#### 内核态的数据和用户态数据拷贝

```c
// Copy from user to kernel.
// Copy len bytes to dst from virtual address srcva in a given page table.
// Return 0 on success, -1 on error.
//传入一个用户进程的页表，dst内核虚拟地址，srcva用户虚拟地址，len长度
//因为copyin是内核里执行的函数，所以必须软件模拟硬件，
//根据用户页表查找用户虚拟地址的物理地址，再拷贝到内核的虚拟地址上去
int copyin(pagetable_t pagetable, char *dst, uint64 srcva, uint64 len)
{
  uint64 n, va0, pa0;

  while(len > 0){
      //用户进程VA
    va0 = PGROUNDDOWN(srcva);
       //用户进程PA
    pa0 = walkaddr(pagetable, va0);
    if(pa0 == 0)
      return -1;
    n = PGSIZE - (srcva - va0);
    if(n > len)
      n = len;
      //把用户物理内存这一页的某个开始位置(pa0 + (dstva - va0))，长度为n，移动到dst
    memmove(dst, (void *)(pa0 + (srcva - va0)), n);

    len -= n;
    dst += n;
    srcva = va0 + PGSIZE;
  }
  return 0;
}


int copyout(pagetable_t pagetable, uint64 dstva, char *src, uint64 len)
{
  uint64 n, va0, pa0;

  while(len > 0){
      //用户进程VA
    va0 = PGROUNDDOWN(dstva);
      //用户进程PA
    pa0 = walkaddr(pagetable, va0);
    if(pa0 == 0)
      return -1;
    n = PGSIZE - (dstva - va0);
    if(n > len)
      n = len;
      //内存src地址的内容拷贝到用户物理内存的(pa0 + (dstva - va0))位置
    memmove((void *)(pa0 + (dstva - va0)), src, n);

    len -= n;
    src += n;
    dstva = va0 + PGSIZE;
  }
  return 0;
}
```

### Exec

exec是根据文件来初始化用户地址空间系统调用。

```c
int
exec(char *path, char **argv)
{
  // 一系列需要使用的变量声明
  char *s, *last;
  int i, off;
  uint64 argc, sz = 0, sp, ustack[MAXARG], stackbase;
  struct elfhdr elf;
  struct inode *ip;
  struct proghdr ph;
  pagetable_t pagetable = 0, oldpagetable;
  struct proc *p = myproc();	// 获取当前进程
  
  // begin_op是开启文件系统的日志功能
  // 每当进行一个与文件系统相关的系统调用时都要记录
  begin_op();
  
  // namei同样是一个文件系统操作，它返回对应路径文件的索引节点(index node)的内存拷贝
  // 索引节点中记录着文件的一些元数据(meta data)
  // 如果出错就使用end_op结束当前调用的日志功能
  if((ip = namei(path)) == 0){
    end_op();
    return -1;
  }
  
  // 给索引节点加锁，防止访问冲突
  ilock(ip);

  // Check ELF header
  // 读取ELF文件头，查看文件头部的魔数(MAGIC NUMBER)是否符合要求，这在xv6中有详细说明
  // 如果不符合就跳转到错误处理程序
  if(readi(ip, 0, (uint64)&elf, 0, sizeof(elf)) != sizeof(elf))
    goto bad;
  if(elf.magic != ELF_MAGIC)
    goto bad;
  
  // 创建一个用户页表，将trampoline页面和trapframe页面映射进去
  // 保持用户内存部分留白
  if((pagetable = proc_pagetable(p)) == 0)
    goto bad;

  // Load program into memory.
  // 译：将程序加载到内存中去
  // elf.phoff字段指向program header的开始地址，program header通常紧随elf header之后
  // elf.phnum字段表示program header的个数，在xv6中只有一条program header
  for(i=0, off=elf.phoff; i<elf.phnum; i++, off+=sizeof(ph)){
  	// 读取对应的program header， 出错则转入错误处理程序
    if(readi(ip, 0, (uint64)&ph, off, sizeof(ph)) != sizeof(ph))
      goto bad;
    // 如果不是LOAD段，则读取下一段，xv6中只定义了LOAD这一种类型的program header(kernel/elf.h)
    // LOAD意为可载入内存的程序段
    if(ph.type != ELF_PROG_LOAD)
      continue;
      
    // memsz：在内存中的段大小(以字节计)
    // filesz：文件镜像大小
    // 一般来说，filesz <= memsz，中间的差值使用0来填充
    // memsz < filesz就是一种异常的情况，会跳转到错误处理程序
    if(ph.memsz < ph.filesz)
      goto bad;
      
    // 安全检测，防止当前程序段载入之后地址溢出 
    if(ph.vaddr + ph.memsz < ph.vaddr)
      goto bad;
    
    // 尝试为当前程序段分配地址空间并建立映射关系
    // 这里正好满足了loadseg要求的映射关系建立的要求 
    // uvmalloc函数见完全解析系列博客(2)
    uint64 sz1;
    if((sz1 = uvmalloc(pagetable, sz, ph.vaddr + ph.memsz)) == 0)
      goto bad;
    
    // 更新sz大小，sz记录着当前已复制的地址空间的大小
    sz = sz1;
	
	// 如果ph.vaddr不是页对齐的，则跳转到出错程序
	// 这也是为了呼应loadseg函数va必须对齐的要求
    if((ph.vaddr % PGSIZE) != 0)
      goto bad;
    
    // 调用loadseg函数将程序段读入前面已经分配好的页面
    // 如读取不成功则跳转到错误处理程序
    if(loadseg(pagetable, ph.vaddr, ip, ph.off, ph.filesz) < 0)
      goto bad;
  } // 持续循环直到读完所有程序段
  
  // iunlockput函数实际上将iunlock和iput函数结合了起来
  // iunlock是释放ip的锁，和前面的ilock对应
  // iput是在索引节点引用减少时尝试回收节点的函数
  iunlockput(ip);
	
  // 结束日志操作，和前面的begin_op对应
  end_op();
  
  // 将索引节点置为空指针
  ip = 0;
  
  // 获取当前进程并读取出原先进程占用内存大小
  // myproc这个函数定义在kernel/proc.c中
  // 有关进程也是一个很大的话题，需要仔细研究
  p = myproc();
  uint64 oldsz = p->sz;

  // Allocate two pages at the next page boundary.
  // Use the second as the user stack.
  // 译：在当前页之后再分配两页内存
  // 并使用第二页作为用户栈
  // 这和xv6 book中展示的用户地址空间是完全一致的，可以参考一下
  // 在text、data段之后是一页guard page，然后是user stack
  sz = PGROUNDUP(sz);
  uint64 sz1;
  if((sz1 = uvmalloc(pagetable, sz, sz + 2*PGSIZE)) == 0)
    goto bad;
  sz = sz1;
  
  // 清除守护页的用户权限 
  uvmclear(pagetable, sz-2*PGSIZE);
  
  // sz当前的位置就是栈指针stack pointer的位置，即栈顶
  // stackbase是栈底位置，即栈顶位置减去一个页面
  sp = sz;
  stackbase = sp - PGSIZE;

  // Push argument strings, prepare rest of stack in ustack.
  // 在用户栈中压入参数字符串
  // 准备剩下的栈空间在ustack变量中
  // 读取exec函数传递进来的参数列表，直至遇到结束符
  for(argc = 0; argv[argc]; argc++) {
  	// 传入的参数超过上限，则转入错误处理程序
    if(argc >= MAXARG)
      goto bad;
    
    // 栈顶指针下移，给存放的参数留下足够空间
    // 多下移一个字节是为了存放结束符
    sp -= strlen(argv[argc]) + 1;
    sp -= sp % 16; // riscv sp must be 16-byte aligned，对齐sp指针
    
    // 如果超过了栈底指针，表示栈溢出了
    if(sp < stackbase)
      goto bad;
    
    // 使用copyout函数将参数从内核态拷贝到用户页表的对应位置
    if(copyout(pagetable, sp, argv[argc], strlen(argv[argc]) + 1) < 0)
      goto bad;
    
    // 将参数的地址放置在ustack变量的对应位置
    // 注意：ustack数组存放的是函数参数的地址(虚拟地址)
    ustack[argc] = sp;
  }
  // 在ustack数组的结尾存放一个空字符串，表示结束
  ustack[argc] = 0;

  // push the array of argv[] pointers.
  // 将参数的地址数组放入用户栈中，即将ustack数组拷贝到用户地址空间中
  // argc个参数加一个空字符串，一共是argc + 1个参数
  // 对齐指针到16字节并检测是否越界
  sp -= (argc+1) * sizeof(uint64);
  sp -= sp % 16;
  if(sp < stackbase)
    goto bad;
  
  // 从内核态拷贝ustack到用户地址空间的对应位置
  if(copyout(pagetable, sp, (char *)ustack, (argc+1)*sizeof(uint64)) < 0)
    goto bad;

  // arguments to user main(argc, argv)
  // argc is returned via the system call return
  // value, which goes in a0.
  // 译：用户main程序的参数
  // argc作为返回值返回，存放在a0中，稍后我们就会看到这个调用的返回值就是argc
  // sp作为argv，即指向参数0的指针的指针，存放在a1中返回
  // < 疑问：按照xv6 book所述，这里的栈里应该还有argv和argc的存放 >
  // < 这个地方留个坑，等研究完系统调用和陷阱的全流程之后再来解释 >
  p->trapframe->a1 = sp;

  // Save program name for debugging.
  // 译：保留程序名，方便debug
  // 将程序名截取下来放到进程p的name中去
  for(last=s=path; *s; s++)
    if(*s == '/')
      last = s+1;
  safestrcpy(p->name, last, sizeof(p->name));
    
  // Commit to the user image.
  // 译：提交用户镜像
  // 在此，首先将用户的原始地址空间保存下来，以备后面释放
  // 然后将新的地址空间全部设置好
  // epc设置为elf.entry，elf.entry这个地址里存放的是进程的开始执行地址
  // sp设置为当前栈顶位置sp
  oldpagetable = p->pagetable;
  p->pagetable = pagetable;
  p->sz = sz;
  p->trapframe->epc = elf.entry;  // initial program counter = main
  p->trapframe->sp = sp;          // initial stack pointer
  proc_freepagetable(oldpagetable, oldsz);

  return argc; // this ends up in a0, the first argument to main(argc, argv)
 
 // 错误处理程序所做的事：
 // 释放已经分配的新进程的空间
 // 解锁索引节点并结束日志记录，返回-1表示出错
 bad:
  if(pagetable)
    proc_freepagetable(pagetable, sz);
  if(ip){
    iunlockput(ip);
    end_op();
  }
  return -1;
}

```



