## 为进程添加内核页表
实验目标：allow the kernel to directly dereference user pointers.

流程：
1. proc结构体添加内核页表指针
2. 模仿kernel/vm.c的kvminit函数，用类似原本内核页表初始化的方式，初始化进程的内核页表-`kvmcreate`
3. `allocproc`(proc.c)的时候，照猫画虎创建页表之余，把内核栈也映射到进程内核页表之中。`kwalkaddr`+`mappages`
4. 使用`copypgtbl`(vm.c)将用户进程页表内容按照虚拟地址范围一并复制到用户内核进程页表中: `userinit`,`growproc`,`fork`,`exec`(proc.c, exec.c)，让用户内核页表随着sz增长而增长
5. `freeproc`的时候，对于构建的用户内核页表，释放该页表本身即可，不要释放指向的页！
6. `scheduler`切换进程的时候，切换他们的内核页表

`proc_pagetable`: Create a user page table for a given process, with no user memory, but with trampoline pages.

## 血泪教训：
**不要**修改`uvmalloc`,`uvmdealloc`,`mappages`,`uvmunmap`等函数！你可能一开始想着修改这些函数，以便用户用户内核页表随着用户页表同时增长同时释放，这样做会非常危险！！我当初修改这些函数以后，和各种稀奇古怪的bugs搏斗了三四天！最后程序的症状是：除了exec，其他test都能过。但是exec失败是非常严重的失败，后来改不动了，干脆换成只复制用户进程页表的方式，在进程初始化阶段如`exec`，`fork`,`userinit`, 进程内存增长阶段`growproc`复制用户进程页表内容到用户内核页表。`allocproc`略有不同，只负责创建页表(类似`kvminit`)，并且把内核栈也映射到进程内核页表之中。

## 还有什么要做的
- 如果程序`sbrk`缩减用户虚拟地址空间，页表映射并未删除！不去除的问题在于内核页表多了一些用户页表不该访问的映射，正常情况下用户不会去访问那些他页表里没有的地址。但是如果（只是传个地址给内核）访问了，内核一旦检查自己是否有这些页面，结果发现有（可能sbrk缩小之后空出来的物理地址已经被其他进程申请走了）就访问到其他进程的数据了，破坏了隔离。
- `exec`的时候，理论上`p->krl_pagetable`也要释放掉属于用户页表的映射！如果没有释放，p->krl_pagetable还是残留了一些之前user页表项。一旦访问到这些数据，实际就可能会读到其他进程的数据。

实测你不去unmap掉之前krl_pagetable, 也能通过测试，test没有测试这种情况。但是我的建议还是随着用户虚拟内存分配和释放同时修改用户内核页表。

`exec`的时候，你可以：
```C
uvmunmap(p->krl_pagetable, 0, PGROUNDUP(p->sz) / PGSIZE, 0);
```
`growproc`缩减的时候，你可以(仿照`uvmdealloc`，但是只去掉映射)：
```C
uint oldsz = sz;
uint newsz = sz + n;
if(PGROUNDUP(newsz) < PGROUNDUP(oldsz)){
  int npages = (PGROUNDUP(oldsz) - PGROUNDUP(newsz)) / PGSIZE;
  uvmunmap(p->krl_pagetable, PGROUNDUP(newsz), npages, 0);
}
```
记得把`uvmunmap`的not mapped的panic注释掉。

留下个疑问：内核栈是用来干吗的？