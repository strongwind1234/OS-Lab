# lab0.5+lab1实验报告
**组员：梁锦洲，赵乐**


## lab0.5
**练习1: 使用GDB验证启动流程**
>为了熟悉使用qemu和gdb进行调试工作,使用gdb调试QEMU模拟的RISC-V计算机加电开始运行到执行应用程序的第一条指令（即跳转到0x80200000）这个阶段的执行过程，说明RISC-V硬件加电后的几条指令在哪里？完成了哪些功能？要求在报告中简要写出练习过程和回答。

lab0.5运行过程：
加电后pc初始值：`0x1000`
从0x1000到0x1010的指令为：

`0x1000:	auipc	t0,0x0`
将PC的值存入t0

`0x1004:	addi	a1,t0,32`
a1=t0+32

`0x1008:	csrr	a0,mhartid`
a0被赋予mhartid寄存器的值

`0x100c:	ld	t0,24(t0)`
从地址[t0+24]中读取一个64位的值

`0x1010:	jr	t0`
跳转到t0指定的地址（0x80000000）

接下来跳转到0x80000000，开始加载OpenSBI.bin
然后跳转到0x80200000，加载内核镜像


## lab1
**练习1：理解内核启动中的程序入口操作**
>阅读 kern/init/entry.S内容代码，结合操作系统内核启动流程，说明指令 la sp, bootstacktop 完成了什么操作，目的是什么？ tail kern_init 完成了什么操作，目的是什么？

- `la sp,bootstacktop`这条指令是将bootstacktop标签的地址加载到sp栈寄存器中，即内核栈的最高地址，接下来内核栈将从高地址向低地址增长，以便于后续函数调用和变量的存储。
- `tail kern_init`尾调用kern_init函数，kern_init是内核初始化函数，尾调用可以将被调函数的返回值作为调用函数的返回值，这样可以不消耗额外的调用栈空间，直接替换当前栈帧。

**练习2：完善中断处理**
- 根据题目要求补充代码如下：
```
    /*(1)设置下次时钟中断- clock_set_next_event()
    *(2)计数器（ticks）加一
    *(3)当计数器加到100的时候，我们会输出一个`100ticks`表示我们触发了100次时钟中断，同时打印次数（num）加一
    * (4)判断打印次数，当打印次数为10时，调用<sbi.h>中的关机函数关机
    */
    clock_set_next_event();
    ticks++;
    if(ticks==TICK_NUM){
    print_ticks();
    ticks=0;
    num++;
    }
    if(num==10){
    sbi_shutdown();
    }
```
- 实现过程
首先调用clock_set_next_event()函数设置时钟中断，每次时钟中断时计数器ticks+1,判断ticks的值，每达到100次时调用print_ticks()函数打印中断次数，同时计数器归零并且将打印次数num+1。当打印次数到达十次时，调用关机函数关闭qemu。
- 运行效果
```
    100 ticks
    100 ticks
    100 ticks
    100 ticks
    100 ticks
    100 ticks
    100 ticks
    100 ticks
    100 ticks
    100 ticks
```


**扩展练习 Challenge1：描述与理解中断流程**
>回答：描述ucore中处理中断异常的流程（从异常的产生开始），其中mov a0，sp的目的是什么？SAVE_ALL中寄寄存器保存在栈中的位置是什么确定的？对于任何中断，__alltraps 中都需要保存所有寄存器吗？请说明理由。

- 异常产生后，跳转到指定的__alltraps（在trapentry.S里）标记处，保存当前PC值和其他状态到相关的控制和状态寄存器中，然后将栈顶指针赋值给a0寄存器作为异常处理程序trap.c的参数，使其找到异常处理的入口，再由trap.c依照类型的不同交给interrupt_handler（中断）或是exception_handler（异常）进行处理。`move a0，sp`的目的是将当前栈帧地址传给异常处理程序，找到处理入口。

- SAVE_ALL中寄存器保存在栈中的位置是由栈帧地址、寄存器编号等确定的。对于任何中断，__alltraps 中都需要保存所有寄存器。以确保在中断处理程序执行完毕后能够恢复到中断发生前的状态，保证不会出错。


**扩展练习 Challenge2：理解上下文切换机制**
>回答：在trapentry.S中汇编代码 csrw sscratch, sp；csrrw s0, sscratch, x0实现了什么操作，目的是什么？save all里面保存了stval scause这些csr，而在restore all里面却不还原它们？那这样store的意义何在呢？

- `csrw sscratch, sp`这条指令将原先的栈顶指针存入sscratch中，由于sscrath是一个CSRs寄存器，对其读写时使用csrw等指令。`csrrw s0, sscratch, x0`这条指令先将sscratch寄存器中的值存入s0中，再将sscratc的值置零。目的时作为标识来表示目前是内核态中断而非用户态。
- scause记录了中断或异常的类型信息，svtal记录了处理中断时的辅助信息，这些信息只在处理中断时能够用到，在恢复上下文时无需恢复。

**扩展练习Challenge3：完善异常中断**
- 根据题目要求补充代码如下：
- trap.c中代码补充：
  ```
  // 非法指令异常处理
    /* LAB1 CHALLENGE3   2212221 :  */
    /*(1)输出指令异常类型（ Illegal instruction）
    *(2)输出异常指令地址
    *(3)更新 tf->epc寄存器
    */
    cprintf("Exception type:Illegal instruction\n");
    cprintf("Illegal instruction caught at 0x%08x\n",tf->epc);
    tf->epc+=4;
  //断点异常处理
    /* LAB1 CHALLLENGE3   2113874 :  */
    /*(1)输出指令异常类型（ breakpoint）
    *(2)输出异常指令地址
    *(3)更新 tf->epc寄存器
    */
    cprintf("Exception type: breakpoint\n");
    cprintf("ebreak caught at 0x%08x\n",tf->epc);
    tf->epc += 3;
    break;
  ```
- init.c中代码补充:
```
int kern_init(void) {
    extern char edata[], end[];
    memset(edata, 0, end - edata);
    cons_init();  // init the console
    const char *message = "(THU.CST) os is loading ...\n";
    cprintf("%s\n\n", message);
    print_kerninfo();
    // grade_backtrace();
    idt_init();  // init interrupt descriptor table
    // rdtime in mbare mode crashes
    clock_init();  // init clock interrupt
    intr_enable();  // enable irq interrupt
    
    asm volatile("ebreak");//设置断点
    asm volatile(".word 0x3052FFFF");//非法指令
  
    while (1)
        ;
}
```
- 具体实验步骤：
首先在kern/init/init.c文件中的内核初始化函数kern_init中用内联汇编指令设置断点和非法指令，再调用cprint打印异常原因和异常产生地址，其中地址存放在之前定义的结构体变量tf->epc中。然后`tf->epc+=4`保证恢复后pc指向下一条指令。
- 运行效果：
```
Exception type: breakpoint
ebreak caught at 0x8020004e
Exception type:Illegal instruction
Illegal instruction caught at 0x80200051

```

