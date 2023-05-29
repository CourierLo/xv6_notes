# Lab2: System calls & Lab4: Traps
Lab2主要是让学生上手写两个系统调用，让学生亲自感受xv6系统调用的流程。Lab4让你手动强制kernel每隔几个ticks调用一次`periodic`函数(通过调用`sigalarm`&`sigreturn`)，感受一下内核是如何保护现场以及恢复现场的。

## 系统调用流程纵览
`user/user.h`:		用户态程序调用跳板函数 `trace()`

`user/usys.S`:		跳板函数 `trace()` 使用 CPU 提供的 `ecall` 指令，进入内核态

`kernel/syscall.c`:	到达内核态统一系统调用处理函数 `syscall()`，所有系统调用都会跳到这里来处理。

`kernel/syscall.c`:	`syscall()` 根据跳板传进来的系统调用编号，查询 `syscalls[]` 表，找到对应的内核函数并调用

`kernel/sysproc.c`:	到达 `sys_trace()` 函数，执行具体内核操作

看似繁琐的系统调用流程主要目的在于实现用户态和内核态的隔离。

## user的跳板函数(usys.pl生成)
```pl
sub entry {
    my $name = shift;
    print ".global $name\n";
    print "${name}:\n";
    print " li a7, SYS_${name}\n";
    print " ecall\n";
    print " ret\n";
}
	
entry("fork");
```
每个`entry`(比如`fork`)其实相当于提供给用户态的系统调用接口，它们是一段用perl脚本生成的汇编代码。用户一旦调用这些函数，`SYS_${name}`(系统调用编号)便会被加载到寄存器`a7`，`ecall`进入内核态以后，我们便可以根据寄存器`a7`的内容读出用户需要什么系统调用。 

当然咯，做到Lab2的时候你可能直接认为`ecall`之后就直接到了syscall.c的`syscall`了。其实并不是，在Lab4中你会认识到用户空间和内核空间切换是怎样的。

发生以下这几种情况，操作系统需要从用户态切换到内核态：
- 用户程序执行系统调用
- 用户程序出现类似缺页，运算除0的错误
- 一个设备触发了中断使得当前程序运行需要响应内核设备驱动

用户空间到内核空间的切换通常称为"trap", trap的安全隔离和性能十分重要。很多应用程序要么因为系统调用，要么缺页，需要频繁地切换到内核中。因此，trap的机制要尽量简单。空间的切换需要硬件状态的配合。

用户可以使用32个用户寄存器。其中具有特殊功能的寄存器：
- SP: stack pointer

此外又有一些具有特殊功能的寄存器：
- PC: program counter程序计数器
- MODE：标志位表明是supervisor mode还是user mode
- SATP: 指向page table的**物理**内存地址
- STVEC：指向内核中处理trap的指令起始地址(trap.c sets stvec to userver)
- SEPC：trap过程中保存了PC的值
- SSRATCH：指向进程的p->trapframe

**这些寄存器表明了执行系统调用时计算机的状态**。

在trap的最开始，CPU的所有状态都设置为运行用户代码而不是内核代码。trap过程中需要更改这些寄存器的状态，以便运行系统内核中的C程序。那么，trap要干些什么呢？

- 保存32个用户寄存器用于恢复现场。我们希望内核响应中断后，在用户程序无感知的情况下恢复用户代码执行。
- 保存PC。用户程序要在运行中断的位置继续执行用户程序。
- user mode改为supervisor mode。我们要在内核中使用特权指令。
- 修改SATP，使它指向内核页表。
- 堆栈寄存器指向位于内核的一个地址，因为我们需要使用一个堆栈调用内核的C函数。
- 一旦准备万全，硬件状态也适合在内核中使用，需要跳入内核的C代码。

为了实现安全隔离性，状态切换不可以依赖32个用户寄存器，以免不安全数据影响内核。要时刻防止用户代码欺骗内核。进入supervisor mode之后，我们可以读写SATP、STVEC、SEPC、SSCRATCH，并且可以使用PTE_U=0的PTE。除此之外supervisor mode也不能干别的了。需要特别指出的是，supervisor mode中的代码**并不能读写任意物理地址**。在supervisor mode中，就像普通的用户代码一样，也需要通过page table来访问内存。如果一个虚拟地址并不在当前由SATP指向的page table中，又或者SATP指向的page table中PTE_U=1，那么supervisor mode不能使用那个地址。所以，即使我们在supervisor mode，我们还是受限于当前page table设置的虚拟地址。

回到`ecall`这条指令。`ecall`会干三件事：
- 切换到具有supervisor mode的内核中
- 保存$pc到$sepc
- 跳转到$stvec保存的地址`uservec`，也就是trampoline page的开头

注意，trampoline page包含了内核的trap处理代码。`ecall`并不会切换pagetable，这是`ecall`指令的一个非常重要的特点。所以这意味着，trap处理代码必须存在于每一个user pagetable中。又因为`ecall`并不会切换pagetable，我们需要在user pagetable中的某个地方来执行最初的内核代码。而这个trampoline page，是由内核小心的映射到每一个user pagetable中，以使得当我们仍然在使用user pagetable时，内核在一个地方能够执行trap机制的最开始的一些指令。`ecall`只完成尽量少必须要完成的工作.

