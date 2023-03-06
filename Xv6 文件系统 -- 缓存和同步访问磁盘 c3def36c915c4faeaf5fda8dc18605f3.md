# Xv6 文件系统 -- 缓存和同步访问磁盘

板块: IO与文件系统

# Xv6的IO是一种同步阻塞式IO，利用这个和outstanding计数(在同一commit内FS syscall/事务数)保证 FS syscall 原子性

# Xv6的七层文件系统

![Untitled](Xv6%20%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%20--%20%E7%BC%93%E5%AD%98%E5%92%8C%E5%90%8C%E6%AD%A5%E8%AE%BF%E9%97%AE%E7%A3%81%E7%9B%98%20c3def36c915c4faeaf5fda8dc18605f3/Untitled.png)

# 1. Xv6 文件系统布局

<aside>
💡 P.S. 文件系统的制作是由mkfs/mkfs.c制作

</aside>

文件系统眼中的磁盘是一个 巨型数组， 将数据结构放置在数组中，重启后又能根据从这些数据结构恢复文件系统

![Untitled](Xv6%20%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%20--%20%E7%BC%93%E5%AD%98%E5%92%8C%E5%90%8C%E6%AD%A5%E8%AE%BF%E9%97%AE%E7%A3%81%E7%9B%98%20c3def36c915c4faeaf5fda8dc18605f3/Untitled%201.png)

文件系统以块(block)分配硬盘存储空间，一个块在XV6中为1024bytes，其将硬盘存储空间编好块号(block number)，并以此寻址

## boot

引导操作系统启动

## super

描述文件系统 文件系统自身的基本数据(file system meta-data)；以块为单位的文件系统大小、数据块的数量、inode的数量和日志中的块数

super block在xv6由另外的`mkfs`程序所填充，其建立最初的文件系统

## log

![Untitled](Xv6%20%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%20--%20%E7%BC%93%E5%AD%98%E5%92%8C%E5%90%8C%E6%AD%A5%E8%AE%BF%E9%97%AE%E7%A3%81%E7%9B%98%20c3def36c915c4faeaf5fda8dc18605f3/Untitled%202.png)

**这个n值是crash recovery与否的关键**

## inodes

此区域一个block中有许多inodes信息

一个inode代表一个文件，其中包含的两点信息如右图所示，包含：

1. **文件元数据**
    - 文件类型
    - 文件硬链接数
    - 文件大小
    - ...
2. **文件数据域块号索引**
    - 12个直接块号索引(可通过内含的块号直接找到文件数据块)
    - 1个间接块号索引(通过内含块号找到一个内含 多个数据块直接块号索引 数据的一级间接索引块)

![Untitled](Xv6%20%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%20--%20%E7%BC%93%E5%AD%98%E5%92%8C%E5%90%8C%E6%AD%A5%E8%AE%BF%E9%97%AE%E7%A3%81%E7%9B%98%20c3def36c915c4faeaf5fda8dc18605f3/Untitled%203.png)

对于xv6中的一个inode的数据块块号索引中出现了一个一级索引，当然我们还可以实现如下的多级索引，多建几层数据块的“目录”

![Untitled](Xv6%20%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%20--%20%E7%BC%93%E5%AD%98%E5%92%8C%E5%90%8C%E6%AD%A5%E8%AE%BF%E9%97%AE%E7%A3%81%E7%9B%98%20c3def36c915c4faeaf5fda8dc18605f3/Untitled%204.png)

> **目录文件**
> 

上面我们提到了**目录**也是一个特殊的文件，只不过这个文件的数据域是本层目录的 文件名--块号 的entries。
如下图：
目录inode的直接块号索引找到了目录文件的数据块，其数据块由一个个entry构成，一个entry代表该目录下的一个文件，entry保存了目录下该文件的inode、目录项/entry长度、文件名长度和文件名

![Untitled](Xv6%20%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%20--%20%E7%BC%93%E5%AD%98%E5%92%8C%E5%90%8C%E6%AD%A5%E8%AE%BF%E9%97%AE%E7%A3%81%E7%9B%98%20c3def36c915c4faeaf5fda8dc18605f3/Untitled%205.png)

在Xv6中我们将其表示成为左图，一个目录文件的一个data block为1024bytes，其中一个entry为16bytes，一共可以表示256个entry

![Untitled](Xv6%20%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%20--%20%E7%BC%93%E5%AD%98%E5%92%8C%E5%90%8C%E6%AD%A5%E8%AE%BF%E9%97%AE%E7%A3%81%E7%9B%98%20c3def36c915c4faeaf5fda8dc18605f3/Untitled%206.png)

