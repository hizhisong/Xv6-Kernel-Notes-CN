# RCU -- Read Copy Update

Comment: GC; LockFree; GracePeriod
板块: 同步

# R/W Lock

```cpp
// Here's a simplified version of Linux's read/write lock.
struct rwlock { int n; };
//  n=0  -> not locked
//  n=-1 -> locked by one writer
//  n>0  -> locked by n readers

r_lock(l):
  ****while 1:
    x = l->n
    ****if x < 0
      continue
    if CAS(&l->n, x, x + 1)
      return

// CAS(p, a, b) is atomic compare-and-swap instruction
//  if *p == a, set *p = b, return true
//  else return false

w_lock(l):
  while 1:
    if CAS(&l->n, 0, -1)
      return
```

## R/W Lock缺点

糟糕的性能，锁长时间的等待

# RCU -- 实现并发的算法 [lock-free programming]

- **适用场景**：
    
    大量的数据读取，少量的数据写入场景
    

--> 在读取时避免了加锁成本，仅需要在写入时处理锁问题，可以算是对读写锁对高频读和加解读锁的优化

- 以链表为例**写入的三个场景**：
    1. 原地 **修改** 链表节点内容 -- 此时读可能导致读取到新旧拼接数据
    2. **插入** 一个链表节点 -- 若新节点未初始化就被头插，那么导致读者读到非法数据与非法next
    3. **删除** 一个链表节点 -- 获取到删除的节点空间对其非法访问
    

<aside>
💡 ***** 如果我们完全不想为数据读取者提供任何锁，那么我们需要考虑这三个场景
***** RCU的主要任务就是修复上面的三种数据读取者可能会陷入问题的场景，它的具体做法是让数据写入者变得更加复杂一些，所以数据写入者会更慢一些(但本身的场景就是数据写的频率较少，那么写一次时间略长点也合情合理)
***** 读取者不用有加锁的代价 🙂

</aside>

## 场景1：原地修改一个节点？NO, just Copy Update Insert

- RCU禁止原地修改某一个节点，需要将原节点拷贝，修改，再插入，再释放旧节点([When?](RCU%20--%20Read%20Copy%20Update%202d654d1edbeb482bbf33dc6de7aa9329.md))
    
    注意：我们保证从prev->next = 旧节点切换到新节点是原子的
    
    (一次改动一个内存地址硬件可以保证原子，树、单向链表可以做到，双向链表大多不可)
    
- 因此读者就会在读时面临两个结果：-- 保证了读取单个节点的原子性
    - 读到完整的旧节点
    - 读到完整的新节点
    

## 场景2、3：读写代码前后逻辑被乱序 -- 使用内存屏障 memory barrier

- 计算机中程序执行流不太存在：之后、然后
    
    通常来说所有的编译器和许多微处理器都会重排内存操作
    
- memory barrier禁止编译器和硬件将barrier前后代码交换重排
    
    我们使用memory barrier可以解决
    
    ### Writer
    
    **要保证的正确顺序：**在node初始化执行后才能插入让prev->next指向
    
    ==>在node初始化后，插入之前放入一个barrier，防止交换这两个动作
    
    ### Reader
    
    **要保证的正确顺序：**在node节点地址加载到寄存器后，再对该寄存器存入的节点地址进行访问
    
    ==>在node节点地址加载到寄存器后，对该寄存器存入的节点地址进行访问之前，放入一个barrier，防止交换这两个动作
    

## 保证旧节点被正确释放 -- 上下文切换再释放规约

- 对于Writer来说，我们不能让Reader读取到的旧节点还没有使用完就被释放
    
    那对于节点的释放时机，我们不可能对每个节点进行引用计数，太大的代价，因此采取一下措施：
    
- 保证Reader和Writer遵守如下规则：
    - Reader：数据读取者不允许在context switch时持有一个被RCU保护的数据（也就是链表元素）的指针 / 数据读取者不允许在RCU critical 区域内让出当前CPU -- 同Spinlock的规则
    
    - Writer：**当每个Reader都读过一次后，**CPU核都执行过至少一次context switch之后(保证所有核读取完毕)再释放链表元素
    

<aside>
💡 保证所有核都要发生一次context switch使得writer可以释放旧节点的方法有多种，论文中提到最简单的一个方式是：通过**当每个Reader都读过一次后**，这个过程中每个CPU核必然完成了一次context switching，最后writer就会释放旧元素（由于读者读时不允许发生被调度context switch）

writer调用会**synchronize_rcu()等待所有reader进程读完，完成上下文切换，让write在每个core上都运行一会来保证
因此把synchronize_rcu()改名成wait_for_readers_to_leave()会更加直观**

</aside>

