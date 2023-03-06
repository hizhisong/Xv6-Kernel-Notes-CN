# Lab: Copy-on-Write Fork for xv6

Status: Completed
板块: Lab, page fault

## COW

- **COW fork**
    
    COW ****** fork中的基本设计是父进程和子进程最初共享**所有**的物理页面，但将它们映射设置为只读。因此，当子进程或父进程执行store指令时，RISC-V CPU会引发一个页面故障异常。作为对这个异常的响应，**内核会拷贝一份 包含故障地址的页**。然后将一个副本的读/写映射在子进程地址空间，另一个副本的读/写映射在父进程地址空间。更新页表后，内核在引起故障的指令处恢复故障处理。因为内核已经更新了相关的PTE，允许写入，所以现在故障指令将正常执行。
    

## Hints

![Untitled](Lab%20Copy-on-Write%20Fork%20for%20xv6%200e7ea2e8c47f49cf93b7db0ef4e9a51c/Untitled.png)

## Solution

![Untitled](Lab%20Copy-on-Write%20Fork%20for%20xv6%200e7ea2e8c47f49cf93b7db0ef4e9a51c/Untitled%201.png)

![2021-12-06 18.35.18.866.jpg](Lab%20Copy-on-Write%20Fork%20for%20xv6%200e7ea2e8c47f49cf93b7db0ef4e9a51c/2021-12-06_18.35.18.866.jpg)

对于pageIndex-count的存储，如果我们采用Linux内核哈希表实现方法存储，其成本是比直接使用固定数组大的

<aside>
💡 哈希表优点：查找速度快
上述两种实现都可以看成是哈希表
由于我们本身的key和value的空间就很小，因此用来进行链哈希时产生的meta-data就过大，不如第一种直接存储法
存储成本考量：
采用链哈希(func(key)用链表串起来，同一func(key)的不同key-value用链表串起来)
当我们的链表节点中sizeof(key)>sizeof(keyType*) && sizeof(value) > sizeof(valueType*)，这样采用链哈希的存法存储成本低于第一种存储

</aside>

[http://kerneltravel.net/blog/2020/hash_list_szp/](http://kerneltravel.net/blog/2020/hash_list_szp/)

[uml.org.cn/embeded/201904173.asp](http://uml.org.cn/embeded/201904173.asp)

[https://blog.csdn.net/JohnJim0/article/details/109240582](https://blog.csdn.net/JohnJim0/article/details/109240582)

---

**本题思路不难，但是问题就是几个Corner cases折磨人，让人过不了cowtest的file和usertests**

- 我们创建管理引用计数的数据结构，考虑到并发，我们这里封装一个锁

![Untitled](Lab%20Copy-on-Write%20Fork%20for%20xv6%200e7ea2e8c47f49cf93b7db0ef4e9a51c/Untitled%202.png)

- 对于几个宏

![Untitled](Lab%20Copy-on-Write%20Fork%20for%20xv6%200e7ea2e8c47f49cf93b7db0ef4e9a51c/Untitled%203.png)

- 对引用计数数据结构初始化

![Untitled](Lab%20Copy-on-Write%20Fork%20for%20xv6%200e7ea2e8c47f49cf93b7db0ef4e9a51c/Untitled%204.png)

- 我们从整个COW-Fork流程开始
    - fork -> uvmcopy
        
        我们将原有的直接kalloc空间转变成对父进程整个虚拟地址的物理页的引用复制，在我们子进程的新页表中
        
        ![Untitled](Lab%20Copy-on-Write%20Fork%20for%20xv6%200e7ea2e8c47f49cf93b7db0ef4e9a51c/Untitled%205.png)
        
    - 当我们的父/子进程(肯定都是用户态进程)任一进程访问到该物理页时，我们需要在trap中将其处理：新建物理页、复制内容、重新执行指令(cow page fault  handler部分)
        
        ![Untitled](Lab%20Copy-on-Write%20Fork%20for%20xv6%200e7ea2e8c47f49cf93b7db0ef4e9a51c/Untitled%206.png)
        
        这里是处理所有无法由虚拟地址翻译成物理地址的地方，处理不当就会cause some problems(usertest)
        load操作数无法被翻译
        store操作数无法被翻译
        指令本身的地址无法被翻译成物理地址
        
        pte **=** 0 **==>** 虚拟地址没有对应的物理地址
        
        物理地址不可写 pte的PTE_W位为0
        
        cow共享页不可写 pte的PTE_COW位为1
        
        虚拟地址值不合法(va **>** MAXVA **||** (va **<=** PGROUNDDOWN(p->trapframe->sp) **&&** va **>=** PGROUNDDOWN(p->trapframe->sp)**-**PGSIZE))
        
        具体handler中错误(例如cow中无法再kalloc了(OOM))
        
        ...
        
        这些情况都需要将程序kill掉
        
    - 还会有需要cow page handler处理的地方 -- copyout
        
        ![Untitled](Lab%20Copy-on-Write%20Fork%20for%20xv6%200e7ea2e8c47f49cf93b7db0ef4e9a51c/Untitled%207.png)
        
    - 我们来看cow page fault handler
        
        在理解cow page fault handler之前我们看看两个操作 -- kalloc、kfree
        
        ![Untitled](Lab%20Copy-on-Write%20Fork%20for%20xv6%200e7ea2e8c47f49cf93b7db0ef4e9a51c/Untitled%208.png)
        
        ![Untitled](Lab%20Copy-on-Write%20Fork%20for%20xv6%200e7ea2e8c47f49cf93b7db0ef4e9a51c/Untitled%209.png)
        
        kalloc引用计数++很好理解，但在kfree这里，当引用计数>1时证明之前有两个及以上的进程在共享这个物理页，因此这次free只能对引用-1，而当引用数为1证明只有一个进程在使用该页，因此此次free是对该页的彻底释放回收
        
        我们回到cow page fault handler
        
        ![Untitled](Lab%20Copy-on-Write%20Fork%20for%20xv6%200e7ea2e8c47f49cf93b7db0ef4e9a51c/Untitled%2010.png)
        
        // todo
        
        // 上述这种方案啰嗦，我们可以将虚拟地址对应物理地址不存在、虚拟地址超范围等行为放在handler处理，简化返回等逻辑处理
        
        解法二不同的是在usertrap、copyout、cowhandler这里处理略有不同，极易采坑
        
        ![Untitled](Lab%20Copy-on-Write%20Fork%20for%20xv6%200e7ea2e8c47f49cf93b7db0ef4e9a51c/Untitled%2011.png)
        
        我们只需要判断copyout目的用户进程虚拟地址是否合法且为cow
        
        ![Untitled](Lab%20Copy-on-Write%20Fork%20for%20xv6%200e7ea2e8c47f49cf93b7db0ef4e9a51c/Untitled%2012.png)
        
        usertrap这里需要额外地址合法性处理并进行handler
        
        ![Untitled](Lab%20Copy-on-Write%20Fork%20for%20xv6%200e7ea2e8c47f49cf93b7db0ef4e9a51c/Untitled%2013.png)
        
        handler多起了一份地址合法性等检测
        

---

## 测试

```bash
$ cowtest
$ usertest  // 一些Corner cases的测试
```