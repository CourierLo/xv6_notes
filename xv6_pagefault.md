## Lazy allocation
程序可能申请很多空间，但是它却不用。这种情况没必要每次都老老实实给它分配内存。
在`sys_sbrk`中，如果要申请内存，只增加p->sz而不再进行`growproc`，但是如果要缩减虚拟内存还是要`growproc`
在`usertrap()`中处理缺页异常
``` C
// page fault. 12 instruction / 13 load / 15 Store or AMO
// stval: 保存了缺页异常的虚拟地址
if(r_scause() == 13 || r_scause() == 15){
  uint64 faddr = r_stval();
  if(faddr < p->sz && faddr >= p->trapframe->sp){  // 检查地址是否合法, below user stack也不行
    char *mem = kalloc();
    if(mem == 0){
      p->killed = 1;
    } else {  
      memset(mem, 0, sizeof(mem));
      mappages(p->pagetable, PGROUNDDOWN(faddr), PGSIZE, (uint64)mem, PTE_U | PTE_W | PTE_V | PTE_R);  
    }
  } else {
    // not a legal addr
    p->killed = 1;
  }
} else {
  printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
  printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
  p->killed = 1;
}
```
对于不存在的页，`uvmunmap`和`uvmcopy`不需要panic，继续执行就可以了。对于`copyin`和`copyout`，他们同样有可能遇到由于懒分配而不存在的页，复制之前像`usertrap`那样分配一下就可以了。

`fork`的`uvmcopy`不再需要检测pte或page是否存在了，因为可能本来就没有

## COW(Copy On Write)实现一头奶牛
当Shell处理指令时，它会通过fork创建一个子进程。fork会创建一个Shell进程的拷贝，所以这时我们有一个父进程（原来的Shell）和一个子进程。Shell的子进程执行的第一件事情就是调用exec运行一些其他程序，比如运行echo。现在的情况是，fork创建了Shell地址空间的一个完整的拷贝，而exec做的第一件事情就是丢弃这个地址空间，并分配新的page来包含echo相关的内容。这里看起来有点浪费。

所以对于这个特定场景有一个非常有效的优化：当我们创建子进程时，与其创建，分配并拷贝内容到新的物理内存，其实我们可以直接共享父进程的物理内存page。所以这里，我们可以设置子进程的PTE指向父进程对应的物理内存page。

当然，再次要提及的是，我们这里需要非常小心。因为一旦子进程想要修改这些内存的内容，相应的更新应该对父进程不可见，因为我们希望在父进程和子进程之间有强隔离性，所以这里我们需要更加小心一些。为了确保进程间的隔离性，我们可以将这里的父进程和子进程的PTE的标志位都设置成只读的。

<img src="cow_example", style="zoom:70%">

在某个时间点，当我们需要更改内存的内容时，我们会得到page fault。因为父进程和子进程都会继续运行，而父进程或者子进程都可能会执行store指令来更新一些全局变量，这时就会触发page fault，因为现在在向一个只读的PTE写数据。
在得到page fault之后，我们需要拷贝相应的物理page。假设现在是子进程在执行store指令，那么我们会分配一个新的物理内存page，然后将page fault相关的物理内存page拷贝到新分配的物理内存page中，并将新分配的物理内存page映射到子进程。这时，新分配的物理内存page只对子进程的地址空间可见，所以我们可以将相应的PTE设置成可读写，并且我们可以重新执行store指令。实际上，对于触发刚刚page fault的物理page，因为现在只对父进程可见，相应的PTE对于父进程也变成可读写的了。

所以现在，我们拷贝了一个page，将新的page映射到相应的用户地址空间，并重新执行用户指令。重新执行用户指令是指调用userret函数（注，详见6.8），也即是lec06中介绍的返回到用户空间的方法。

