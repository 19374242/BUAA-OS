# 基本概念

**1.在用户态下，用户进程不能访问系统的内核空间，也就是说它不能存取内核使用的内存数据，也不能调用内核函数， 这一点是由CPU的硬件结构保证的。然而，用户进程在特定的场景下往往需要执行一些只能由内核完成的操作，如 操作硬件、动态分配内存，以及与其他进程进行通信等。允许在内核态执行用户程序提供的代码显然是不安全的，因此操作系统设计了一系列内核空间 中的函数，当用户进程需要进行这些操作时，会引发特定的异常以陷入内核态，由内核调用对应的函数，从而安全地为用户进程提供受限的系统级操作， 我们把这种机制称为系统调用 。**

2.在 lab4 中，我们需要实现上述的系统调用机制，并在此基础上实现进程间通信（IPC）机制和一个重要的进程创建机制 fork。 在fork部分的实验中，我们会介绍一种被称为写时复制（COW）的特性，以及与其相关的页写入异常处理。

3.MOS 中的用户空间包括 kuseg， 而内核空间主要包括 kseg0 和 kseg1。每个进程的用户空间通常通过页表映射到不同的物理页，而内核空间则直接映射到固定的物理页以及外部硬件设备。 CPU 在内核态下可以访问任何内存区域，对物理内存等硬件设备有完整的控制权，而在用户态下则只能访问用户空间。可以认为内核是存在于所有进程地址空间中的一段代码

4.syscall是系统调用的意思，处理一些只能由内核来完成的操作（如读写设备、创建进程、IO等），通过执行syscall指令，用户进程可以陷入到内核态，请求内核提供的服务。内核将自己所能够提供的服务以系统调用的方式提供给用户空间，以供用户程序完成一些特殊的系统级操作。这样一来，所有的特殊操作就全部在操作系统的掌控之中了，因为用户程序只能将服务相关的参数交予操作系统，而实际完成需要特权的操作是由内核经过重重检查后执行的，所以系统调用可以确保系统的安全性。

5.三种函数：
syscall_……：用户空间内的函数，与sys_……成对存在

msyscall：设置**系统调用号**并让系统陷入内核态的函数

sys_……：内核函数

我们先来整理一下在MOS中进行系统调用的流程：

1. 调用一个封装好的用户空间的库函数（如writef）
2. 调用用户空间的syscall_* 函数
3. 调用msyscall，用于陷入内核态
4. 陷入内核，内核取得信息，执行对应的内核空间的系统调用函数（sys_*）
5. 执行系统调用，并返回用户态，同时将返回值“传递”回用户态
6. 从库函数返回，回到用户程序调用处

### 进程间通信机制(IPC)

各个进程的地址空间是相互独立的。相信你在实现内存管理的时候已经深刻体会到了这一点， 每个进程都有各自的地址空间，这些地址空间之间是相互独立的，同一个虚拟地址但却可能在不同进程下对应不同的物理页面，自然 对应的值就不同。因此，要想传递数据， 我们就需要想办法把一个地址空间中的东西传给另一个地址空间。

所有的进程都共享了内核所在的2G空间。对于任意的进程，这2G都是一样的。 因此，想要在不同空间之间交换数据，我们就可以借助于内核的空间来实现。也就是说，发送方进程可以将数据以系统调用的形式存放在内核空间中，接收方进程同样以系统调用的方式在内核找到对应的数据，读取并返回。那么，我们把要传递的消息放在哪里比较好呢？ 发送和接受消息和进程有关，消息都是由一个进程发送给另一个进程的。内核里什么地方和进程最相关呢？——进程控制块！

```
struct Env {
    // Lab 4 IPC
    u_int env_ipc_value;            // data value sent to us
    u_int env_ipc_from;             // envid of the sender
    u_int env_ipc_recving;          // env is blocked receiving
    u_int env_ipc_dstva;            // va at which to map received page
    u_int env_ipc_perm;             // perm of page mapping received
};
```

|  env_ipc_value  | 进程传递的具体数值                           |
| :-------------: | -------------------------------------------- |
|  env_ipc_from   | 发送方的进程ID                               |
| env_ipc_recving | 1：等待接受数据中；0：不可接受数据           |
|  env_ipc_dstva  | 接收到的页面需要与自身的哪个虚拟页面完成映射 |
|  env_ipc_perm   | 传递的页面的权限位设置                       |