## bitmap

位图域维护一个data blocks的位图，某位值表示某个数据块是否被占用

## data

这里是存放文件内容数据的地方，分配资源时，以块(block)为单位

# 文件七层抽象(自底向上)

![Untitled](Xv6%20%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%20--%20%E7%BC%93%E5%AD%98%E5%92%8C%E5%90%8C%E6%AD%A5%E8%AE%BF%E9%97%AE%E7%A3%81%E7%9B%98%20c3def36c915c4faeaf5fda8dc18605f3/Untitled.png)

# 1. Buffer Cache(Linux中叫Page Cache)

## 概览

buffer cache占用一定的内存大小，在用户程序想使用文件时将文件blocks加载到buffer cache中，并且一个block在buffer cache中的copy同一时刻仅供一条内核线程使用

buffer cache使用LRU算法回收在内存中(buffer cache中)的block copy，供新blocks加载到buffer cache时分配

sleeplock保证同一block在某一时刻只能有一个内核线程使用

## buffer cache作用

- **同步**：保证硬盘上的一个block在内存中**只有一份拷贝** 且 仅有**一个内核线程**使用这个拷贝
- **缓存**：缓存热数据(实现在bio.c)

## API

- bread: 在内存中的对硬盘上的一个block的copy-- `buf` 供这个接口使用
- brelse: 对copy操作完毕释放对此的独占(因为对block的使用要保证某时仅有一个内核线程)
- bwrite: 将在内存中修改过的block的copy写入硬盘

## 实现

### 1. 数据结构

双向循环链表

不同位置的节点表示不同的访问热度：最近被线程操作结束很可能再次有线程来使用的block buf放在链表靠头位置，已经被线程操作并释放完很久的buffer节点位于链表的末端，这就是我们选择新buffer供予新disk block分配的目标。

![Untitled](Xv6%20%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%20--%20%E7%BC%93%E5%AD%98%E5%92%8C%E5%90%8C%E6%AD%A5%E8%AE%BF%E9%97%AE%E7%A3%81%E7%9B%98%20c3def36c915c4faeaf5fda8dc18605f3/Untitled%207.png)

buffer cache为固定大小(`struct buf buf[NBUF]`)

![Untitled](Xv6%20%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%20--%20%E7%BC%93%E5%AD%98%E5%92%8C%E5%90%8C%E6%AD%A5%E8%AE%BF%E9%97%AE%E7%A3%81%E7%9B%98%20c3def36c915c4faeaf5fda8dc18605f3/Untitled%208.png)

每一个供给内核线程使用的block在内存中的缓存`struct buf`都是由一些附加信息+数据域`data`组成，并且使用sleeplock保证一个硬盘中的block某刻仅供一个内核线程使用。

valid表示该块是否可以从disk中加载数据

**refcnt用于表示在刻有多少内核进程想使用/正在使用该块(某刻只能有一个内核线程在用)**

- **为什么每个block buffer都使用sleeplock来保护自己？**
    
     最主要是Xv6没有使用DMA进行数据传输，依靠UART的中断进行中断处理，以此来获取数据(持有spinlock时必须关中断)
    
    （但其实就算有DMA在数据传输开始和结束的时候都还是需要中断让CPU与DMA互动委派数据总线的使用权的）
    
    ![Untitled](Xv6%20%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%20--%20%E7%BC%93%E5%AD%98%E5%92%8C%E5%90%8C%E6%AD%A5%E8%AE%BF%E9%97%AE%E7%A3%81%E7%9B%98%20c3def36c915c4faeaf5fda8dc18605f3/Untitled%209.png)
    

### 2. 图解使用buffer cache

![34E73513-2280-4466-8177-4590E5DCC927.jpeg](Xv6%20%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%20--%20%E7%BC%93%E5%AD%98%E5%92%8C%E5%90%8C%E6%AD%A5%E8%AE%BF%E9%97%AE%E7%A3%81%E7%9B%98%20c3def36c915c4faeaf5fda8dc18605f3/34E73513-2280-4466-8177-4590E5DCC927.jpeg)

## bread -- 持有对block的唯一读写使用权

内核线程向buffer cache请求获取dev设备的块号为block的块，bread调用其代理bget获取这个块在buffer cache中的缓存struct buf*

**bread使用睡眠锁**

![Untitled](Xv6%20%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%20--%20%E7%BC%93%E5%AD%98%E5%92%8C%E5%90%8C%E6%AD%A5%E8%AE%BF%E9%97%AE%E7%A3%81%E7%9B%98%20c3def36c915c4faeaf5fda8dc18605f3/Untitled%2010.png)