cow lab实现：
1. 魔改`uvmcopy`，新增`uvmcow`, `fork`不再采用`uvmcopy`而用`uvmcow`
```C
int
uvmcow(pagetable_t old, pagetable_t new, uint64 sz){
  pte_t *pte;
  uint64 pa, i;
  uint flags;

  for(i = 0; i < sz; i += PGSIZE){
    if((pte = walk(old, i, 0)) == 0)
      panic("uvmcow: pte should exist");
    if((*pte & PTE_V) == 0)
      panic("uvmcow: page not present");
    
    // eliminate PTE_W flag
    *pte &= (~PTE_W);
    pa = PTE2PA(*pte);
    flags = PTE_FLAGS(*pte);
    if(mappages(new, i, PGSIZE, pa, flags) != 0){
      // kfree((void *)pa);  // 因为共用page，即使map不成功也不能free!
      goto err;
    }

    // update reference count and set two page as COW page
    pageref[(pa - KERNBASE) / PGSIZE]++;
    *pte |= PG_COW;

    pte = walk(new, i, 0);
    *pte |= PG_COW;
  }

  return 0;

 err:
  uvmunmap(new, 0, i / PGSIZE, 1);
  return -1;
}
```
使用pte的预留位，可以判断该页是否用来COW的。这里我选择第8位bit。`#define PG_COW (1L << 8)`

`walk`返回最后一级pte的指针，然后你可以用它来修改标志位。

这里我们使用一个页引用数数组`pageref`记录以下每个页被引用的数量，如果一个页引用数量归0，才可以真正释放掉。`pageref`大小为128 * 1024 * 1024 / 4096 * 4Byte.

当然了，为了不影响系统其他`kfree`，我们不妨在`kfree`进行引用数量判断
```C
// Free the page of physical memory pointed at by v,
// which normally should have been returned by a
// call to kalloc().  (The exception is when
// initializing the allocator; see kinit above.)
void
kfree(void *pa)
{
  struct run *r;

  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");

  int ref = ((uint64)pa - KERNBASE) / PGSIZE;
  // subtract reference count, if still > 0, return, else free current page
  // printf("kfree: No. %d page ref cnt %d, addr %p\n", ref, pageref[ref], pa);
  // acquire(&kmem.lock);
  pageref[ref] -= 1;
  if(pageref[ref] > 0){
    // release(&kmem.lock);
    return; 
  }

  // Fill with junk to catch dangling refs.
  memset(pa, 1, PGSIZE);

  r = (struct run*)pa;

  acquire(&kmem.lock);
  r->next = kmem.freelist;
  kmem.freelist = r;
  release(&kmem.lock);
}

// Allocate one 4096-byte page of physical memory.
// Returns a pointer that the kernel can use.
// Returns 0 if the memory cannot be allocated.
void *
kalloc(void)
{
  struct run *r;

  acquire(&kmem.lock);
  r = kmem.freelist;
  if(r)
    kmem.freelist = r->next;

  release(&kmem.lock);

  if(r){
    // r's ref = 1
    uint64 ref = ((uint64)r - KERNBASE) / PGSIZE;
    // printf("kalloc: No. %d page, addr %p.\n", ref, r);
    pageref[ref] = 1;
  }

  if(r)
    memset((char*)r, 5, PGSIZE); // fill with junk

  return (void*)r;
}
```

跨越多个文件，如何对`pageref`上同一把锁完成增减？用一个函数完成即可
```C
// 为 pa 的引用计数增加 1
void krefpage(void *pa) {
  acquire(&pgreflock);
  PA2PGREF(pa)++;
  release(&pgreflock);
}
```