### 系统调用fork

1.内核通过 `env_alloc` 函数创建一个进程。但如果要让一个进程创建一个进程， 就像是父亲与孩子那样，我们就需要基于系统调用，引入新的 fork 机制了。

2.在新的进程中，这一 fork() 调用的返回值为 0，而在旧进程，也就是所谓的父进程中，同一调用的返回值是子进程的进程 ID（MOS 中的 env_id），且一定大于 0。

- 只有父进程会执行 fork 之前的代码段。
- 父子进程同时开始执行 fork 之后的代码段。
- fork 在不同的进程中返回值不一样，在子进程中返回值为 0，在父进程中返回值不为 0，而为子进程的 pid（Linux 中进程专属的 id，类似于 MOS 中的 envid）。
- 父进程和子进程虽然很多信息相同，但他们的进程控制块是不同的。

3.父子进程共享物理内存是有前提条件的：共享的物理内存不会被任一进程修改。那么，对于那些父进程或子进程修改的内存我们又该如何处理呢？ 这里我们引入一个新的概念——写时复制（Copy On Write，简称 COW)。COW 类似于一种对虚拟页的保护机制，通俗来讲就是在 fork 后的父子进程中有**修改**内存（一般是数据段或栈）的行为发生时，内核会捕获到一种**页写入异常**，并在异常处理时为**修改内存的进程**的地址空间中相应地址分配新的物理页面。一般来说，子进程的代码段仍会共享父进程的物理空间，两者的程序镜像也完全相同。在这样的保护下，用户程序可以在行为上认为 fork 时父进程中内存的状态被完整复制到了子进程中，此后父子进程可以独立操作各自的内存。

写时复制：（页写入异常）

1. 用户进程触发页写入异常，跳转到`handle_mod`函数，再跳转到`page_fault_handler`函数。
2. `page_fault_handler`函数负责将当前现场保存在异常处理栈中，并设置epc寄存器的值，使得从中断恢复后能够跳转到env_pgfault_handler域存储的异常处理函数的地址。
3. 退出中断，跳转到异常处理函数中，这个函数首先跳转到`pgfault`函数（定义在fork.c中）进行写时复制处理，之后恢复事先保存好的现场，并恢复sp寄存器的值，使得子进程恢复执行。

# lab4

1.*user/syscall_wrap.S*中的`msyscall`函数为陷入内核态函数，有5个参数，这些参数是系 统调用时需要传递给内核的参数。而为了方便传递参数，我们采 用的是最多参数的系统调用所需要的参数数量（`syscall_mem_map`函数需要5个参数）。

2.在通过 syscall 指令陷入内核态后，处理器将PC寄存器指向一个内核中固定的异常处理入口。在初始化异常向量表时，trap_init 函数将系统调用这一异常类型的处理入口设置为 *lib/syscall.S*中handle_sys 函数，我们需要在lib/syscall.S中实现该函数。**多于四个的参数会被放到内存中，而这个空间是存在于用户态的**，因此**我们需要在内核中将这些参数转移到内核空间内**，这步工作需要在`handle_sys()`函数的汇编代码实现了。***<u>`handle_sys`函数通过分析传入的参数来找到具体的系统调用目标函数，并将传入的参数放到寄存器中，然后进入目标函数（下为具体调用函数）。</u>***

3.int sys_mem_alloc(int sysno,u_int envid, u_int va, u_int perm) 给进程分配内存（给该程序所允许的虚拟内存空间 显式地分配实际的物理内存）

4.*lib/syscall_all.c中的int sys_mem_map(int sysno,u_int srcid, u_int srcva, u_int dstid, u_int dstva, u_int perm)函数*：将源进程地址空间中的相应内存映射到目标进程的相应 地址空间的相应虚拟内存中去。换句话说，此时两者共享一页物理内存。

5.*lib/syscall_all.c中的int sys_mem_unmap(int sysno,u_int envid, u_int va)函数*：解除某个进程地址空间 虚拟内存和物理内存之间的映射关系。

6.*lib/syscall_all.c中的void sys_yield(void)函数*：实现用户进程对CPU的放弃， 从而调度其他的进程。

