# Scheduling in xv6 -- 固定时间片的Round Robin

Comment: 依赖于时钟中断、主动放弃、进程退出、睡眠进行调度
板块: 进程 线程

- 什么时候发生进程切换
    
    使用sleep & wakeup机制
    
    对长期占用CPU寄存但未sleep的进程进行强制sleep
    
- 进程独自占有一个CPU执行 与 内存页分配器和页表给每个进程独自的虚拟内存空间的隔离性一样

- 实现对进程Multiplexing到CPUs上的挑战:
    
    ![Untitled](Scheduling%20in%20xv6%20--%20%E5%9B%BA%E5%AE%9A%E6%97%B6%E9%97%B4%E7%89%87%E7%9A%84Round%20Robin%20571c419ff034455799e1412d9585cbd8/Untitled.png)
    
- Xv6中的用户进程间切换示意图
    
    ![Untitled](Scheduling%20in%20xv6%20--%20%E5%9B%BA%E5%AE%9A%E6%97%B6%E9%97%B4%E7%89%87%E7%9A%84Round%20Robin%20571c419ff034455799e1412d9585cbd8/Untitled%201.png)
    
- Code: Context Switching & Scheduling
    - Xv6为每个CPU各自专门准备了一个调度线程，使用独立的栈空间与寄存器
        
        <aside>
        💡 至此，我们理解的线程有这三种: **用户进程的用户线程**、**用户进程的内核线程**(用户进程在内核中执行时使用的线程，有自己的内核栈和寄存器，例如syscall的调用)、**调度线程**
        
        </aside>
        
    
    ```c
    // Per-CPU process scheduler. (每个CPU都有一个Scheduler，每个CPU遍历所有进程调度)
    // struct cpu提供了每隔调度器进程运行的状态上下文(寄存器) 当跳转到scheduler的时候执行C代码是在machine mode的start.c中定义的stack0(per-CPU的内核栈)
    // Each CPU calls scheduler() after setting itself up.
    // Scheduler never returns.  It loops, doing:
    //  - choose a process to run.
    //  - swtch to start running that process.
    //  - eventually that process transfers control
    //    via swtch back to the scheduler.
    // 关于scheduler的锁：
    // 调用scheduler之前必须持有自身进程的锁
    // (调用sched的有 exit sleep yield(trap(interrupt exception syscall)) 都会锁上当前执行进程的lock)
    // 在scheduler的swtch(&c->context, &p->context);后开始执行 此时又释放当前进程的锁
    // 接下来在PCB数组中找就绪进程 找每个进程前锁进程PCB 进行调度后从进程进入调度器的exit/sleep/yield解锁
    void
    scheduler(void)
    {
      struct proc *p;
      struct cpu *c = mycpu();
      
      c->proc = 0;
      for(;;){
        // Avoid deadlock by ensuring that devices can interrupt.
        intr_on();
        
        int nproc = 0;
        for(p = proc; p < &proc[NPROC]; p++) {
        // >>> p->lock: 对选出的新进程处理 <<<
        // 选择一个进程将其投入运行时，会将该进程的内核线程的context加载到寄存器中，这个阶段不能进入中断
        // 否则进程被修改成running后，但其寄存器值没有被加载全转而就去执行中断，中断又对该内核线程寄存器进行不完整保存到context对象，形成错误
        // 在这种情况下，切换到一个新进程的过程中，也需要获取新进程的锁以确保其他的CPU核不能看到这个进程
          acquire(&p->lock);      // 这里的acquir会在返回后某地释放(yield sleep forkret)
          if(p->state != UNUSED) {
            nproc++;
          }
          if(p->state == RUNNABLE) {
            // Switch to chosen process.  It is the process's job
            // to release its lock and then reacquire it
            // before jumping back to us.
            p->state = RUNNING;
            c->proc = p;
            swtch(&c->context, &p->context);    // 执行swtch后下一步执行的就是ra，也就是放弃CPU时进程执行的代码的位置sched()
    
            // Process is done running for now.
            // It should have changed its p->state before coming back.
            c->proc = 0;
          }
          release(&p->lock);
        }
        if(nproc <= 2) {   // only init and sh exist
          intr_on();
          asm volatile("wfi");
        }
      }
    }
    ```
    
    <aside>
    💡 更多内容：Xv6-book笔记 && proc.c代码注释
    
    </aside>