# lab4: traps

## part0
略

## part1:backtrace

preparation lab



打印当前栈上所有栈帧返回值的地址。打印方法：通过遍历每个栈帧，获取栈帧指针，返回地址位于栈帧指针偏移-8位置。



第一个栈帧指针通过r_fp()获得，此后每个栈帧指针通过距离栈帧指针偏移量为-16的pre fp获得，该地址负责保存上一个栈帧指针的位置



解析方式：移动栈帧指针到pre fp地址位置（获得pre fp地址），将其类型转化为指针后，进行解引用以获得上一个栈帧指针



地址内“保存”内容



## part2:alarm

添加系统调用sys_sigalarm与sys_sigreturn



为proc结构体添加：保存传入的时钟数与函数指针、保存发生alarm后的当前时钟数、执行函数前的页表状态



sys_sigalarm：向结构体传入变量并将其余初始化，其中“保存页表“的初始化发生在初始化进程时



sys_sigreturn：执行函数结束，返回执行前的状态以便函数能够继续执行下去（即恢复原状态）。主要工作：修改当前trapframe为执行函数前的trapframe，将当前时钟数清空



仅仅通过将sepc置为执行函数的地址是不够的，因为在执行函数结束后其他寄存器的内容相较于alarm发生前已经发生了改变，因此要在执行函数结束后将这些寄存器的内容恢复原样，即重新加载原本的trapframe的所有内容即可



防止对处理程序的重复调用——如果处理程序还没有返回，内核就不应该再次调用它。`test2`测试这个。



## 核心思路：

理解普通指令的暂停、trap的流程，系统切换时寄存器的变化。





指令在执行的时候是会暂停的，以下三种情况下会暂停普通指令的执行：

1. 系统调用。例如系统执行 ecall 指令。
2. 异常。例如除零或使用无效地址。
3. 中断。例如设备发出读写请求。

以上三种情况统称为 trap 。trap 是透明的，也就是被执行的代码感知不到 trap 的存在。

trap 的大致流程如下：

1. 控制权交给内核。
2. 内核保存寄存器状态方便日后恢复。
3. 内核执行相应代码。
4. 内核恢复状态并从 trap 中返回。
5. 代码回到原来的地方。

在切换的过程中需要修改寄存器的状态，以下是一些重要寄存器的介绍。

- 程序计数器(Program Counter Register) ，指向了当前正在指向的下一条指令。
- mode ，表明当前mode的标志位，这个标志位表明了当前是supervisor mode还是user mode。当我们在运行Shell的时候，自然是在user mode。
- SATP（Supervisor Address Translation and Protection） 指向page table的物理内存地址。
- STVEC（Supervisor Trap Vector Base Address Register）指向了内核中处理trap的指令的起始地址。
- SEPC（Supervisor Exception Program Counter）在trap的过程中保存程序计数器的值。
- SSRATCH（Supervisor Scratch Register）寄存器，这也是个非常重要的寄存器（详见6.5）。

这些寄存器表明了执行系统调用时计算机的状态。

trap 流程

1. 保存 32 个用户寄存器。
2. 保存程序计数器 CP ，中断完成后通过之前的 PC 继续执行。
3. 将 mode 改成 supervisor mode 。
4. 将 SATP 指向 kernel page table 。
5. 将堆栈寄存器指向位于内核的一个地址，因为我们需要一个堆栈来调用内核的C函数。
6. 设置好后跳入内核的C代码。

这一节的重点是如何从将程序执行从用户空间切换到内核的一个位置。

用户代码不能接入到 user/kernel 切换，因为会破坏安全性，所以trap中涉及到的硬件和内核机制不能依赖任何来自用户空间东西。例如我们不能依赖32个用户寄存器，它们可能保存的是恶意的数据，所以，XV6的trap机制不会查看这些寄存器，而只是将它们保存起来。

trap 机制对用户代码是透明的。也就是用户代码察觉不到 trap 的执行。

可以在supervisor mode完成，但是不能在user mode完成的工作：读写SATP寄存器，也就是page table的指针；STVEC，也就是处理trap的内核指令地址；SEPC，保存当发生trap时的程序计数器；SSCRATCH等等。

在 supervisor mode 下可以使用PTE_U标志位为0的PTE。当PTE_U标志位为1的时候，表明用户代码可以使用这个页表；

supervisor mode 中的代码并不能读写任意物理地址。在supervisor mode中，就像普通的用户代码一样，也需要通过page table来访问内存。如果一个虚拟地址并不在当前由SATP指向的page table中，又或者SATP指向的page table中PTE_U=1，那么supervisor mode不能使用那个地址。所以，即使我们在supervisor mode，我们还是受限于当前page table设置的虚拟地址。

如何通过trap进入到内核空间：

1. write 通过执行 ECALL 指令来执行系统调用。
2. ECALL指令会切换到具有supervisor mode的内核中。ecall 具体干了三件事情：
   1. 将代码从user mode改到supervisor mode。
   2. 将程序计数器的值保存在了SEPC寄存器。
   3. ecall会跳转到STVEC寄存器指向的指令。
3. 在内核中执行的第一个指令是一个由汇编语言写的函数，叫做 uservec ，位于 trampoline.s 中。
   1. 保存现场，也就是保存 32 个通用寄存器中的数据。
   2. 切换到内核页表，内核栈，将当前进程的 CPUid 加载到寄存器中。
   3. 跳转到 usertrap() 。
4. 之后跳转到 trap.c 中的 usertrap() 中。
   1. 更改STVEC寄存器。(从用户态到内核态，如果已经在内核中了那么很多操作将会省去)
   2. 通过 myproc 函数获取当前正在执行的进行。
   3. 保存当前进程的SEPC寄存器到一个与该进程关联的内存中(trapframe)，因为中间如果发生进程切换可能会导致数据被覆盖。
   4. 分析为什么执行 usertrap() 调用，8 表示因为系统调用而执行 usertrap() 函数。
   5. 判断当前进程是否已被杀掉。
   6. 之前保存的 PC + 4，指向返回地址。
   7. 执行 syscall 前开启中断。此时是可以被中断的。
5. 在 usertrap 中，执行 syscall 函数。
   1. 根据传入的代表系统调用的数字进行查找，并在内核中执行具体实现了系统调用功能的函数。此时就是sys_write。
   2. sys_write 将要显示数据输出到console上，完成后会返回给 syscall 函数。
6. usertrap 最终调用了 trap.c 中的 usertrapret() 函数，该函数实现了在C代码中实现的返回到用户空间的工作。
   1. 首先关闭中断，更新STVEC寄存器来指向用户空间的trap处理代码，将STVEC更新到指向用户空间的trap处理代码时。
   2. 存储了内核页表，内核栈的指针。
   3. 存储了usertrap函数的指针，这样trampoline代码才能跳转到这个函数（注，详见6.5中 ld t0 (16)a0 指令）。
   4. 从tp寄存器中读取当前的CPU核编号，并存储在trapframe中，这样trampoline代码才能恢复这个数字，因为用户代码可能会修改这个数字。
7. 此时又回到 trampoline.s 中，执行 userret 完成的了一些细节。调用机器指令返回到用户空间并恢复ECALL之后的用户程序的执行。
   1. 切换page table 。

trampoline page 中包含了内核的trap处理代码。

ecall尽量的简单可以提升软件设计的灵活性。