## bget -- bread的代理

其bget先是在buffer cache中寻找，如果找不到该块就**使用LRU从链表自尾向前寻找一个引用为0的buffer使用**，标记buf的valid位为0，最终向硬盘读取加载到buffer cache中来

如果一个内核线程想使用某个buf，那么就会直接给其buf的refcnt进行+1操作计数，不必等到该内核线程真正使用上

![Untitled](Xv6%20%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%20--%20%E7%BC%93%E5%AD%98%E5%92%8C%E5%90%8C%E6%AD%A5%E8%AE%BF%E9%97%AE%E7%A3%81%E7%9B%98%20c3def36c915c4faeaf5fda8dc18605f3/Untitled%2011.png)

注意：我们在获取到相应block的buf后要对buf加**睡眠锁**仅供当前内核线程使用，当后来的想使用该buf线程会get stuck在bget第一个循环上，触发睡眠锁

<aside>
🔒 我们可以看到在struct bcache中选一个struct buf后，我们不需要从申请到该buf用来存data block后 一直到彻底释放的过程中 对该struct buf加自旋锁表示这个块已经被装载data block，正在使用，不能被装载新的data block吗？

不可以

1. 在struct buf被多个进程轮流使用对应硬盘绑定data block期间不可以加自旋锁，自旋锁会关闭中断，无法实现数据在CPU<->内存<->硬盘的传输(这依靠UART中断进行)
2. 在struct buf中使用一个refcnt标记就好，表示该struct buf已经被装载且被多少线程计划引用，只要bcache在每次分配struct buf装载data block前检查refcnt是否为0即可防止对buf的重复使用
</aside>

## bwrite -- 实际将内存中的数据写入硬盘

![Untitled](Xv6%20%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%20--%20%E7%BC%93%E5%AD%98%E5%92%8C%E5%90%8C%E6%AD%A5%E8%AE%BF%E9%97%AE%E7%A3%81%E7%9B%98%20c3def36c915c4faeaf5fda8dc18605f3/Untitled%2012.png)

内核线程在调用完bread获取到对block的使用权后，对`struct buf` 更新完毕就可以调用bwrite将数据更新到硬盘，若线程不再使用该block则可以release

## brelse -- 释放对一个block的唯一读写使用权（释放睡眠锁）

当一个内核线程完成对一个block的使用占有后(读bread写bwrite结束**后**不再使用)：

1. 先 对block buffer的使用后需要对其独占使用权(sleeplock)进行释放
2. 接着触发LRU规则将buf节点移动至**链表头部**，表示该buffer最近被使用，这样同样引用等待该block buffer的内核线程就会被唤醒进行抢占该资源

![Untitled](Xv6%20%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%20--%20%E7%BC%93%E5%AD%98%E5%92%8C%E5%90%8C%E6%AD%A5%E8%AE%BF%E9%97%AE%E7%A3%81%E7%9B%98%20c3def36c915c4faeaf5fda8dc18605f3/Untitled%2013.png)

`refcnt--`代表对该block使用减少一个线程，仅当refcnt == 0时该struct buf在bcache中可再次被利用装载其他disk data block

# 2. Logging -- 有序组织利用Buffer Cache

文件系统crash之后的问题的解决方案，其实就是logging。这是来自于数据库的一种解决方案。

# Problems -- 逻辑上视为原子性的多次write操作的实现

有一些连续的多个写的操作在对文件的操作逻辑上应该被看做成原子的一个操作，然而实际执行起来不是原子进行的（写inode属性到硬盘、写bitmap标记数据block占用与否，log同步到硬盘log域），crash可能发生在以上多个写操作中的某一个，那么后续的写操作如果没有特殊措施，那么等系统恢复时应该都是后续操作丢失了

后果可能严重可能不严重，可能inode没有被释放，可能多个inode使用同一个block

# Solution：

我们借用数据库中的事务(transaction)的概念，对内核线程的写操作记录日志。

## log记录写操作们

对于进程的写操作，Xv6是这样做的：
1. 一个syscall操作，大小在合理范围内的写操作可以看成一个原子的事务，那么syscall记录其涉及到的所有的写操作成一个日志(log)，当所有的写操作被打包记录成一个日志时，向disk写一个特殊的commit，表示该日志为complete状态(所有写操作记录成log的任务完成)
2. 之后这个syscall再去将所有要真正写入的数据从log域一次事务中缓存的data block(s) copies 至disk相应需要被写入的data block位置
3. 当写操作完成，syscall还会在disk清除该log

## Implementation