7.*lib/syscall_all.c*中`sys_ipc_recv(int sysno,u_int dstva)`函数用于接受消息。sys_ipc_can_send(int sysno, u_int envid, u_int value, u_int srcva, u_int perm)函数用于发送消息。

### 系统调用fork

1.在我们的 MOS 操作系统实验中，进程调用 fork 时，其所有的可写入的内存页面，都需要通过设置页表项标志位 PTE_COW 的方式被保护起来。 无论父进程还是子进程何时试图写一个被保护的页面，都会产生一个页写入异常，而在其处理函数中，操作系统会进行**写时复制**，把该页面重新映射到一个新分配的物理页中，并将原物理页中的内容复制过来，同时取消虚拟页的这一标志位。

2.*`sys_env_alloc` 函数*：创建子进程。

​	CP0是一组寄存器，至少包括（SR 状态寄存器，包括中断引脚使能，其他 CPU 模式等位域，Cause 记录导致异常的原因和EPC异常结束后程序恢复执行的位置 ）

​	e->env_tf.pc = e->env_tf.cp0_epc;指子进程的现场中的程序计数器（PC）应该被设置为从内核态返回后的地址，也就是使它陷入异常的 syscall 指令的后一条指令的地址（异常结束后程序恢复执行的位置 ）。

​	e->env_tf.regs[2] = 0; // $v0 <- return value  对应这个系统调用本身是需要一个返回值的，我们希望系统调用在内核态返回的 envid 只传递给父进程，对于子进程则在恢复现场时用 0 覆盖系统调用原来的返回值。（即子进程返回0）

3.*user/fork.c* 中fork函数：对于 fork 后的子进程，它具有了一个与父亲不同的进程控制块，因此在子进程第一次被调度的时候（当然这时还是在fork函数中）需要对 env 指针进行更新，使其仍指向当前进程的控制块。

4.*user/fork.c 中的 `duppage` 函数*：将父进程地址空间中需要与子进程共享的页面映射给子进程。（PPN(va)=VPN(va)=va>>12）

​	PTE_R是可写而不是可读。。。。

​	进程id是0代表当前进程，在envi2env内有写

​	如果可写且不共享，标记 PTE_COW （写时复制）

​	在syscall_mem_map(0, addr, *envid*, addr, perm);中只修改了子进程的perm。所以需要syscall_mem_map(0, addr, 0, addr, perm);修改父进程perm。

5.`page_fault_handler`函数负责将当前现场保存在异常处理栈中，并设置epc寄存器的值，使得从中断恢复后能够跳转到env_pgfault_handler域存储的异常处理函数的地址。

6.user/fork.c 中的 `pgfault` 函数：处理中断

### exam 4-1

> 俺们这个MOS进行系统调用时，一次只能传6个参数，现在要syscall_*和msyscall要传参数的个数不确定了，并且保证第二个参数就是要传入的总的参数个数。

背景 常用的操作系统内核如Linux、XNU等，系统调用的参数个数少至0个、多至十多个不等。系统调用参数的传递往往适配参数的个数以避免性能的损失。 下面我们将尝试实现这一方式的系统调用参数传递。

#### 题目要求（第一部分）

