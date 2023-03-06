# Xv6 启动

板块: 启动

## **Xv6 编译启动**

![Untitled](Xv6%20%E5%90%AF%E5%8A%A8%20ee0c30670ebe4c808fae3ceee3ed1566/Untitled.png)

## **Xv6 entry -- entry.S**

![Untitled](Xv6%20%E5%90%AF%E5%8A%A8%20ee0c30670ebe4c808fae3ceee3ed1566/Untitled%201.png)

> 为了接下来的内核C函数可以执行，要为每个CPU设置 **调度函数(调度线程)执行的内核栈**，每个核心的调度线程内核栈大小为4096bytes (A page in xv6)

**注意：这里不是用户态进程进入内核态时执行内核代码使用的栈，这个栈是给调度线程使用的，每个用户态进程自身都在进程创建时分配了一个内核栈，栈地址放在每个进程PCB的trapframe->kernel_sp**
> 

## **Xv6 start()**

![Untitled](Xv6%20%E5%90%AF%E5%8A%A8%20ee0c30670ebe4c808fae3ceee3ed1566/Untitled%202.png)

> 每个CPU都要设置一下，使每个CPU都能接收到Trap并在Supervisor Mode能陷入
> 
> 
> Trap:
> 
> 1. 中断 来自中断控制器的External Interrupt 来自软件发出的Software Interrupt
> 2. Exception 除零操作、Page Fault、Syscall etc.
> 
> 接着每个CPU进入main.c:main
> 

## **Xv6 main()**

![Untitled](Xv6%20%E5%90%AF%E5%8A%A8%20ee0c30670ebe4c808fae3ceee3ed1566/Untitled%203.png)

> 仅有hart id为0的CPU核心会进入第一个用户进程user/init.c (userinit的行为是exec(user/_init)) 所有的核心都会完成许多内核模块的初始化任务(见代码注释)
> 
> 
> 最终，除了第一个运行init进程的CPU，剩下CPU被调度器分配调度任务
> 

## **Xv6 init.c -- main()**

![Untitled](Xv6%20%E5%90%AF%E5%8A%A8%20ee0c30670ebe4c808fae3ceee3ed1566/Untitled%204.png)

> 父进程通过fork子进程保证有shell可以执行，其在一个infinity loop中等待子进程shell退出 一旦退出重新建立一个shell进程，这样我们就可以通过子进程进入到了shell，用其执行其他程序了
>