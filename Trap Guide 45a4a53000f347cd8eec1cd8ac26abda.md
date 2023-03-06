# Trap Guide

板块: Trap

[「Coding Master」第20话 扒一扒中断向量表的底裤_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1ah411D7vw)

# **Xv6 trap 处理分为四个阶段**

1. CPU采取硬件行为
2. "汇编代码"为下一步C code 处理做准备(为内核C代码准备的汇编入口)
3. C Trap Handler(处理trap的C 处理程序)
    
    system call or device-driver service routine
    
4. 来自用户空间的trap、来自内核空间的trap和定时器中断，设置单独的 汇编入口 和 C trap处理程序

### **Trap第一阶段 -- CPU硬件行为**

1. **如果该trap是设备中断，*且* sstatus SIE位为0，那么就不要执行以下任何操作。（**SIE -- 0：禁止中断. 1: 开启中断 ;**）**
2. 通过清除SIE来**禁用中断**。
3. **复制pc到sepc**
4. 将当前模式(用户或监督者)保存在sstatus的SPP位。
5. **在scause设置该次trap的原因。**
6. **将模式转换为监督者(S mode)。**
7. **将stvec复制到pc。**(**stvec**指向的trap处理程序地址为**uservec**地址，由于此时还为用户页表，因此uservec在内核和用户页表地址一样，即trampoline页)
8. **执行新的pc (uservec)**
    
    注意，CPU不会切换到内核页表，不会切换到内核中的栈，也不会保存pc以外的任何寄存器。内核软件必须执行这些任务。
    

### **中断处理程序路径 -- Trap第二、三、四软件阶段**

在用户空间执行时，如果用户程序进行了系统调用(**ecall**指令)，或者做了一些非法的事情，或者设备中断，都可能发生trap。

1. 来自用户空间的trap的处理路径是**uservec**(kernel/trampoline.S:16)
    - uservec: 交换sscratch与a0，之后a0存着用户trapframe，将用户寄存器全部**存**储到trapframe
    - 从trapframe中的用户寄存器中**读**kernel stack、hartid、usertrap、kernel pagetable到内核使用相应寄存器中
    - 跳转到usertrap不返回 (jr t0  (t0存着usertrap函数的地址))
2. 然后是**usertrap**(kernel/trap.c:37)
    - 确定trap的原因，对应原因处理它(例如对于系统调用执行`syscall()`)，返回
3. 返回跳转到**usertrapret**(kernel/trap.c:90)
    - 设置RISC-V控制寄存器，为以后用户空间trap做准备：
        
        改变**stvec**来引用**uservec**，准备**uservec**所依赖的**trapframe**字段，并将**sepc**设置为先前保存的用户程序计数器（在userret汇编中 最后的sret指令将用户PC值改为sepc值，sepc又在此步被设置）。最后，**usertrapret**在用户页表和内核页表中映射的trampoline页上调用**userret**，因为**userret**中的汇编代码会切换页表，这步调用userret时将trapframe地址保存至sscratch。
        
4. 然后是**userret**(kernel/trampoline.S:16)
5. 恢复trapframe(用户现场) 跳转用户pc