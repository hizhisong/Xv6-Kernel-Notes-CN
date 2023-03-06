# Lab: Page table

Status: Completed
板块: Lab

## Lab地址：

[https://pdos.csail.mit.edu/6.828/2020/labs/pgtbl.html](https://pdos.csail.mit.edu/6.828/2020/labs/pgtbl.html)

## Q1. Print a page table

`page table 0x0000000087f6e000
..0: pte 0x0000000021fda801 pa 0x0000000087f6a000
.. ..0: pte 0x0000000021fda401 pa 0x0000000087f69000
.. .. ..0: pte 0x0000000021fdac1f pa 0x0000000087f6b000
.. .. ..1: pte 0x0000000021fda00f pa 0x0000000087f68000
.. .. ..2: pte 0x0000000021fd9c1f pa 0x0000000087f67000
..255: pte 0x0000000021fdb401 pa 0x0000000087f6d000
.. ..511: pte 0x0000000021fdb001 pa 0x0000000087f6c000
.. .. ..510: pte 0x0000000021fdd807 pa 0x0000000087f76000
.. .. ..511: pte 0x0000000020001c0b pa 0x0000000080007000`

仔细观察，(PTE_R | PTE_W | PTE_X)一旦被有一种被设置，那么说明这个PTE下面一定没有下一个页表页

Q: Explain the output of vmprint in terms of Fig 3-4 from the text. What does page 0 contain? What is in page 2? When running in user mode, could the process read/write the memory mapped by page 1?

Page 0 contain第0级的下一级页表(第1级)的页表页物理地址，Page2 的第0、1、2个PTE是进程页的物理地址，物理页的alloc从底到顶，这些PTE依次内含data&text、guard page、user stack

进程物理页511 flag位：X Valid，因此是trampoline page，切换用户态和内核态的汇编代码执行代码

![Untitled](Lab%20Page%20table%20cd291a9a9c2e42439a2a2f3487c6834c/Untitled.png)

```c
void vmprint(pagetable_t pt, int level) {
  // A pagetable page contains 512 PTEs
  for (int i = 0; i < 512; i++) {
    pte_t pte = pt[i];
    if (pte & PTE_V) {
      for (int j = level; j > 0; j--) {
        printf(".. ");
      }
      printf("..");
      printf("%d: pte %p pa %p\n", i, pte, PTE2PA(pte));
      uint64 child = PTE2PA(pte);
      if ((pte & (PTE_R | PTE_W | PTE_X)) == 0)   // (PTE_R | PTE_W | PTE_X)一旦被有一种被设置，那么说明这个PTE下面一定没有下一个页表页
        vmprint((pagetable_t)child, level + 1);
    }
  }
}
```

<aside>
💡 **hints：题2与题3是一体的，都认真读完题后才能了解清楚所有题目条件与要求**

</aside>

## Q2. A kernel page table per process

![Untitled](Lab%20Page%20table%20cd291a9a9c2e42439a2a2f3487c6834c/Untitled%201.png)

**内核页表的创建**

**"内存映像"**

在Linux中，用户进程在通过中断或者系统调用**进入内核之后(CPU执行指令的特权级改变)**，MMU页表基地址依旧是当前进程的页表基地址，只不过在内核态可以访问所有的地址空间（对于32位机 0-4GB），当然这时候在内核态如果访问低地址（0-3GB）内存的话，就是访问到了当前进程的用户地址空间。因此使用copy_from_user的时候，用户空间地址参数在内核是可以访问的，因为此时内核可以访问该进程的用户空间页表。copy_to_user也一样。

在Xv6中，整个操作系统中有一个 内核页表 (代码中是一个全局变量，描述内核地址空间) 和 为每一个进程准备的 页表

![Untitled](Lab%20Page%20table%20cd291a9a9c2e42439a2a2f3487c6834c/Untitled%202.png)

```c
/*
 * the kernel's page table.
 */
**pagetable_t kernel_pagetable;**// Per-process state
struct proc {
  struct spinlock lock;
  // p->lock must be held when using these:
  enum procstate state;        // Process state
  struct proc *****parent;         // Parent process
  void *****chan;                  // If non-zero, sleeping on chan
  int killed;                  // If non-zero, have been killed
  int xstate;                  // Exit status to be returned to parent's wait
  int pid;                     // Process ID
  // these are private to the process, so p->lock need not be held.
  uint64 kstack;               // Virtual address of kernel stack
  uint64 sz;                   // Size of process memory (bytes)
  **pagetable_t pagetable;       // User page table**  struct trapframe *****trapframe; // data page for trampoline.S
  struct context context;      // swtch() here to run process
  struct file *****ofile[NOFILE];  // Open files
  struct inode *****cwd;           // Current directory
  char name[16];               // Process name (debugging)
};
```

在本实验开始以前，当一个进程陷入内核态，内核需要处理用户态数据的指针时，由于内核页表中没有用户页表的映射关系，因此不能直接通过MMU解引用用户指针找到物理地址。现在需要在内核态手动实现walk函数遍历查找用户进程的页表中用户数据指针指向的物理地址，再转而内核访问该空间读取数据。

因为当你访问一个虚拟地址时，mmu会根据satp中的页表地址，自动遍历页表，找到物理地址。

而原本的copy_in之所以需要手动编写程序使cpu来完成页表遍历，是因为用户的虚拟地址在当前的内核页表中不存在映射，将用户进程的虚拟地址直接交由mmu翻译必然会产生错误。

现在题目提供的方案放弃原有做法，让每个进程拥有一个内核页表，这个内核页表，与全局内核页表共享部分一致，只不过每次调度切换时，satp寄存器内部为该进程的内核页表，该页表映射的起始空间部分作为与全局内核页表不一样的不共享的部分，每个进程将这部分用于和虚拟内存空间映射（题中说明进程大小不会超过这部分进程内核页表非共享部分映射的物理空间）

由于进程内核页表作为进程`struct proc`的一部分，这样当用户进程执行陷入内核时，传参进入内核态的用户数据指针(虚拟地址)内核会直接解析成进程拥有的物理页表上映射的等值的物理地址，访问得到数据，避免手动解析用户进程页表和虚拟地址(软件模拟MMU)解得用户数据指针(虚拟地址)对应的物理地址的开销

核心：这个lab需要你将进程的页表映射同样添加到内核线程页表中。

kernel pagetable在内核空间中使用，但使用者是o/s；

process's pagetable在用户空间中使用，使用者是普通进程；

process's kernel pagetable 则是普通进程在进入内核时使用。

Tips.

所有进程的内核栈在内核虚拟空间的上面，Trampoline下面（图示的多个进程的kernel stack）

![Untitled](Lab%20Page%20table%20cd291a9a9c2e42439a2a2f3487c6834c/Untitled%203.png)

## Q3. Simplify copyin/copyinstr

第三题是说目前内核态读取进程指针(虚拟地址)都是在软件中使用walk函数遍历查找用户态进程的虚拟内存页表，找到对应物理地址，再在内核对数据访问。(在内核态时，satp寄存器中为内核页表基地址而不是用户进程虚拟内存页表的基地址，因此对用户进程数据的虚拟地址无法让MMU硬件查表获得物理地址)

现在想在内核页表中添加对用户态进程虚拟内存页表的映射，可以直接对用户态进程数据指针访问

Your job in this part of the lab is to add user mappings to each process's kernel page table (created in the previous section) that allow copyin (and the related string function copyinstr) to directly dereference user pointers.

![Untitled](Lab%20Page%20table%20cd291a9a9c2e42439a2a2f3487c6834c/Untitled%204.png)

**本题要求实现对上面建立的进程内内核页表的某部分空间与虚拟内存的映射**

然而这个映射建立在进程内核页表的哪里呢？

由于题目描述：

This scheme relies on the user virtual address range not overlapping the range of virtual addresses that the kernel uses for its own instructions and data. Xv6 uses virtual addresses that start at zero for user address spaces, and luckily the kernel's memory starts at higher addresses. However, **this scheme does limit the maximum size of a user process to be less than the kernel's lowest virtual address.** After the kernel has booted, that address is 0xC000000 in xv6, the address of the PLIC registers; see kvminit() in kernel/vm.c, kernel/memlayout.h, and Figure 3-4 in the text. **You'll need to modify xv6 to prevent user processes from growing larger than the PLIC address.**

- 题目要求实现的这种方案是在要映射用户进程虚拟地址范围与内核自身占用内存范围不会重叠，对用户进程大小做了限制的大前提下，让你实现这样的映射
- 根据题目要求，将用户虚拟地址空间从其**0x**0**L** ~ **0x**0c000000**L** 对应映射到用户进程的内核页表的**0x**0**L** ~ **0x**0c000000**L** (PLIC)即可

# 题解

## T2 - Solution

1. Copy page table (Way1)
2. Share the kernel page table (Way2)

为每个进程创建一个新的内核页表，这个内核页表的部分内容是与全局内核页表一致的，这些内容都是些共享且不会改变的，位置基本上是从CLINT到PLIC地址以上的任何内容，这些内容是不变的共享的

- 添加进程内核页表数据结构

```c
struct proc {
  struct spinlock lock;
  // p->lock must be held when using these:
  enum procstate state;        // Process state
  struct proc *****parent;         // Parent process
  void *****chan;                  // If non-zero, sleeping on chan
  int killed;                  // If non-zero, have been killed
  int xstate;                  // Exit status to be returned to parent's wait
  int pid;                     // Process ID 
  **pagetable_t kernel_pt;       // A kernel_pagetable**  // these are private to the process, so p->lock need not be held.
  uint64 kstack;               // Virtual address of kernel stack
  uint64 sz;                   // Size of process memory (bytes)
  pagetable_t pagetable;       // User page table
  struct trapframe *****trapframe; // data page for trampoline.S
  struct context context;      // swtch() here to run process
  struct file *****ofile[NOFILE];  // Open files
  struct inode *****cwd;           // Current directory
  char name[16];               // Process name (debugging)
};
```

- 建立进程内核页表

```c
void uvmkpmap(pagetable_t kernel_pt, uint64 va, uint64 pa, uint64 sz, int perm)
{
  **if**(mappages(kernel_pt, va, sz, pa, perm) **!=** 0)
    panic("uvmkpmap");
}
// For simplicity, we just copy all PTEs from global kernel_pagetable except entry 0 in the 0th level of global kernel_pagetable
// (Because the entry 0 has ability to represent 1*(2^9)*(2^9)*4KB = 1GB physical pages. That enough to build mapping for user process memory limit.
// (https://pdos.csail.mit.edu/6.828/2020/labs/pgtbl.html#:~:text=You%27ll%20need%20to%20modify%20xv6%20to%20prevent%20user%20processes%20from%20growing%20larger%20than%20the%20PLIC%20address )
// Processes of kernel stacks, kernel self's data, kernel self's instruction and etc. are mapping by entry from entry 1 to entry 511(actually 255)
pagetable_t proc_user_kernel_pagetable() {
  pagetable_t kernel_pt **=** uvmcreate();
  **if** (**!**kernel_pt) {
    **return** 0;
  }
  // Unnecessary actually
    **for** (int i **=** 1; i **<** 512; i**++**) {
    kernel_pt[i] **=** kernel_pagetable[i];
  }
  // uart registers
  uvmkpmap(kernel_pt, UART0, UART0, PGSIZE, PTE_R **|** PTE_W);
  // virtio mmio disk interface
  uvmkpmap(kernel_pt, VIRTIO0, VIRTIO0, PGSIZE, PTE_R **|** PTE_W);
  // CLINT
  uvmkpmap(kernel_pt, CLINT, CLINT, **0x**10000, PTE_R **|** PTE_W);
  uvmkpmap(kernel_pt, PLIC, PLIC, **0x**400000, PTE_R **|** PTE_W);
  **return** kernel_pt;
}
```

- 在allocproc中创建进程内核页表

```c
  // Alloc new kernel page table for each process own
  p->kernel_pt **=** proc_user_kernel_pagetable();
  **if** (**!**p->kernel_pt) {
    freeproc(p);
    release(**&**p->lock);
    **return** 0;
  }
```

由于进程内核页表内核栈部分始终与全局内核页表中内核栈部分保持一致，因此这里可以不做特殊处理

- 下题这部分实现进程内核页表非共享部分的映射处理
- 在进程调度上下文切换时实现satp页表基地址寄存器切换

```c
// scheduler():

int found **=** 0;
**for**(p **=** proc; p **<** **&**proc[NPROC]; p**++**) {
  acquire(**&**p->lock);
  **if**(p->state **==** RUNNABLE) {
    // Switch to chosen process.  It is the process's job
    // to release its lock and then reacquire it
    // before jumping back to us.
    p->state **=** RUNNING;
    c->proc **=** p;

    **// Starting excuting process's instructions**
				w_satp(MAKE_SATP(p->kernel_pt));
        sfence_vma();   // 刷新当前CPU的TLB缓存

        swtch(&c->context, &p->context);

    **// End of process -- Using global kernel pagetable after processing the process in the `kernel mode`**
				w_satp(MAKE_SATP(kernel_pagetable));
        sfence_vma();   // 刷新当前CPU的TLB缓存

    // It should have changed its p->state before coming back.
    c->proc **=** 0;
    found **=** 1;
  }
```

- 对于内核进程清零 -- 进程内核页表部分
    
    <aside>
    💡 Q: 为什么这里仅仅清除 用户内核页表 的第0级页表的第0项下对应的下级多个页表？
    A: 用户虚拟地址从0x0开始，为了使得用户内核页表的用户虚拟内核映射部分直接映射用户虚拟内存的映射，根据后续实验，我们能修改的内核地址空间不超过顶级页表的第一个 entry 的地址范围，所以我们和`kernel_pagetable`共享其他 entry，直接进行复制，这样能够节约次级页表占用的内存空间
    
    </aside>
    
    ```c
    // Only allocation realted to entry 0 at level 0 in user kernel page is exclusive to each process
    // 每个进程的内核页表的第0级的第0个entry下申请的页都是每个进程不一样的，不共享的，而其他的PTE内容都是共享global kernel_pagetable的
    // Thus, kfree the user kernel pagetable and allocation related to entry 0 at level 0
    **static** void proc_free_user_kernel_pagetable(pagetable_t kernel_pt) {
      // Only kfree entry 0 at level 0 user kernel pagetable
      pte_t pte;
      pagetable_t level_1_pt **=** (pagetable_t)PTE2PA((pte_t)(kernel_pt[0]));
      **for** (int i **=** 0; i **<** 512; i**++**) {
        pte **=** level_1_pt[i];
        **if** (pte **&** PTE_V) {
          uint64 phy_page **=** PTE2PA(pte);
          kfree((void*****)phy_page);
          level_1_pt[i] **=** 0;
        }
      }
      kfree((void *****)level_1_pt);  // 中间(第1级)页表
      kfree((void *****)kernel_pt);   // 整个user kernel page table
    }
    ```
    

## T3 - Solution

1. 如何映射？

Allocate new pages from oldsz(address) to newsz(address) **because the user process memory start at 0x0.**

```c
// For fork(), exec(), and sbrk() process space building
**// Allocate new pages from oldsz(address) to newsz(address) because of user process memory start at 0x0**

void uvmCopyUserPt2UkernelPt(pagetable_t userPt, pagetable_t ukPt, uint64 oldsz, uint newsz) {
  if (newsz >= PLIC) {
    panic("uvmCopyUserPt2UkernelPt: User process overflowed");
  }

  for (uint64 va = oldsz; va < newsz; va += PGSIZE) {
    pte_t* uPTE  = walk(userPt, va, 0);
    pte_t* ukPTE = walk(ukPt, va, 1);

    *ukPTE  = *uPTE;
    *ukPTE &= ~(PTE_U|PTE_W|PTE_X);   // 取消不必要权限，这里只需要R，防止用户态内存破坏内核
  }
}
```

使用walk遍历已有的用户进程虚拟地址空间，将其新建PTE与对应物理页进行映射，注意对权限的管控

1. 在哪里调用映射函数？
- fork --> alloc新进程继承父进程内存后
    
    `uvmCopyUserPt2UkernelPt(np->pagetable, np->kernel_pt, 0, np->sz);`
    
- exec --> 对现在进程替换内存后
    
    `uvmCopyUserPt2UkernelPt(p->pagetable, p->kernel_pt, 0, p->sz);`
    
    `// 对已有进程切换建立新映像的用户内核页表` 
    
    注意：这里是重用旧映像的用户内核页表，重写映射
    
- sbrk --> 调整完进程堆内存后

```c
// 对growproc改造
*int*growproc(*int* *n*)
{
  *uint* sz;
  *struct* *proc* *****p **=** myproc();
  sz **=** p->sz;
  **if**(*n* **>** 0){
    **if** (PGROUNDUP(sz **+** *n*) **>=** PLIC) {
      **return** **-**1;
    }
    **if**((sz **=** uvmalloc(p->pagetable, sz, sz **+** *n*)) **==** 0) {
      **return** **-**1;
    }
  } **else** **if**(*n* **<** 0){
    sz **=** uvmdealloc(p->pagetable, sz, sz **+** *n*);
  }
  **// 坑：p->sz还没更新 是旧地址空间范围边界 sz是sbrk后新返回来的地址空间边界**  
	**uvmCopyUserPt2UkernelPt(p->pagetable, p->kernel_pt, p->sz, sz);**   
  p->sz **=** sz;
  **return** 0;
}
```

[Lab pgtbl.pdf](Lab%20Page%20table%20cd291a9a9c2e42439a2a2f3487c6834c/Lab_pgtbl.pdf)