# lab5: lazy

## preparation

陷阱指令和系统调用

### 陷入机制

重要寄存器：管理模式下处理trap

* `stvec`：内核写入其陷阱处理程序的地址；risc-v跳转到这里处理陷阱
* `sepc`：发生trap时，系统保存pc到此位置。（因为`pc`会被`stvec`覆盖）`sret`指令会将`sepc`复制到`pc`返回，内核可以写入`sepc`来控制`sret`的去向
* `scause`：risc-v在这里放置一个描述陷阱原因的数字
  * 0：无法识别
  * 1：外设
  * 2：时间中断
* `sscratch`：内核在这里放置一个 在陷阱处理程序一开始便会派上用场 的值
* `sstatus`：其中的**SIE**位控制设备中断是否启用。若内核清空**SIE**，risc-v将推迟设备终端直到内核重新设置**SIE**。**SPP**位指示陷阱是来自用户模式还是管理模式，并控制`sret`的返回模式



强制trap时

1. 若trap是设备中断，且`SIE`位被清空，则不执行以下操作
2. 清除**SIE**以禁用中断。
3. 将`pc`复制到`sepc`。
4. 将当前模式（用户或管理）保存在状态的**SPP**位中。
5. 设置`scause`以反映产生陷阱的原因。
6. 将模式设置为管理模式。
7. 将`stvec`复制到`pc`。
8. 在新的`pc`上开始执行。



内核陷入流程：uservec usertrap usertrapret userret



### 页面错误异常

xv6对异常的响应：若user发生异常，kernel则终止故障进程；若kernel发生异常，则内核崩溃。

真正的os：使用page default进行COW。

当前xv6的cow: 通过调用`uvmcopy`为子级分配物理内存，并将父级的内存复制到其中，使父子拥有相同的内容，但会导致父子通过对共享堆栈的写入来中断彼此的执行。

COW: 让父子最初共享所有的物理页面且设置为只读，当执行存储指令时会发生page fault。为响应异常，kernel复制包含错误地址的页面，在父子级地址空间中各自映射一个权限为读/写的副本。更新页表后，内核会在故障的指令处恢复故障的执行，由于内核更新了相关PTE，所以现在可以正常执行指令。

`scause`指示页面错误的类型，`stval`包含无法翻译的地址

xv6 page fault类型：加载页面错误（加载指令无法转换其虚拟地址）、存储页面错误（存储指令无法转换其虚拟地址）、指令页面错误（当指令的地址无法转换时）



惰性分配（lazy）：首先，当应用程序调用`sbrk`时，内核增加地址空间，但在页表中将新地址标记为无效。其次，对于包含于其中的地址的页面错误，内核分配物理内存并将其映射到页表中。由于应用程序通常要求比他们需要的更多的内存，惰性分配可以称得上一次胜利: 内核仅在应用程序实际使用它时才分配内存。



## Eliminate allocation from sbrk()

删除growproc调用，保留增加进程大小

输出：

```bash
init: starting sh
$ echo hi
usertrap(): unexpected scause 0x000000000000000f pid=3
            sepc=0x0000000000001258 stval=0x0000000000004008
va=0x0000000000004000 pte=0x0000000000000000
panic: uvmunmap: not mapped
```

##  Lazy allocation

发生page fault时，为错误位置分配内存（一页）

* `r_scause`判断错误原因是否为page fault(13, 15)，`r_stavl`读取造成错误的虚拟地址
* 参考`growproc`中对`uvmalloc`的调用中，对`kalloc`与`mappages`的调用方法，对`usertrap`的分配内存进行改写
  * 错误情况：内存分配失败（无可用内存），分配页失败......(part3补充)
* 分配页需要对齐，`PGROUNDDOWN`向下取整
* 跳过`uvmunmap`中引起panic的部分：pte不存在与pte不可用

## Lazytests and Usertests

主要完成：对错误情况的处理（非法访问）

1. `sbrk()`参数为负数：”缩小“
2. 异常情况：地址增长时，上溢到trapframe以上；地址减少时，下溢到sp以下
3. 异常情况：分配内存时，虚拟地址超过原本”虚假“的地址；回收内存时，虚拟地址发生下溢
4. 正确的拷贝处理：跳过`uvmcopy`中引起panic部分：pte不存在与pte不可用
5. 系统调用中传递有效地址
   1. 首先搞清楚函数执行流程，在调用write后系统trap到内核态，执行copyin来把用户程序va处的内容复制到内核空间，此时若va处并未分配内存，walkaddr会返回0导致系统调用失败。因此我们要做的就是在walkaddr中分配内存
