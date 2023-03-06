# Lab lazy: Lazy allocation

Status: Completed
板块: Lab, page fault

Lab: xv6 lazy page allocation

实验目的：

添加Lazy Allocation特性到Xv6中

## Q1 取消sbrk的eager allocation特性

![Untitled](Lab%20lazy%20Lazy%20allocation%2032e0376ae16c4f50818763435e51c975/Untitled.png)

我们`echo hi`时，execcmd会调用malloc，malloc又会使用sbrk系统调用

![Untitled](Lab%20lazy%20Lazy%20allocation%2032e0376ae16c4f50818763435e51c975/Untitled%201.png)

使用sbrk系统调用新分配一块内存到堆上并且返回这块内存地址，那我们紧接着使用这块内存时，由于未建立页表与分配物理内存，因此导致了page fault

这是page falut handler(usertrap())提示panic: uvmunmap: not mapped

![Untitled](Lab%20lazy%20Lazy%20allocation%2032e0376ae16c4f50818763435e51c975/Untitled%202.png)

并且Xv6原始对待page fault方式就很粗鲁，给出错误err信息后就直接杀死这个进程

## Q2 完成Lazy allocation -- 使echo hi正常工作

根据hints和uvmalloc即可完成

解法：

在trap.c的page fault handler中来在错误地址上映射新分配的物理内存页，然后返回到用户空间，让进程继续执行，从而从用户空间响应页面错误

提示：

- 注意下表表标题，Exception Code (scause Reg)是13 15对应是什么原因

You can check whether a fault is a page fault by seeing if r_scause() is 13 or 15 in usertrap().

![Untitled](Lab%20lazy%20Lazy%20allocation%2032e0376ae16c4f50818763435e51c975/Untitled%203.png)

思路：

- 在sys_sbrk中只增加myproc()->sz

Xv6 applications ask the kernel **for** `heap` memory using the sbrk() system call.

Lazy allocation only increases `p**->**sz` in sys_sbrk().

The process uses unallocate memory virtual address leading to page fault after sbrk.

- 检验是否lazy allocation(by flag bits of PTEs) （本实验不必要）

Create PTEs 建立映射关系 && allocate a page (usertrap()中)

- 重新执行引起错误的指令
    - 检验是否lazy allocation （本实验不必要）
    - 在usertrap()中有p->trapframe->epc用来保存用户程序PC
    - 在usertrapret()中会将PC换成p->trapframe->epc
    - 回到用户态将重新执行导致page fault的指令
    
    ![Untitled](Lab%20lazy%20Lazy%20allocation%2032e0376ae16c4f50818763435e51c975/Untitled%204.png)
    

![Untitled](Lab%20lazy%20Lazy%20allocation%2032e0376ae16c4f50818763435e51c975/Untitled%205.png)

**在uvmunmap中的两个panic:**

![Untitled](Lab%20lazy%20Lazy%20allocation%2032e0376ae16c4f50818763435e51c975/Untitled%206.png)

- 第一个if下的panic发生的原因，比如，用sbrk申请内存，但是没有真正使用，然后再调用这个函数释放，这个会导致第一个if panic发生。
- 第二个if下的panic发生的情况是，比如在sh下输入echo hi，sh进程在用户态用了malloc或者sbrk，这个时候申请的内存还没有在页表中反应出来，仅仅增加了p->sz，所以在sh进程中往malloc出来的空间中写东西会page fault，然后进trap，在trap中，sh进程的killed标记设为1，然后开始释放进程sh，其中有一步是，在uvmunmap中，之所以能通过第一个if的检查，因为sh进程虚存实际大小只占了数个page（4个），三级页表里还有几百个空的pte，sbrk也是从低地址开始的，所以如果sbrk的空间不大，那这些空间的地址也是在相同的三级页表中，所以walk出来的pte指针的不为0。也就说，虽然malloc出来的地址在这个三级页表的范围内，但是还没有对应的物理空间，所以*pte==0，然后panic。
    
    **释放页表**
    

当然如果malloc出来的地址对应的任意一级页表恰好不存在，第一个if也会panic。假设sh进程正好占满了一个三级页表，然后再malloc，这个时候就是第一个if panic，而不是第二个if panic了，所以要强调sh的page少。当然如果sh程序本身的text段占了逻辑上从零开始的连续513个页表项，那再malloc，就是第二个if panic了。也还跟malloc的实现有关，比较复杂。

## Q3 Lazy allocation -- Final 要求

<aside>
💡 注意：
* 我们之前谈的都是**用户进程访问用户虚拟地址空间**时，遇到访问valid but not map(lazy allocation)时，内核会使其陷入Trap处理页故障，检测是否为合法但未映射页，并对这种页分配物理内存并建立映射 （malloc->sbrk 再对valid but not mapped虚拟地址进行访问）

![Untitled](Lab%20lazy%20Lazy%20allocation%2032e0376ae16c4f50818763435e51c975/Untitled%207.png)

* 但本题涉及到**内核访问用户虚拟地址空间**时出错，找不到对应物理地址，需要分配物理页建立映射或其他方法，这次不会进入到Trap中，而是需要在翻译地址和copyin、copyout中处理缺页故障

</aside>