RISC-V中，supervisor mode下的代码不允许直接访问物理内存。

`uservec`需要干以下这么几件事情：
- 保存32个用户寄存器(到trapframe page, 0x3ffffffe000)
- 切换到内核页表
- 创建或者找到kernel stack，将sp指向那个kernel stack，为C代码提供栈
- 跳转到C代码合理位置

切换到内核页表后，由于内核页表和用户页表的trampoline page的映射是一样的，所以代码不会崩溃。之所以叫trampoline page，是因为你某种程度在它上面“弹跳”了一下，然后从用户空间走到了内核空间。

- csrrw a0, sscratch, a0
在进入到user space之前，内核会将trapframe page的地址保存在这个寄存器中，也就是0x3fffffe000这个地址。在内核前一次切换回用户空间时，内核会执行set sscratch指令，将这个寄存器的内容设置为0x3fffffe000，也就是trapframe page的虚拟地址。

执行完`uservec`后，再跳转到trap.c中的`usertrap()`中。`usertrap()`会判断出这是一个系统调用，于是调用`syscall()`函数，通过查找函数表单，执行对应的`sys_xxx`. 执行完回到`syscall`后，会调用函数`usertrapret`(trap.c)。这个函数完成了部分方便在C代码中实现的返回用户态的工作。除此之外，最终还有一些工作只能在汇编语言中完成。这部分工作通过汇编语言实现，并且存在于trampoline.S文件中的`userret`函数中。最终，在这个汇编函数中会调用机器指令返回到用户空间，并且恢复`ecall`之后的用户程序的执行。

<img src="./pic/syscall">

## Alarm
用`sigalarm`注册每隔几个ticks报一个alarm警告。时钟中断时，如果已经经过了这么多个ticks，转去执行`periodic`函数输出"alarm"。需要手动保存现场，将下次$pc指向fn，也就是periodic初始地址。同一个进程使用同一个栈，理论上进入`periodic`是可以直接读到j的值的，不知怎地-
```C
// usertrap in trap.c
// somehow, in alarmtest, j lies on 0x2f9c, but test1's sp points at 0x2f90
// it's very strange that j lies below sp, so in periodic (0x2: sd ra,8(sp)) will overwrite j
// we must minus sp before we jump to periodic, so as to protect j's value
p->trapframe->sp -= 16; 
```
这个bug在网上其他博客我没看到有别人遇到过，非常神奇，`test1`中j理应在栈指针的上方，难道是qemu有问题？不得而知。

```C
// alarmtest.c
void
test1()
{
  int i;
  int j;

  printf("test1 start\n");
  count = 0;
  j = 0;
  sigalarm(2, periodic);
  for(i = 0; i < 500000000; i++){
    if(count >= 10)
      break;
    foo(i, &j);
  }
  if(count < 10){
    printf("\ntest1 failed: too few calls to the handler\n");
  } else if(i != j){
    // the loop should have called foo() i times, and foo() should
    // have incremented j once per call, so j should equal i.
    // once possible source of errors is that the handler may
    // return somewhere other than where the timer interrupt
    // occurred; another is that that registers may not be
    // restored correctly, causing i or j or the address of j
    // to get an incorrect value.
    printf("i, j: %d %d\n", i, j);
    printf("\ntest1 failed: foo() executed fewer times than it was called\n");
  } else {
    printf("test1 passed\n");
  }
}
```

```S
void
periodic()
{
   0:	1141                	addi	sp,sp,-16
   2:	e406                	sd	ra,8(sp)  # overwrite j!!!
   4:	e022                	sd	s0,0(sp)
   6:	0800                	addi	s0,sp,16
  count = count + 1;
   8:	00001797          	auipc	a5,0x1
   c:	d387a783          	lw	a5,-712(a5) # d40 <count>
  10:	2785                	addiw	a5,a5,1
  12:	00001717          	auipc	a4,0x1
  16:	d2f72723          	sw	a5,-722(a4) # d40 <count>
  printf("alarm!\n");
  1a:	00001517          	auipc	a0,0x1
  1e:	b5e50513          	addi	a0,a0,-1186 # b78 <malloc+0xe4>
  22:	00001097          	auipc	ra,0x1
  26:	9b4080e7          	jalr	-1612(ra) # 9d6 <printf>
  sigreturn();
  2a:	00000097          	auipc	ra,0x0
  2e:	6cc080e7          	jalr	1740(ra) # 6f6 <sigreturn>
}
  32:	60a2                	ld	ra,8(sp)
  34:	6402                	ld	s0,0(sp)
  36:	0141                	addi	sp,sp,16
  38:	8082                	ret

000000000000003a <slow_handler>:
  }
}
```

输出完"alarm"，使用`sigreturn`让恢复到test1的现场。