```cpp
// writer:
|
|
node->next = old->next;
BARRIER();
prev->next = node; // 原子的
**synchronize_rcu(); // 阻塞写者直到所有旧数据可能的读者执行完毕 [Grace Period]
									 // 当每个Reader都读过一次后，
									 // synchronize_rcu迫使每个CPU核都发生一次switch
									 // 通过调整线程调度器，使得写入线程简短的在操作系统的每个CPU核上都运行一下，
									 // 发生上下文切换使得占有旧节点的写进程释放节点**
free(old); //此时所有核心抓取到的next节点已经使用完毕(reader的一次switch表示对数据的持有结束)
|
|
```

Tip，可以异步跳过等待所有核switch并在结束执行回调，但是如果有大量的这样调用那么保留的old node太多可能会造成OOM

## RCU代码使用实例

```c
rcu_read_lock()  // 设置**当前CPU核心**的禁止上下文切换 -- 关抢占

rcu_read_unlock() // 设置**当前CPU核心**的允许上下文切换 -- 开抢占
```

```cpp
// List reader using Linux's RCU interface:
  rcu_read_lock()  // 设置**当前CPU核心**的禁止上下文切换
// =========== RCU critical START =============
// 可能读到新可能读到旧 无论读到哪个 所有reader得完成一次完整读并switch才能free old
  e = head
  while(p){
    e = rcu_dereference(e)  // 插入memory barrier，有代价但比spinlock小得多
    look at e->x ...
    e = e->next
  }
// ============= RCU critical END ==============
  rcu_read_unlock() // 设置**当前CPU核心**的允许上下文切换

// Note: code can only use e inside rcu_read_lock/unlock! 因为读到的可能是旧元素即将被释放
//  It cannot e.g. return(e)  但可以返回一个拷贝 不能返回指针/引用(字符串也是指针)

// Code to replace the first list element:

  acquire(lock)
  old = head
  e = alloc()
  e->x = ...
  e->next = head->next
  rcu_assign_pointer(&head, e)  // 设置一个memory barrier 保证所有写操作完毕再assign
  release(lock)

  synchronize_rcu() // 确保任何一个可能持有了旧的链表元素的CPU都执行一次context switch
  free(old)
```

 把synchronize_rcu()改名成wait_for_readers_to_leave()会更加直观

## 新旧数据的讨论

Q:我们可以看到，在一整轮CPU们Context Switch过程中，某个CPU上的Reader可能读取到的还是旧数据而不是新数据，那我们能说明这样还能读取到旧数据是正确的吗？

A: 因为这里数据的读写者是并发的，通常来说如果两件事情是**并发**执行的，你是不会认为它们的执行顺序是确定的。

但RCU论文中的确提到了一些RCU带来的不正确性，但笔者这里不能太理解

### RCU read & write cost

```
RCU performance versus r/w locks?
  For readers:
    RCU imposes nearly **zero cost on reads**.
    r/w locks might take 100s or 1000s of cycles (Figure 5, upper line).
    Also, RCU can read while a write is in progress!
  For **writers**:
    RCU can **take much longer due to synchronize_rcu()**.
    So RCU makes sense when writes are rare or non-time-sensitive.

```

## RCU额外的思考 TODO

RCU能工作的核心思想是为资源释放（Garbage Collection）增加了grace period，在grace period中会确保所有的数据读取者都使用完了数据。所以尽管RCU是一种同步技术，也可以将其看做是一种特殊的GC技术。[https://pdos.csail.mit.edu/6.828/2020/lec/l-rcu.txt](https://pdos.csail.mit.edu/6.828/2020/lec/l-rcu.txt)

RCU也依赖数据结构 — `RCU is not universally applicable.`

```
RCU is not universally applicable.
  **Doesn't help writes.
  Only improves performance when reads >> writes.**
  Doesn't work for code that must hold references across yield/sleep.
    Much like spinlocks.
  Doesn't work if data structure not amenable to single committing write.
    E.g. if readers scan both directions of doubly-linked list.
    Or if in-place updates are required.
  Readers can see stale data.
    E.g. udp_sendmsg() in Figure 6 might send with old IP options.
    But only if concurrent with setsockopt().
    Not a problem here; may be in a few cases (paper mentions Sys V IPC).
  RCU adds complexity to writers.
    Search the web for "review checklist for RCU patches"
  RCU needs context switches to be frequent, and it needs to know about them.
    Easy in the kernel, harder for user-space threads.

```

[https://pdos.csail.mit.edu/6.828/2020/readings/rcu-decade-later.pdf](https://pdos.csail.mit.edu/6.828/2020/readings/rcu-decade-later.pdf)

[http://www2.rdrop.com/users/paulmck/RCU/rclockpdcsproof.pdf](http://www2.rdrop.com/users/paulmck/RCU/rclockpdcsproof.pdf)

[https://lwn.net/Articles/850202/](https://lwn.net/Articles/850202/)

[https://zhuanlan.zhihu.com/p/89439043](https://zhuanlan.zhihu.com/p/89439043)