- Handle negative sbrk() arguments. ---- `uvmdealloc` Mem 取消映射与deallocate mem防止内存泄露
    
    ![Untitled](Lab%20lazy%20Lazy%20allocation%2032e0376ae16c4f50818763435e51c975/Untitled%208.png)
    
- Kill a process if it page-faults on a virtual memory address higher than any allocated with sbrk().
- Handle faults on the invalid page below the user stack.
    
    ![Untitled](Lab%20lazy%20Lazy%20allocation%2032e0376ae16c4f50818763435e51c975/Untitled%209.png)
    
- Handle the parent-to-child memory copy in **fork**() correctly.

思路：

在父进程中sbrk但没有实际分配的内存在子进程在父进程中sbrk但没有实际分配的内存在子进程中也是保持原状在父进程中sbrk但没有实际分配的内存在子进程中也是保持原状中也是保持原状

![Untitled](Lab%20lazy%20Lazy%20allocation%2032e0376ae16c4f50818763435e51c975/Untitled%2010.png)

![Untitled](Lab%20lazy%20Lazy%20allocation%2032e0376ae16c4f50818763435e51c975/Untitled%2011.png)

fork-->uvmcopy

- Handle out-of-memory correctly: if kalloc() fails in the page fault handler, kill the current process.

![Untitled](Lab%20lazy%20Lazy%20allocation%2032e0376ae16c4f50818763435e51c975/Untitled%2012.png)

- **Handle the case in which a process passes a valid address from sbrk() to a system call such as read or write, but the memory for that address has not yet been allocated.**

这个hint不是给usertrap中说的

![Untitled](Lab%20lazy%20Lazy%20allocation%2032e0376ae16c4f50818763435e51c975/Untitled%2013.png)

我们来看看出错的地方

![Untitled](Lab%20lazy%20Lazy%20allocation%2032e0376ae16c4f50818763435e51c975/Untitled%2014.png)

1. 给a进行lazy allocation 的一个Page实际上是没有的，只是对p->sz增加了，我们这里从a write到fd时出错（测试点是从a**读取**内容到内核，内核再写给fd）

![Untitled](Lab%20lazy%20Lazy%20allocation%2032e0376ae16c4f50818763435e51c975/Untitled%2015.png)

这里用户态进程usertest调用系统调用write传入虚拟地址0x0000000000011000，进入内核态：

![Untitled](Lab%20lazy%20Lazy%20allocation%2032e0376ae16c4f50818763435e51c975/Untitled%2016.png)

![Untitled](Lab%20lazy%20Lazy%20allocation%2032e0376ae16c4f50818763435e51c975/Untitled%2017.png)

![Untitled](Lab%20lazy%20Lazy%20allocation%2032e0376ae16c4f50818763435e51c975/Untitled%2018.png)

![Untitled](Lab%20lazy%20Lazy%20allocation%2032e0376ae16c4f50818763435e51c975/Untitled%2019.png)

![Untitled](Lab%20lazy%20Lazy%20allocation%2032e0376ae16c4f50818763435e51c975/Untitled%2020.png)

pa0为0时，可能对应虚拟地址非法，也有可能是虚拟地址合法但由于是sbrk的lazy allocation，对于后者就是我们题中要求handle的，对于**从未分配物理内存的虚拟地址读内容是有问题的，我们跳过就好**，如果是写入合法的但未分配虚拟地址进行分配物理页再映射
为了区分两种情况，我们在walkaddr翻译地址时进行区分

![Untitled](Lab%20lazy%20Lazy%20allocation%2032e0376ae16c4f50818763435e51c975/Untitled%2021.png)

![Untitled](Lab%20lazy%20Lazy%20allocation%2032e0376ae16c4f50818763435e51c975/Untitled%2022.png)

![Untitled](Lab%20lazy%20Lazy%20allocation%2032e0376ae16c4f50818763435e51c975/Untitled%2023.png)

![Untitled](Lab%20lazy%20Lazy%20allocation%2032e0376ae16c4f50818763435e51c975/Untitled%2024.png)

在walk寻找虚拟地址对应物理地址时找不到映射关系，且walk中alloc参数为0，则没有映射关系直接return 0 

最终返回到最上层-1，则写入失败，由于无法找到映射的物理地址

> 这里不是对一个不存在的地址访问，因此不会进入usertrap
> 

1. 我们修改后再test下
    
    ![Untitled](Lab%20lazy%20Lazy%20allocation%2032e0376ae16c4f50818763435e51c975/Untitled%2025.png)
    
    这次向虚拟地址a指向的空间取2*sizeof(in地址a指向的空间取2*sizeof(int)大小**写入**分配的管道的两个口的文件描述符值
    
    ![Untitled](Lab%20lazy%20Lazy%20allocation%2032e0376ae16c4f50818763435e51c975/Untitled%2026.png)
    
    ![Untitled](Lab%20lazy%20Lazy%20allocation%2032e0376ae16c4f50818763435e51c975/Untitled%2027.png)
    
    ![Untitled](Lab%20lazy%20Lazy%20allocation%2032e0376ae16c4f50818763435e51c975/Untitled%2028.png)
    
    ![Untitled](Lab%20lazy%20Lazy%20allocation%2032e0376ae16c4f50818763435e51c975/Untitled%2029.png)
    
    和copyin是一样的问题