发生缺页异常的时候，在`usertrap`分配新的页即可
```C
void
usertrap(void)
{
  int which_dev = 0;

  if((r_sstatus() & SSTATUS_SPP) != 0)
    panic("usertrap: not from user mode");

  // send interrupts and exceptions to kerneltrap(),
  // since we're now in the kernel.
  w_stvec((uint64)kernelvec);

  struct proc *p = myproc();
  
  // save user program counter.
  p->trapframe->epc = r_sepc();
  
  if(r_scause() == 8){
    // system call

    if(p->killed)
      exit(-1);

    // sepc points to the ecall instruction,
    // but we want to return to the next instruction.
    p->trapframe->epc += 4;

    // an interrupt will change sstatus &c registers,
    // so don't enable until done with those registers.
    intr_on();

    syscall();
  } else if((which_dev = devintr()) != 0){
    // ok
  } else {
    // deal with write page fault here
    // three codes in scause representing page fault
    // 12 instruction / 13 load / 15 Store or AMO
    if(r_scause() == 15){
      // printf("usertrap(): scause 15\n");
      uint64 faddr = r_stval();   // va of page fault
      pte_t *pte;
      uint64 pa;
      uint flags;
      int ref;
      char *mem;

      if(faddr < 0 || faddr >= p->sz){
        p->killed = 1;
        printf("usertrap(): faddr %p illegal.\n", faddr);
        printf("p->sp: %p, p->sz: %p\n", p->trapframe->sp, p->sz);
        exit(-1);
      }

      if((pte = walk(p->pagetable, faddr, 0)) == 0)
        panic("usertrap(): pte should exist");
      
      pa = PTE2PA(*pte);
      flags = PTE_FLAGS(*pte) | PTE_W;
      ref = ((uint64)pa - KERNBASE) / PGSIZE;
      if (*pte & PG_COW){
        if((mem = kalloc()) == 0){
          p->killed = 1;
          exit(-1);
        }

        memmove(mem, (char*)pa, PGSIZE);
        // 如果直接用mappages的话，原本的*pte就没去掉
        pageref[ref]--;
        *pte = PA2PTE(mem);
        *pte |= flags;

        if(pageref[ref] <= 0) kfree((void *)pa); 
      } else {
        p->killed = 1;
      }
    } else {
      printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
      printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
      p->killed = 1;
    }
  }

  if(p->killed)
    exit(-1);

  // give up the CPU if this is a timer interrupt.
  if(which_dev == 2)
    yield();

  usertrapret();
}
```

`copyout`也是有可能写入到用户的COW pages的，所以得把这些COW pages复制一份再写入才行
```C
int
copyout(pagetable_t pagetable, uint64 dstva, char *src, uint64 len)
{
  uint64 n, va0, pa0;
  pte_t *pte;
  char *mem = 0;
  int flags;
  uint ref;

  while(len > 0){
    va0 = PGROUNDDOWN(dstva);
    // pa0 = walkaddr(pagetable, va0);
    
    if((pte = walk(pagetable, va0, 0)) == 0)  return -1;

    pa0 = PTE2PA(*pte);
    if(*pte & PG_COW) {
      // COW page need to reallocate when ref > 1
      ref = (pa0 - KERNBASE) / PGSIZE;
      if(pageref[ref] == 1) {
        *pte &= ~(PG_COW);
        *pte |= PTE_W;
      } else if(pageref[ref] > 1){
        if((mem = kalloc()) == 0) return -1;
        memmove(mem, (char*)pa0, PGSIZE);
        flags = PTE_FLAGS(*pte) | PTE_W;
        pageref[ref]--;
        if(!pageref[ref]){
          printf("copyout(): pageref is 0!\n");
        }
        *pte = (PA2PTE(mem) | flags);

        pa0 = PTE2PA(*pte);
        // mappages(pagetable, va0, PGSIZE, (uint64)mem, flags);
        // pageref[((uint64)mem - KERNBASE) / PGSIZE] = 1;
      } else if(*pte){
        panic("copyout(): Use a freed page.");
      }
    }

    // walk again
    // if((pte = walk(pagetable, va0, 0)) == 0)  return -1;
    if(pa0 == 0)
      return -1;
    n = PGSIZE - (dstva - va0);
    if(n > len)
      n = len;
    memmove((void *)(pa0 + (dstva - va0)), src, n);

    len -= n;
    src += n;
    dstva = va0 + PGSIZE;
  }
  return 0;
}
```