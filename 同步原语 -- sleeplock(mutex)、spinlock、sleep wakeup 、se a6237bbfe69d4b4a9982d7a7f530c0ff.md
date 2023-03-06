# 同步原语 -- sleeplock(mutex)、spinlock、sleep wakeup 、semaphore &&  同步bug[防御性编程]

板块: 同步, 进程 线程

相关实现与代码注释

```c
git checkout lock ==> sleeplock.c spinlock.c semaphore.c sematest.c
```

## 做好防御性编程

**同步bug的根本原因**: 编程语言的缺陷 (软件只是现实世界需求在数字世界的部分投影，可能会丢失很多信息(**信息失真**))

例如: 银行余额overflow (e.g. 无符号值减余额到0再减overflow到一个巨大的值)

解决方案: 使用防御性编程把语言满足的条件表示出来

例如对变量范围的检查: 是否栈溢出 是否超过合法值范围

**`assert(Expected Conditions we want to happend)`**

<aside>
💡 死锁和数据竞争不一定导致严重的后果

</aside>

## 死锁 -- 线程间彼此等待停滞不前(hang住)

两种可能导致死锁的加锁顺序

### AA型

在一个执行流中，先后对同一把锁连续加锁

![18FAB5F4-6EC2-4359-9130-BF8EDD3AE638.jpeg](%E5%90%8C%E6%AD%A5%E5%8E%9F%E8%AF%AD%20--%20sleeplock(mutex)%E3%80%81spinlock%E3%80%81sleep%20wakeup%20%E3%80%81se%20a6237bbfe69d4b4a9982d7a7f530c0ff/18FAB5F4-6EC2-4359-9130-BF8EDD3AE638.jpeg)

我们加锁时，一个是位了提高性能，再一个是在中断里可能会产生数据竞争

如果我们在解锁时直接开，而不是通过对加锁引用计数再解锁倒数到0时再开中断，那么就有可能在spin_unlock(&xxx)后发生中断，在中断handler对之前线程持有的锁再加锁，导致死锁

**解决方案:** 

spinlock实现加锁时: 

1. 关中断 并且进行关中断计数
2. 当前线程不可以再次持有本线程正在持有的锁

       if(holding(lk))   panic();

spinlock解锁时:

仅当关中断技术为0时再开中断，防止中断handler中争用用户已经持有的锁发生死锁

### ABBA型

![108E5A93-A8AA-4A83-BC85-EFBC836F30DB.jpeg](%E5%90%8C%E6%AD%A5%E5%8E%9F%E8%AF%AD%20--%20sleeplock(mutex)%E3%80%81spinlock%E3%80%81sleep%20wakeup%20%E3%80%81se%20a6237bbfe69d4b4a9982d7a7f530c0ff/108E5A93-A8AA-4A83-BC85-EFBC836F30DB.jpeg)

严格按照固定的顺序获得所有锁 (lock ordering; 消除 “循环等待”)

**遇事不决可视化：画加锁顺序图看有没有环**

进而证明 T1:A→B→C;     T2:B→C 是安全的

**在任意时刻总是有获得 “最靠后” 锁的可以继续执行**

## data race

def: 不同的线程 **同时访问** 同一段内存**，且** 至少有一个是写

1. 不写无锁的并发程序
2. 用互斥锁串行化，不让同一时间数据被不同线程操作

## 更多同步bug

1. 违反原子性 -- 忘记上锁 

       我们很容易在写一个函数时认为这个函数的执行是原子的

![6123AEAD-D57B-462B-A39F-DDF929E7598F.jpeg](%E5%90%8C%E6%AD%A5%E5%8E%9F%E8%AF%AD%20--%20sleeplock(mutex)%E3%80%81spinlock%E3%80%81sleep%20wakeup%20%E3%80%81se%20a6237bbfe69d4b4a9982d7a7f530c0ff/6123AEAD-D57B-462B-A39F-DDF929E7598F.jpeg)

1. 顺序违反 --  忘记同步

![AB6A2471-1EB8-407E-930A-2837D4D90877.jpeg](%E5%90%8C%E6%AD%A5%E5%8E%9F%E8%AF%AD%20--%20sleeplock(mutex)%E3%80%81spinlock%E3%80%81sleep%20wakeup%20%E3%80%81se%20a6237bbfe69d4b4a9982d7a7f530c0ff/AB6A2471-1EB8-407E-930A-2837D4D90877.jpeg)

我们期待S2执行完毕后执行S4，但是忘记做了同步，可能导致S4在S2之前执行，最终导致就算donewating，readwriteproc还是要wait，就hang住了

## 同步bug的处理