![Untitled](Xv6%20%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%20--%20%E7%BC%93%E5%AD%98%E5%92%8C%E5%90%8C%E6%AD%A5%E8%AE%BF%E9%97%AE%E7%A3%81%E7%9B%98%20c3def36c915c4faeaf5fda8dc18605f3/Untitled%2014.png)

## log 域在硬盘上的布局

log域有固定的大小，我们看到log域分为两部分：

- A header block：
    - 计数N：在一次已经提交的事务/commit中，涉及到了对N个data blocks的写
    - Slot Array：记录在这次已经提交的事务/commit中，涉及到的N个data block的block numbers集合
- Log blocks: (中转功能)
    
    对改写了的data block在内存中的**struct buf**进行缓存，等待被写到相应的data block
    

## 数据结构

- superblock中记录了log域的起始块号和给log域分配了多少个块
- struct log：由头部`struct logheader`和其他log域附属信息组成
    
    ![Untitled](Xv6%20%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%20--%20%E7%BC%93%E5%AD%98%E5%92%8C%E5%90%8C%E6%AD%A5%E8%AE%BF%E9%97%AE%E7%A3%81%E7%9B%98%20c3def36c915c4faeaf5fda8dc18605f3/Untitled%2015.png)
    
- `n`
    
    作为counter，其值有两种情况 -- 0与非0
    
    - 0值：目前在log域中没有事务，日志已经从bcache中到达硬盘log域且被完全转移至home location
    - 非0值：这个事务中目前被修改的block数
        - 如果是在crash recovery时检测硬盘log域该值，若为非0证明一定是之前commit了但没有写入home location，该事务完整，则恢复执行
        - 如果是在内存中操作block时未commit前的值，那只能说明当前事务中，这个事务中目前被修改的block数，不能说明已经被commit到硬盘log域了
    
    > **全局变量`struct log log;`中n值计数规则：**
    事务开始之前：内存中为0
    事务开始记录众多写操作 到 commit前：内存中为非0值，动态变化
    开始commit log cache block到commit log cache block结束之前：磁盘上为0
    开始commit log header到commit log header结束之前：磁盘上为0
    开始将磁盘上log cache block写入home location到写入结束之前：磁盘上为非0值
    完成事务清除log：磁盘上为0值
    > 
- `outstanding`
    
    当前正在进行(调用了begin_op()但没有调用end_op()) 的FS Syscall数/事务数
    
    **当outstanding为0即在本次commit中没有事务在执行后：执行commit (最后一个事务的end_op()触发)**
    
- `committing`
    
    值：0 or 1
    
    是否正在committing中（commit、install home location、bzreo meta-data）
    

## 如何保证事务原子性：

<aside>
💡 **澄清概念：
事务：一对 begin_op() 和 end_op() 组成一个事务，保证要么全发生，要么什么都不发生
commit：一次commit中可能只有一个事务，也可能有多个事务(通过struct log中的outstanding标记) -- 可以任务一个commit是个更大的事务**

</aside>

1. 我们开始一个事务通过begin_op()标志事务开始
    
    ![Untitled](Xv6%20%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%20--%20%E7%BC%93%E5%AD%98%E5%92%8C%E5%90%8C%E6%AD%A5%E8%AE%BF%E9%97%AE%E7%A3%81%E7%9B%98%20c3def36c915c4faeaf5fda8dc18605f3/Untitled%2016.png)
    
    当log不在写入硬盘时，且有足够的空间容纳新日志写时：
    
    对全局struct log->outstanding++ 表示在本次commit中，多一个事务，并且该事务占有该commit中，别开始commit提交
    
2. 在此事务中，可能多次写操作我们每次都要进行日志记录
    
    ![writei()](Xv6%20%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%20--%20%E7%BC%93%E5%AD%98%E5%92%8C%E5%90%8C%E6%AD%A5%E8%AE%BF%E9%97%AE%E7%A3%81%E7%9B%98%20c3def36c915c4faeaf5fda8dc18605f3/Untitled%2017.png)
    
    writei()
    
    ![Untitled](Xv6%20%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%20--%20%E7%BC%93%E5%AD%98%E5%92%8C%E5%90%8C%E6%AD%A5%E8%AE%BF%E9%97%AE%E7%A3%81%E7%9B%98%20c3def36c915c4faeaf5fda8dc18605f3/Untitled%2018.png)
    
    log_write()的逻辑就是我们将程序希望的写操作暂存到我们内存中的log域中，并及时更改struct log的header信息
    