现在我们要重新定义系统调用的参数传递规则(下面的参数顺序是用户态调用syscall指令时的顺序）

1. 第一个参数为系统调用号（和之前含义相同）

2. 第二个参数为此系统调用参数个数

3. 之后的参数依次是这个系统调用的参数（个数与第二个参数一致） 用户态按着这个新规则的系统调用实现方式为：

   ```
   int syscall_mem_map(u_int srcid, u_int srcva, u_int dstid, u_int dstva, u_int perm)
   {
       return msyscall(SYS_mem_map, 5, srcid, srcva, dstid, dstva, perm);
   }
   ```

同学们需要对已有的用户态系统调用均按此规则修改。 内核态处理系统调用参数，合理操作寄存器与栈，以保证在原有lib/syscall_all.c中代码不更改的情况下可以使系统调用机制正常运行。

#### 题目要求（第二部分）

在基础测试的基础上，实现一个新的系统调用`syscall_ipc_can_multi_send`，支持同时向5个进程通信，其用户态函数接口如下，请在`user/lib.h`中声明：

```
int syscall_ipc_can_multi_send(u_int value, u_int srcva, u_int perm, u_int envid_1, u_int envid_2, u_int envid_3, u_int envid_4, u_int envid_5);
```

其中value，srcva，perm的含义和`syscall_ipc_can_send`相同，后面的参数为要发送给的各个进程的进程号 系统调用号为`SYS_ipc_can_multi_send`，请在`unistd.h`中添加相关定义 需要在`user/syscall_lib.c`添加并实现用户态函数`syscall_ipc_can_multi_send`

需要在`lib/syscall_all.c`添加并实现内核态函数`sys_ipc_can_multi_send`。

对于该系统调用的行为规范如下：

1. 若接收的进程env_ipc_recving不全为1，则返回-E_IPC_NOT_RECV，
2. 若全为1，则发送消息

必须为原子操作，不可使用ipc_send函数来完成，必须要实现sys_ipc_can_multi_send函数 成功时返回0

修改user/lib.h中的msyscall函数声明，可以参考printf 修改user/syscall_lib.c，使每个系统调用都符合新规则 修改lib/syscall.S，增加拷贝多个参数的循环（重点） 修改include/unistd.h，增加系统调用号的定义 汇编内的跳转注意延迟槽 不支持mult,div等乘除指令，请不要使用 内核态参数拷贝时注意使用s0-s7保存栈空间 函数调用等寄存器使用规则参考指导书lab1相关内容。 系统调用寄存器使用规则参考指导书lab4相关内容。

你需要修改的文件有： user/lib.h user/syscall_lib.c lib/syscall_all.c lib/syscall.S include/unistd.h 该部分不允许修改的文件： user/syscall_wrap.S 该部分不允许新建文件

#### 解答

建议先看这一篇：[lab4-笔记](https://github.com/rfhits/Operating-System-BUAA-2021/blob/main/4-lab4/笔记.md)

直接放自己syscall.S。

```
/*** exercise 4.2 ***/
NESTED(handle_sys,TF_SIZE, sp)
    SAVE_ALL                            // Macro used to save trapframe
    CLI                                 // Clean Interrupt Mask
    nop
    .set at                             // Resume use of $at

    // TODO: Fetch EPC from Trapframe, calculate a proper value and store it back to trapframe.
        lw t0, TF_EPC(sp)
        addiu t0, t0, 4
        sw t0, TF_EPC(sp)

    // TODO: Copy the syscall number into $a0.
    lw a0, TF_REG4(sp)

    addiu   a0, a0, -__SYSCALL_BASE     // a0 <- relative syscall number
    sll     t0, a0, 2                   // t0 <- relative syscall number times 4
    la      t1, sys_call_table          // t1 <- syscall table base
    addu    t1, t1, t0                  // t1 <- table entry of specific syscall
    lw      t2, 0(t1)                   // t2 <- function entry of specific syscall

    lw      t0, TF_REG29(sp)            // t0 <- user's stack pointer

        // t2: func entry, t0 = sp_user

        // 1. store the first 4 arg into reg, not include cnt, the 5th arg may in stack if exists
        lw      a0, TF_REG4(sp)
        lw      a1, TF_REG6(sp)
        lw      a2, TF_REG7(sp)

        lw      s1, TF_REG5(sp) // s1 = cnt

        // 2. a3 need to store 5th arg, requires cnt >= 3
        li              t3, 3
        addiu   t0, t0, 16 // user_sp += 16, at stack arg field

        blt     s1, t3, STORE_a3_END
        nop
        lw              a3, 0(t0)
        addiu   t0, t0, 4 // user_sp += 4, at stack arg field
STORE_a3_END:

        li              t3, 4 // t3 = 4, cnt >= 4, stack has args

        // 3. adjust sys_sp place
        addiu   t4, s1, 1 // t4 = cnt + 1
        sll             s2, t4, 2 // s2 = (cnt + 1) * 4

        // just as mips promise, all args num is cnt + 1, save such **s2** places for them
        // but here is a bug, args num least is 4, cnt may less then 3
        // u may need to fix this, threat
        subu    sp, sp, s2 // sys_sp down, sace place for stack arg
        addiu   t5, sp, 16 // where put stack args, t5


// now, copy user_sp args into sys_sp stack field
COPY_ARG_BEG:
        bgt             t3, s1, COPY_ARG_END // t3 = 4, begin
        nop
        lw              t4, 0(t0)
        addiu   t0, t0, 4
        sw              t4, 0(t5)
        addiu   t5, t5, 4
        addiu   t3, t3, 1
        j               COPY_ARG_BEG
        nop
COPY_ARG_END:

        jalr    t2
        nop

        // back, return saved args space
        addu sp, sp, s2

    sw      v0, TF_REG2(sp)             // Store return value of function sys_* (in $v0) into trapframe

    j       ret_from_exception          // Return from exeception
    nop
END(handle_sys)

sys_call_table:                         // Syscall Table
.align 2
    .word sys_putchar
    .word sys_getenvid
    .word sys_yield
    .word sys_env_destroy
    .word sys_set_pgfault_handler
    .word sys_mem_alloc
    .word sys_mem_map
    .word sys_mem_unmap
    .word sys_env_alloc
    .word sys_set_env_status
    .word sys_set_trapframe
    .word sys_panic
    .word sys_ipc_can_send
    .word sys_ipc_recv
    .word sys_cgetc
    .word sys_ipc_can_multi_send
```

我自己做的时候，修了两个bug，

1. parse error before u_int 这个原因类似于`return func(u_int a, u_int b)`，func调用里面不能有参数类型
2. undefined reference of sys_ipc_can_multi_send in syscall.S 表面上看是在syscall.S里面报错了，实际上，是lib/syscall_all.c里面，添加的那个函数，名字打成了`syscall_ipc_can_multi_send(args)`，多了个`call`，导致syscall.S在table那一步没法正常进入entry。

```
#include <asm/regdef.h>
#include <asm/cp0regdef.h>
#include <asm/asm.h>
#include <stackframe.h>
#include <unistd.h>

/*** exercise 4.2 ***/
NESTED(handle_sys,TF_SIZE, sp)
    SAVE_ALL                            // Macro used to save trapframe
    CLI                                 // Clean Interrupt Mask
    nop
    .set at                             // Resume use of $at

    // TODO: Fetch EPC from Trapframe, calculate a proper value and store it back to trapframe.

	lw		t0, TF_EPC(sp)
//	lw		t1, TF_CAUSE(sp)
//	lui		t2, 0x8000
//	and		t1, t1, t2
//	bnez	t1, IS_BD
//	nop
	addiu	t0, t0, 4
//IS_BD:
	sw		t0, TF_EPC(sp)



    // TODO: Copy the syscall number into $a0.

	lw a0, TF_REG4(sp)
    
    addiu   a0, a0, -__SYSCALL_BASE     // a0 <- relative syscall number
    sll     t0, a0, 2                   // t0 <- relative syscall number times 4
    la      t1, sys_call_table          // t1 <- syscall table base
    addu    t1, t1, t0                  // t1 <- table entry of specific syscall
    lw      t2, 0(t1)                   // t2 <- function entry of specific syscall

	lw s0, TF_REG5(sp) // s0 is the size of stack
	addiu s0, s0, 1
	sll s0, s0, 2

    lw      t0, TF_REG29(sp)            // t0 <- user's stack pointer
	lw      a0, TF_REG4(sp)
	lw      a1, TF_REG6(sp)
	lw      a2, TF_REG7(sp)
	lw      a3, 16(t0)

	subu sp, sp, s0

 	li t9, 0
 foo:
 	beq t9, s0, bar
 	addu t3, t0, t9
 	addiu t3, t3, 4
 	lw t5, 0(t3)
 	addu t3, sp, t9
 	sw t5, 0(t3)
 	addu t9, t9, 4
	j foo
 bar:

//    lw      t3, 20(t0)                  // t3 <- the 4th argument of msyscall
//    lw      t4, 24(t0)                  // t4 <- the 5th argument of msyscall
//    lw      t5, 28(t0)                  // t5 <- the 6th argument of msyscall
//    lw      t6, 32(t0)                  // t6 <- the 7th argument of msyscall
//    lw      t7, 36(t0)                  // t7 <- the 8th argument of msyscall
//    lw      t8, 40(t0)                  // t8 <- the 9th argument of msyscall
//
//    // TODO: Allocate a space of six arguments on current kernel stack and copy the six arguments to proper location
//	sw t3, 16(sp)
//	sw t4, 20(sp)
//	sw t5, 24(sp)
//	sw t6, 28(sp)
//	sw t7, 32(sp)
//	sw t8, 36(sp)
    
    
    jalr    t2                          // Invoke sys_* function
    nop
    
    // TODO: Resume current kernel stack

	addu sp, sp, s0
    
    sw      v0, TF_REG2(sp)             // Store return value of function sys_* (in $v0) into trapframe

    j       ret_from_exception          // Return from exeception
    nop
END(handle_sys)

sys_call_table:                         // Syscall Table
.align 2
    .word sys_putchar
    .word sys_getenvid
    .word sys_yield
    .word sys_env_destroy
    .word sys_set_pgfault_handler
    .word sys_mem_alloc
    .word sys_mem_map
    .word sys_mem_unmap
    .word sys_env_alloc
    .word sys_set_env_status
    .word sys_set_trapframe
    .word sys_panic
    .word sys_ipc_can_send
    .word sys_ipc_recv
    .word sys_cgetc
	.word sys_ipc_can_multi_send
```

```
int sys_ipc_can_multi_send(int sysno, u_int value, u_int srcva, u_int perm, u_int envid_1, u_int envid_2, u_int envid_3, u_int envid_4, u_int envid_5) {
	int r;
	struct Env *e1, *e2, *e3, *e4, *e5;
	struct Page *p;

	perm = perm & (BY2PG - 1);

	if (srcva >= UTOP) {
		return -E_INVAL;
	}

	r = envid2env(envid_1, &e1, 0);
	if (r < 0) {
		return r;
	}
	r = envid2env(envid_2, &e2, 0);
	if (r < 0) {
		return r;
	}
	r = envid2env(envid_3, &e3, 0);
	if (r < 0) {
		return r;
	}
	r = envid2env(envid_4, &e4, 0);
	if (r < 0) {
		return r;
	}
	r = envid2env(envid_5, &e5, 0);
	if (r < 0) {
		return r;
	}
	int flag;
	flag = 
		e1->env_ipc_recving &&
		e2->env_ipc_recving &&
		e3->env_ipc_recving &&
		e4->env_ipc_recving &&
		e5->env_ipc_recving;

	if (flag == 0) {
		return -E_IPC_NOT_RECV;
	}

	e1->env_ipc_recving = 0;
	e1->env_ipc_from = curenv->env_id;
	e1->env_ipc_value = value;
	e1->env_ipc_perm = perm;
	e1->env_status = ENV_RUNNABLE;
	e2->env_ipc_recving = 0;
	e2->env_ipc_from = curenv->env_id;
	e2->env_ipc_value = value;
	e2->env_ipc_perm = perm;
	e2->env_status = ENV_RUNNABLE;
	e3->env_ipc_recving = 0;
	e3->env_ipc_from = curenv->env_id;
	e3->env_ipc_value = value;
	e3->env_ipc_perm = perm;
	e3->env_status = ENV_RUNNABLE;
	e4->env_ipc_recving = 0;
	e4->env_ipc_from = curenv->env_id;
	e4->env_ipc_value = value;
	e4->env_ipc_perm = perm;
	e4->env_status = ENV_RUNNABLE;
	e5->env_ipc_recving = 0;
	e5->env_ipc_from = curenv->env_id;
	e5->env_ipc_value = value;
	e5->env_ipc_perm = perm;
	e5->env_status = ENV_RUNNABLE;

	if (srcva != 0) {
		p = page_lookup(curenv->env_pgdir, srcva, NULL);
		if (p == NULL) {
			return -E_INVAL;
		}
		page_insert(e1->env_pgdir, p, e1->env_ipc_dstva, perm);
		page_insert(e2->env_pgdir, p, e2->env_ipc_dstva, perm);
		page_insert(e3->env_pgdir, p, e3->env_ipc_dstva, perm);
		page_insert(e4->env_pgdir, p, e4->env_ipc_dstva, perm);
		page_insert(e5->env_pgdir, p, e5->env_ipc_dstva, perm);
	}

	return 0;
}
```