1. 静态检查分析，检查检查再检查 -- 先相信自己的代码是错的
2. 运行时
    1. 死锁检测
        
        e.g. 工具: **lockdeps** in Linux kernel
        
        原理：检测加锁顺序是否成环
        
    2. data race检测
        
        工具：Thread Sanitizer
        
    
          为所有事件建立happends-before关系图
    
    > **happends-before关系: 描述事件间发生所具有一定的时序关系**
    在同一线程上的 内存操作之间 存在一个先后顺序、用锁串行化的内存操作存在一个先后顺序
    但是如下这种操作4 or 5没有规定与其他事件的happends-before关系，那么就有可能在thread1上发生在任何时候进而造成data race(不同的线程 **同时访问** 同一段内存**，且** 至少有一个是写 )
    > 
    > 
    > ![8C1DDB8B-25E9-4E79-91E1-4DC5FF02DBA3.jpeg](%E5%90%8C%E6%AD%A5%E5%8E%9F%E8%AF%AD%20--%20sleeplock(mutex)%E3%80%81spinlock%E3%80%81sleep%20wakeup%20%E3%80%81se%20a6237bbfe69d4b4a9982d7a7f530c0ff/8C1DDB8B-25E9-4E79-91E1-4DC5FF02DBA3.jpeg)
    > 
    > ![AE48273F-F245-456E-AEA5-B3367478F132.jpeg](%E5%90%8C%E6%AD%A5%E5%8E%9F%E8%AF%AD%20--%20sleeplock(mutex)%E3%80%81spinlock%E3%80%81sleep%20wakeup%20%E3%80%81se%20a6237bbfe69d4b4a9982d7a7f530c0ff/AE48273F-F245-456E-AEA5-B3367478F132.jpeg)
    > 
    
    Thread Sanitizer检测 共享内存 操作时，如果没有明确的happends-before关系的操作存在，那就可能导致 不同的线程 **同时访问** 同一段内存**，且** 至少有一个是写 (数据竞争) 的发生
    
    gcc选项实现的Senitizer
    
    1. Address Sanitizer ===> 非法内存访问运行时检测
    
    -fsanitize=address: 对use after free / overflow mem access / double free进行err(不开这个选项可能就真的就非法修改了，就通过了).  
    
    1. Thread Sanitizer ===> 数据竞争检测
    
    -fsanitize=thread：多线程对未加锁的资源进程访问发生data race 
    
     
    
    ## 实现我们自己简易版本的Lockdep & Senitizer
    
    ### 低配版Lockdep
    
    ```c
    // 低配版Lockdep
    // 不必大费周章统计加锁顺序从而检测是否有环(**有可能**死锁)
    // 统计争用同一锁的次数 大于一个明显不正常的数值就报告
    // 配合GDB在panic上打断点 再看线程的backtrace诊断死锁
    #define LOCKDEP
    
    void
    initlock(struct spinlock *lk, char *name)
    {
      lk->name = name;
      lk->locked = 0;
      lk->cpu = 0;
    }
    
    // Acquire the lock.
    // Loops (spins) until the lock is acquired.
    // acquire自身是不能放弃CPU的，这里一开始就关中断了，目的是为了防止同一CPU上普通线程与中断争用同一把锁导致死锁(sys_sleep 与 clockintr)
    void
    acquire(struct spinlock *lk)
    {
      // 如果不关闭中断，不保证从acquire到release同一把锁这段代码能一次性原子性地执行完，那么就可能导致
      push_off(); // disable interrupts to avoid deadlock.
      if(holding(lk))
        panic("acquire");       // 检测死锁
    
    #ifdef LOCKDEP
    #define SPIN_LIMIT (1000000000)
      int spin_cnt = 1;
    #endif
    
      // On RISC-V, sync_lock_test_and_set turns into an atomic swap:
      //   a5 = 1
      //   s1 = &lk->locked
      //   amoswap.w.aq a5, a5, (s1)
    
      // wrap the swap in a loop, retrying(spinning) until it has acquired the lock
      // amoswap  r, a  -- read the value from register a(address), and put the value into register r(address)
      // __sync_lock_test_and_set wrap the amoswap instruction and returning the old value the register it written
    
      // 对于synchronize指令，任何在它之前的load/store指令，都不能移动到它之后
      // TEST_AND_SET: 将新值设定到变量上并返回变量旧值
      while(__sync_lock_test_and_set(&lk->locked, 1) != 0)      // 实现自旋 与 原子占用操作
      {
        #ifdef LOCKDEP
          if (spin_cnt++ > SPIN_LIMIT) {
            panic("Too many spin!");
          }
        #endif
      }
    
      // Tell the C compiler and the processor to not move loads or stores
      // past this point, to ensure that the critical section's memory
      // references happen strictly after the lock is acquired.
      // On RISC-V, this emits a fence instruction.
      // 禁止CPU和编译器在代码段使用锁时对指令流顺序进行re-order 可能导致锁窗口期的出现
      // 这里使用内存屏障
      // xv6中的屏障几乎在所有重要的情况下都会acquire和release强制顺序，因为xv6在访问共享数据的周围使用锁
      __sync_synchronize();
    
      // Record info about lock acquisition for holding() and debugging.
      lk->cpu = mycpu();
    }
    ```
    
    ### 低配版Sanitizer
    
    Thread Sanitizer ===> 数据竞争检测
    
    内存分配要求：
    
    已分配内存 `S = [L0, R0) 并 [L1, R1) 并 ...`
    
    新分配的内存 kalloc(p) 返回的 `[L, R) 交 S = 空集`则不存在Thread Sanitizer
    
    ```c
    // allocation -- prevent double allocation
    for (int i = 0; (i + 1) * sizeof(u32) <= size; i++) {
      panic_on(((u32 *)ptr)[i] == MAGIC, "double-allocation");
      arr[i] = MAGIC;
    }
    
    // free -- prevent double free
    for (int i = 0; (i + 1) * sizeof(u32) <= alloc_size(ptr); i++) {
      panic_on(((u32 *)ptr)[i] == 0, "double-free");
      arr[i] = 0;
    }
    ```
    
    ## MSVC的做法
    
    1. 屯屯屯
    2. 烫烫烫
    
    （未初始化的填值、free后的填值...）