3. 以filewrite为例，在事务结束后，end_op()进行标记
    
    ![Untitled](Xv6%20%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%20--%20%E7%BC%93%E5%AD%98%E5%92%8C%E5%90%8C%E6%AD%A5%E8%AE%BF%E9%97%AE%E7%A3%81%E7%9B%98%20c3def36c915c4faeaf5fda8dc18605f3/Untitled%2019.png)
    
    ![Untitled](Xv6%20%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%20--%20%E7%BC%93%E5%AD%98%E5%92%8C%E5%90%8C%E6%AD%A5%E8%AE%BF%E9%97%AE%E7%A3%81%E7%9B%98%20c3def36c915c4faeaf5fda8dc18605f3/Untitled%2020.png)
    
    对本次事务结束标记outstanding--，并对可能因为日志空间不足的写进程唤醒，如果本次事务结束仅仅是多次commit中其中之一，并不是最后一个，那就结束；但一旦发现本次事务是这个commit中的最后一个，那么就执行commit操作并标记正在commit中，别开始新的commit新的事务。
    
    ## 累计到一个commit中的 多个系统调用的写操作们(多个事务) 的并发 -- group commit
    
    **我们可以观察到：**
    
    在一次commit中：刚开始`log.outstanding` 为0，进程A的begin_op()执行完毕`log.outstanding++`进入进程A的事务中，此时`log.outstanding` 为1，进程B此时开始执行begin_op()`log.outstanding++` ，使得`log.outstanding` 为2，若此时进程A的事务执行完毕，调用end_op()使得`log.outstanding--`后，`log.outstanding` 为1，进程A在end_op中判断`log.outstanding` 还不为0，不能进行commit提交，此时进程A就结束他的使命。一旦进程B完成他的事务，调用end_op()使得`log.outstanding--` ，检测到此时`log.outstanding` 为0，那么就触发commit提交到硬盘，接着完成写入commit log to the disk log partation，write the log partation to home locations的工作 
    
    ![记得commit结束要恢复n为0并更新该日志为0到硬盘](Xv6%20%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%20--%20%E7%BC%93%E5%AD%98%E5%92%8C%E5%90%8C%E6%AD%A5%E8%AE%BF%E9%97%AE%E7%A3%81%E7%9B%98%20c3def36c915c4faeaf5fda8dc18605f3/Untitled%2021.png)
    
    记得commit结束要恢复n为0并更新该日志为0到硬盘
    
    **commit()的工作有五个部分，如果crash发生在write_head结束之前那么就无法恢复本次commit，发生在log已经完整写入disk后是可以在crash后完整恢复该commit中所有的事务的**
    
    ![Untitled](Xv6%20%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%20--%20%E7%BC%93%E5%AD%98%E5%92%8C%E5%90%8C%E6%AD%A5%E8%AE%BF%E9%97%AE%E7%A3%81%E7%9B%98%20c3def36c915c4faeaf5fda8dc18605f3/Untitled%2022.png)
    
    将内存中的struct log的缓存数据域struct buf先写入硬盘的log域中的缓存区
    
    ![Untitled](Xv6%20%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%20--%20%E7%BC%93%E5%AD%98%E5%92%8C%E5%90%8C%E6%AD%A5%E8%AE%BF%E9%97%AE%E7%A3%81%E7%9B%98%20c3def36c915c4faeaf5fda8dc18605f3/Untitled%2023.png)
    
    再将内存中的struct log中的header写入硬盘中log域的header部分
    
    ![Untitled](Xv6%20%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%20--%20%E7%BC%93%E5%AD%98%E5%92%8C%E5%90%8C%E6%AD%A5%E8%AE%BF%E9%97%AE%E7%A3%81%E7%9B%98%20c3def36c915c4faeaf5fda8dc18605f3/Untitled%2024.png)
    
    再完成对硬盘中的log域数据向他们自己的home location迁移的过程，如果这一步没有执行完毕就crash了也不要紧，因为本次commit的所有事务修改的data block有多少、块号是哪些、更新后的块内容是什么统统已经写入到了硬盘的log域，开机检测恢复执行install_trans就好
    
    自此之后，本次commit中所有事务全部结束，我们也要将内存和磁盘中的log域清除，因为数据已经安全写入到他们的目的地了。
    
    ### 恢复方法
    
    在reboot后，任何线程开始执行之前：
    
    - 检查硬盘上的log域，查看头部n值，非0则表明上次crash**发生在** **完成了向硬盘log域commit** 但是 在此之后的从硬盘log域向home home location迁徙 并 **清除log之前**，我们可以保证写操作恢复：因此我们会将log中的写操作内容重新replay一遍，并清理log域
        
        ![Untitled](Xv6%20%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%20--%20%E7%BC%93%E5%AD%98%E5%92%8C%E5%90%8C%E6%AD%A5%E8%AE%BF%E9%97%AE%E7%A3%81%E7%9B%98%20c3def36c915c4faeaf5fda8dc18605f3/Untitled%2025.png)
        
        ![Untitled](Xv6%20%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%20--%20%E7%BC%93%E5%AD%98%E5%92%8C%E5%90%8C%E6%AD%A5%E8%AE%BF%E9%97%AE%E7%A3%81%E7%9B%98%20c3def36c915c4faeaf5fda8dc18605f3/Untitled%2026.png)
        
        ![Untitled](Xv6%20%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%20--%20%E7%BC%93%E5%AD%98%E5%92%8C%E5%90%8C%E6%AD%A5%E8%AE%BF%E9%97%AE%E7%A3%81%E7%9B%98%20c3def36c915c4faeaf5fda8dc18605f3/Untitled%2027.png)
        
    - 如果是硬盘log域header的n值为0，表示crash发生在commit log到硬盘log域之前，将其操作记录忽略，数据丢失
    
    <aside>
    💡 所以说当一个日志所属的所有写操作真正完成后，日志应该是不见了的，如果crash后日志还在说明那一连串写到home location操作失败，还是会恢复写操作保证其成功，但如果日志n值为0，则说明日志不完整或者无日志，那么将被recovery code忽略，这些写操作丢失
    
    </aside>
    
    **∴** 这样就保证了一个commit log中的写操作，要么全部执行了，要么全部没有执行
    
    ## 特性：快恢复
    
    这种快速恢复（Fast Recovery）：在重启之后，我们不需要做大量的工作来修复文件系统，只需要非常小的工作量。这里的快速是相比另一个解决方案来说，在另一个解决方案中，你可能需要读取文件系统的所有block，读取inode，bitmap block，并检查文件系统是否还在一个正确的状态，再来修复。而logging可以有快速恢复的属性。
    

# sys_write()的原子性？

参考：

[再探Linux内核write系统调用操作的原子性_Netfilter,iptables/OpenVPN/TCP guard:-(-CSDN博客_write系统调用](https://blog.csdn.net/dog250/article/details/78879600)

[What If Two Processes Write to the Same File Simultaneously](https://walkerlala.github.io/archive/what-if-write-to-the-same-file.html)

- MIT 6.S081 2020 中是这样解释系统调用的原子性的：
    
    <aside>
    💡 它可以确保文件系统的系统调用是**原子性**的。比如你调用**create/write**系统调用，这些系统调用的效果是要么完全出现，要么完全不出现，这样就避免了一个系统调用只有部分写磁盘操作出现在磁盘上。
    
    对于这个说法，我们只能说在写大小在一定范围内数据时，是上述描述的原子的
    
    </aside>
    
    ## 多个进程使用write系统调用写一个文件时，write能保证的是什么？
    
    ### 问题引入 与 背景介绍：
    
    ![*The Linux Programming Interface*](Xv6%20%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%20--%20%E7%BC%93%E5%AD%98%E5%92%8C%E5%90%8C%E6%AD%A5%E8%AE%BF%E9%97%AE%E7%A3%81%E7%9B%98%20c3def36c915c4faeaf5fda8dc18605f3/Untitled%2028.png)
    
    *The Linux Programming Interface*
    
    在Xv6和Linux中，关于 文件描述符 <-> 系统全局打开文件描述表 <-> 系统inode表 的对应关系如Figure 5-2所示，其中inode唯一对应文件系统中唯一的文件。（见sys_open()实现）
    
    在Xv6中，每个进程有一个文件描述符表：
    
    ![Untitled](Xv6%20%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%20--%20%E7%BC%93%E5%AD%98%E5%92%8C%E5%90%8C%E6%AD%A5%E8%AE%BF%E9%97%AE%E7%A3%81%E7%9B%98%20c3def36c915c4faeaf5fda8dc18605f3/Untitled%2029.png)
    
    该表的下标即文件描述符，该表中的struct file* 即为系统全局打开文件描述表ftable中的某一条
    
    ![Untitled](Xv6%20%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%20--%20%E7%BC%93%E5%AD%98%E5%92%8C%E5%90%8C%E6%AD%A5%E8%AE%BF%E9%97%AE%E7%A3%81%E7%9B%98%20c3def36c915c4faeaf5fda8dc18605f3/Untitled%2030.png)
    
    可以看到，struct file中struct inode *ip项指向某一文件inode，但inode只能对应唯一的文件
    
    ![Untitled](Xv6%20%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%20--%20%E7%BC%93%E5%AD%98%E5%92%8C%E5%90%8C%E6%AD%A5%E8%AE%BF%E9%97%AE%E7%A3%81%E7%9B%98%20c3def36c915c4faeaf5fda8dc18605f3/Untitled%2031.png)
    
    所以就有了Figure 5-2 的对应关系：
    
    - 多个进程的文件描述符(fork) 或 同一进程的不同两个文件描述符(dup得来) 可以指向同一struct file
    - 多个struct file可以指向同一inode
    - 多个文件描述符各自指向不同的struct file，但struct file们都指向同一inode
    
    由于这样的场景存在，我们就不得不思考关于写完整性、原子事务性的问题。
    
    **在这几种情况下：**
    
    1. 如果是同一进程的两个线程写同一个文件描述符
    
    2. 父子进程各自不同的文件描述符/一个进程内dup的不同两个文件描述符，但指向同一个system open file table中某一项
    
    > 情况1、2还都是指向同一struct file的情况
    > 
    
    3. 两个进程各自打开同一文件创建的两个不同文件描述符，并各自指向的system open file table中的不同项，但这两个不同项又指向同一inode
    
    **问题:**
    
    I. 会确保两个write都写入指定字节大小的数据到文件吗？(一个线程要写1G数据 另一个也写1G 能保证全部写到吗)
    
    II. 会保证一次write调用对文件写的独占吗？
    (假如线程A写'aaaa...aaa'线程B写'bbbb...bbb'会出现'aaababbaab..这样的序列吗)
    
    III. 如果无法做到一次性不中断写完指定大小字节数据，那最终表现是该次写操作被撤销，相当于没执行，还是将已经执行了的留在文件里并对应相应的文件大小
    
    IV. 这三种不同的并行写，内核实现是给谁加锁？inode? 文件描述符? system open file entry? 保证了什么
    
    [write的原子写？-- in Linux Kernel](write%E7%9A%84%E5%8E%9F%E5%AD%90%E5%86%99%EF%BC%9F--%20in%20Linux%20Kernel%20e6de59c87b704d9ab269749abf83b58b.md) in Linux Kernel
    
    ### 解决：
    
    `write() --> sys_write() --> filewrite()`
    
    ![Untitled](Xv6%20%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%20--%20%E7%BC%93%E5%AD%98%E5%92%8C%E5%90%8C%E6%AD%A5%E8%AE%BF%E9%97%AE%E7%A3%81%E7%9B%98%20c3def36c915c4faeaf5fda8dc18605f3/Untitled%2032.png)
    
    我们可以在filewrite()中看到：无论是对同一struct file并发写还是不同struct file但指向同一inode并发写，都是不会出现某个进程在一次writei()中，另一写操作和那个writei()在一次writei()调用中轮流交错写，因为只要想调用writei()进行写都是需要对inode加锁的。
    
    ![Untitled](Xv6%20%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%20--%20%E7%BC%93%E5%AD%98%E5%92%8C%E5%90%8C%E6%AD%A5%E8%AE%BF%E9%97%AE%E7%A3%81%E7%9B%98%20c3def36c915c4faeaf5fda8dc18605f3/Untitled%2033.png)
    
    ![Untitled](Xv6%20%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%20--%20%E7%BC%93%E5%AD%98%E5%92%8C%E5%90%8C%E6%AD%A5%E8%AE%BF%E9%97%AE%E7%A3%81%E7%9B%98%20c3def36c915c4faeaf5fda8dc18605f3/Untitled%2034.png)
    
    **据此，我们可以得知：**
    
    1. **writei()的写是事务原子的，Xv6中实现的writei()中，如果写不到拟定字节大小，那么就放弃该此写，不更新文件inode中的大小信息，并返回-1表示写失败，当然执行iupdate()时inode的struct buf被写入后，返回到write时执行end_op()且outstanding == 0时才写入硬盘才算真正达成目的**
    2. 当某个进程write写入字节小于等于`int max **=** ((MAXOPBLOCKS**-**1**-**1**-**2) **/** 2) ***** BSIZE;` （防止溢出log域）
        
        我们就可以保证：
        
        1. 这次对某个文件的write()仅由当前进程的写占有，其他进程不可能写该文件**（由于在单此writei前都会给inode加锁）**
            
            > 若进程A写"aaa...aaa"(**小于等于max字节**)，进程B写"bbb..bb"(**小于等于max字节**)，两个进程都去写同一文件，先抢到inode锁的先完整写完，才能轮到另一个，文件不会出现"aaabababaaabbb.."这样的序列
            > 
        2. 本次写要么发生了，要么没发生（inode没更新文件size，filewrite() return -1）
            
            ![Untitled](Xv6%20%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%20--%20%E7%BC%93%E5%AD%98%E5%92%8C%E5%90%8C%E6%AD%A5%E8%AE%BF%E9%97%AE%E7%A3%81%E7%9B%98%20c3def36c915c4faeaf5fda8dc18605f3/Untitled%2035.png)
            
    3. 如果一次write()调用拟定大小大于max值，write()在写完一次max大小字节后由于inode解锁可以被其他对同样文件写进程抢占加锁inode，那么就会出现多进程写交错(以≤max为单位的交错)，但不会产生覆盖，也不能保证write()的写完整性
        
        
    
    <aside>
    💡 但Xv6的实现write方式是一个不明智的，鲁棒性差的，在Linux中write执行完后会返回的大小为实际写入字节大小，而不是只有拟定值和-1两种
    Linux下实现是个复杂的话题，笔者目前也没有太清晰，留作以后思考实验解决 **~~（TODO）~~ 已解决 待整理 [write的原子写？-- in Linux Kernel](write%E7%9A%84%E5%8E%9F%E5%AD%90%E5%86%99%EF%BC%9F--%20in%20Linux%20Kernel%20e6de59c87b704d9ab269749abf83b58b.md)**
    
    </aside>
    
    我们来看看Xv6中，write仅在写入内容小于等于((10**-**1**-**1**-**2) **/** 2) ***** BSIZE时才满足课程中所描述的“原子性”
    
    下面是测试Xv6文件系统一次write最多能保障多少字节完整写入的测试程序。
    
    ```c
    user program:
    
    #include "kernel/types.h"
    #include "kernel/stat.h"
    #include "user/user.h"
    #include "kernel/fs.h"
    #include "kernel/fcntl.h"
    
    void main()
    {
        int fd = open("TEST2", O_CREATE | O_RDWR);
        char buf[((10-1-1-2) / 2) * BSIZE + 10] = {0};
        for (int i = 0; i < ((10-1-1-2) / 2) * BSIZE + 9; i++) {
            buf[i] = 'x';
        }
        printf("%s\n", buf);
        write(fd, buf, ((10-1-1-2) / 2) * BSIZE + 10);
        exit(0);
    }
    ```
    
    ![**begin_op()和end_op()表示一次完整的事务**](Xv6%20%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%20--%20%E7%BC%93%E5%AD%98%E5%92%8C%E5%90%8C%E6%AD%A5%E8%AE%BF%E9%97%AE%E7%A3%81%E7%9B%98%20c3def36c915c4faeaf5fda8dc18605f3/Untitled%2036.png)
    
    **begin_op()和end_op()表示一次完整的事务**
    
    当用户使用系统调用写入`((10-1-1-2) / 2) * BSIZE + 9` bytes内容时，如果write中两次writei都成功进行，那么一切正常，看似能保证一次write syscall原子性
    
    ![Untitled](Xv6%20%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%20--%20%E7%BC%93%E5%AD%98%E5%92%8C%E5%90%8C%E6%AD%A5%E8%AE%BF%E9%97%AE%E7%A3%81%E7%9B%98%20c3def36c915c4faeaf5fda8dc18605f3/Untitled%2037.png)
    
    但是当我们模拟在第二个chunk写入磁盘前突然发生了crash，那么用户的这次write会按预期当彻底没发生过一样吗？不能
    
    ![Untitled](Xv6%20%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%20--%20%E7%BC%93%E5%AD%98%E5%92%8C%E5%90%8C%E6%AD%A5%E8%AE%BF%E9%97%AE%E7%A3%81%E7%9B%98%20c3def36c915c4faeaf5fda8dc18605f3/Untitled%2038.png)
    
    ![Untitled](Xv6%20%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%20--%20%E7%BC%93%E5%AD%98%E5%92%8C%E5%90%8C%E6%AD%A5%E8%AE%BF%E9%97%AE%E7%A3%81%E7%9B%98%20c3def36c915c4faeaf5fda8dc18605f3/Untitled%2039.png)
    
    ## xv6日志实际上非常低效
    
    Xv6的日志系统效率低下。提交不能与文件系统系统调用同时发生。系统会记录整个块，即使一个块中只有几个字节被改变。它执行同步的日志写入，一次写一个块，每一个块都可能需要整个磁盘旋转时间。真正的日志系统可以解决所有这些问题。
    

# 3. Inode

<aside>
💡 一个文件**唯一**对应一个inode

</aside>