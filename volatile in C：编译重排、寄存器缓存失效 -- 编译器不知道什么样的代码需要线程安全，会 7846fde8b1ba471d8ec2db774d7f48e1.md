# volatile in C：编译重排、寄存器缓存失效 -- 编译器不知道什么样的代码需要线程安全，会假设所有代码单线程执行

Status: Completed
板块: 内存管理

[编译乱序(Compiler Reordering)](http://www.wowotech.net/kernel_synchronization/453.html)

-- 最终解决

[Linux内核中的READ_ONCE和WRITE_ONCE宏_Roland_Sun的专栏-CSDN博客_linux read_once](https://blog.csdn.net/Roland_Sun/article/details/107365134)

[胡轩： C/C++ 中的 RISC-V 内联汇编 - 20220309 - PLCT实验室_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1rT4y1U7K6)

# volatile

译：易变的

volatile本身的含义就代表了其修饰的变量 **易变的性质** 可能会 带来什么样的**问题**

## volatile修饰变量作用

编译器不要优化后面的代码、变量(每次从内存中读，而不是将值缓存到寄存器中)

<详见下述代码>

## barrier()作用

1. 防止编译器重排，栅栏作用，前后的load、store指令们都不能任意交换顺序 （即：将代码逻辑分两段）
2. **使得每次去内存中(可能是SRAM缓存 但有MESI协议保证一致 且对CPU透明)读取值，而不是在寄存器中读被缓存的数据 <所以这也是在内核中没必要使用volatile的原因 并发数据结构可以用锁 锁里有barrier>**
    
    > Tip 这里讨论的barrier与[并行编程中的barrier](https://en.wikipedia.org/wiki/Barrier_(computer_science))不是同一个东西
    > 

[内存屏障的来历 - 知乎.pdf](volatile%20in%20C%EF%BC%9A%E7%BC%96%E8%AF%91%E9%87%8D%E6%8E%92%E3%80%81%E5%AF%84%E5%AD%98%E5%99%A8%E7%BC%93%E5%AD%98%E5%A4%B1%E6%95%88%20--%20%E7%BC%96%E8%AF%91%E5%99%A8%E4%B8%8D%E7%9F%A5%E9%81%93%E4%BB%80%E4%B9%88%E6%A0%B7%E7%9A%84%E4%BB%A3%E7%A0%81%E9%9C%80%E8%A6%81%E7%BA%BF%E7%A8%8B%E5%AE%89%E5%85%A8%EF%BC%8C%E4%BC%9A%207846fde8b1ba471d8ec2db774d7f48e1/%E5%86%85%E5%AD%98%E5%B1%8F%E9%9A%9C%E7%9A%84%E6%9D%A5%E5%8E%86_-_%E7%9F%A5%E4%B9%8E.pdf)

## volatile什么时候用？

大多数情况下不要使用，基本上涉及到操作memory-mapped I/O时使用，但例如在Linux Kernel中这些都被封装到了`ioremap()`, `writel()`, `readl()` 这样的函数

# volatile修饰的变量为什么会”易变”，不应该是本身的属性吗？

一般情况下，我们在我们写的代码里，直接对变量进行值的改变，那么编译器肯定在编译代码时，一定不会对这样的变量优化成常量。

但是在至少以下几种情况下，

volatile修饰符对应的修饰符是register

一个函数在执行时会使用多个局部变量，包括形参。这些变量使用频度差异较大，编译器会按照使用频度来规划，将常用变量放到闲置寄存器里，使得运算速度得到很大的提高。不常用的那些就放内存里，每次用到就去内存里拿。一个值存放在寄存器和内存里，访问速度的差别可达数百倍甚至上千倍。可见将变量放在寄存器里的意义还是很大的。

但对于可能被抢占的任务，一个变量放在寄存器里就成了问题。因为被抢占后，同一个变量在内存里和寄存器里的值可能会不一样。我这里用了抢占这个词，包括抢占式多任务操作系统里的多进程或多线程调度，也包括嵌入式开发里的中断，当然其实底层都是中断。对非抢占式多任务则不存在这个问题，比如协程。

volatile的含义就是明确告诉编译器，这个变量在每次访问时，都走内存，而不要用寄存器来缓存。这样在抢占式多任务里，就能确保每次拿到最新的值。而register的意思则相反，告诉编译器这个变量尽可能放到寄存器里，以提高速度。当然如果寄存器不够用，那就还是放内存里。

实际使用中，如果一个变量仅在函数内使用，或作为值传递的形参，那么适合使用register修饰符。而如果是全局变量，且可能被多人任务读写，则应该明确的使用volatile。比如中断中访问的全局变量，就应该用volatile。

操作系统课程没仔细看的人，可能难以理解为何寄存器与内存里的值会不同。如下说一下中断发生时，CPU的动作。

中断发生时，CPU会立即把当前所有寄存器的值存入任务的自己的内存区域里。空出所有寄存器，交给中断去做事。中断处理函数返回后，CPU再把需要唤醒的任务的内存区里寄存器部分内容一一载入到寄存器里，并跳转到上次中断的地址，传入PC指针。这样任务就可以继续执行了。这里的关键就是中断结束后恢复到寄存器的值是从任务私有内存区载入的，而不是从原始变量载入的。所以中断期间对变量的修改就无法立即反应到寄存器里。

抢占式多任务的具体实现也是利用了中断。在中断处理函数中协调任务优先级和调用，原理同上

![CSAPP](volatile%20in%20C%EF%BC%9A%E7%BC%96%E8%AF%91%E9%87%8D%E6%8E%92%E3%80%81%E5%AF%84%E5%AD%98%E5%99%A8%E7%BC%93%E5%AD%98%E5%A4%B1%E6%95%88%20--%20%E7%BC%96%E8%AF%91%E5%99%A8%E4%B8%8D%E7%9F%A5%E9%81%93%E4%BB%80%E4%B9%88%E6%A0%B7%E7%9A%84%E4%BB%A3%E7%A0%81%E9%9C%80%E8%A6%81%E7%BA%BF%E7%A8%8B%E5%AE%89%E5%85%A8%EF%BC%8C%E4%BC%9A%207846fde8b1ba471d8ec2db774d7f48e1/Untitled.png)

CSAPP

`volatile` is needed if you are reading from a spot in memory that, say, a completely separate process/device/whatever may write to.

# In C/C++ -- 告诉编译器该variable是可能会变化的 防止编译器对该变量进行任何优化 (maybe优化成常量)

[https://en.wikipedia.org/wiki/Volatile_(computer_programming)#:~:text=Example of memory-mapped I/O in C[edit]](https://en.wikipedia.org/wiki/Volatile_(computer_programming)#:~:text=Example%20of%20memory%2Dmapped%20I/O%20in%20C%5Bedit%5D)

volatile关键字在用C语言编写嵌入式软件里面用得很多，不使用volatile关键字的代码比使用volatile关键字的代码效率要高一些，但就无法保证数据的一致性。volatile的本意是告诉编译器，此变量的值是易变的，每次读写该变量的值时务必从该变量的内存地址中读取或写入，不能为了效率使用对一个“临时”变量的读写来代替对该变量的直接读写。编译器看到了volatile关键字，就一定会生成内存访问指令，每次读写该变量就一定会执行内存访问指令直接读写该变量。若是没有volatile关键字，编译器为了效率，只会在循环开始前使用读内存指令将该变量读到寄存器中，之后在循环内都是用寄存器访问指令来操作这个“临时”变量，在循环结束后再使用内存写指令将这个寄存器中的“临时”变量写回内存。在这个过程中，如果内存中的这个变量被别的因素（其他线程、中断函数、信号处理函数、DMA控制器、其他硬件设备）所改变了，就产生数据不一致的问题。另外，寄存器访问指令的速度要比内存访问指令的速度快，这里说的内存也包括缓存，也就是说内存访问指令实际上也有可能访问的是缓存里的数据，但即便如此，还是不如访问寄存器快的。缓存对于编译器也是透明的，编译器使用内存读写指令时只会认为是在读写内存，内存和缓存间的数据同步由CPU保证。

### Example of signel:

某个变量在某个程序中看起来是不会改变的，因此编译器可能就会将该值优化成const常量，例如以下代码可能陷入到死循环中

```c
static int foo;

void bar(void) {
    foo = 0;

    while (foo != 255)
         ;
}

// HardWare: (NOT CODE)
Interrupt doSignel() {
		foo++;
}
```

优化后：

```c
void bar_optimized(void) {
    foo = 0;

    while (true)
         ;
}

// HardWare: (NOT CODE)
Interrupt doSignel() {
		foo++;
}
```

但是对于foo变量还是可能是被在任意时刻改变，还是有一些不通过代码可以改变这个值的方法的

例如：

1. 硬件会改变该值
2. 另一个正在running的线程使用这个变量
3. signal handler可能改变

### Example of compiler reorder

```c
#define barrier() __asm__ __volatile__("": : :"memory")
 
int flag, data;
 
void foo(int value)
{
	data = value;
	barrier();
	flag = 1;
}

// __asm__ __volatile__  or asm volatile 中 volatile是asm-qualifiers
// 是禁止对asm后代码优化，防止asm后代码被删除
// https://gcc.gnu.org/onlinedocs/gcc/Extended-Asm.html#Volatile
// https://stackoverflow.com/questions/14449141/the-difference-between-asm-asm-volatile-and-clobbering-memory

// 如果没有barrier，编译器可能先改变flag再填充data
// 不能使得还没有填充数据就把填充数据的flag给执行，一旦发生线程切换，其他线程可能看到flag先变1
// 从而读取data，这是错误的
// **当然我们可以在多线程共享变量间使用锁，锁的内部加锁实现一定有barriar**
```

## Linux内核开发：大多数情况不应该使用volatile

[Why the "volatile" type class should not be used - The Linux Kernel documentation](https://www.kernel.org/doc/html/latest/process/volatile-considered-harmful.html#volatile-considered-harmful)

[[RFC/PATCH] doc: volatile considered evil](https://lwn.net/Articles/233482/)

---

1. volatile
    1. volatile能干什么
    2. 为什么Linux Kernel中强烈不建议使用volatile (LWN文章)
    3. 有什么好的替代吗
    4. READ_ONCE WRITE_ONCE一定能保证不会发生乱序重排和原子读写吗
        
        (有条件的 是当且仅当操作变量是READ_ONCE、WRITE_ONCE变量才能)
        

‣ 

1. volatile功能：**抑制编译器优化(将值缓存到寄存器中，每次读取从寄存器中读取，判断为不变值则优化为常量)**（然而优化一般是我们想要的）
2. 在正确编写的内核代码中，volatile只会降低代码的性能
    
    [为什么我们不应该使用volatile类型](https://zhuanlan.zhihu.com/p/102406978)
    
    - 对于map IO寄存器有专用封装，不用使用volatile
    - 对于其他大部分情况，也不必使用，因为只会失去优化从而降低效率
        - **同步场景：**
        对于设计到并发竞争代码区域，我们会使用内核原语（spinlocks，mutexes，memory barriers(会告诉编译器将barrier前后的代码不要重排)等，确保了并发访问共享数据的安全；并且信号量、锁也实现了多线程并发同步的作用
            
            > 如果能正确的使用这些同步原语，当然同时也就 **没有必要** 使用volatile类型，若是使用我们依然需要内核原语来同步，并且使用了volatile会导致编译器对临界区内部也禁止优化，这个就不是我们想要的，这个只会降低性能(因为当持有锁后，`share_data`并不是易变（volatile）的)
            > 
        - **忙等待场景：-- 建议使用cpu_relax()**
            
            ```c
            // 相比memory barrier多了个yield，适当放弃CPU占用使用
            static inline void cpu_relax(void)
            {
                   asm volatile("yield" ::: "memory");
            }
            
            while (my_variable!= what_i_want)
            		cpu_relax();
            ```
            
            我们需要CPU spin等待my_variable等于what_i_want
            
            不合理的做法是：
            
            1. 声明my_variable是volatile变量，然后循环体内为空 
            
            or 
            
            1. my_variable为普通变量，循环体内为barrier
            
            我们要付出的代价就是CPU自身计算资源都被浪费到循环等待上了
            
            更好的做法是在循环体内使用cpu_relax()，其会降低cpu使用功耗或者在超线程处理器上会主动让出使用资源。当然它也充当了一条compiler barrier，使得前后的load指令不被优化，都从内存中读
            
            因此这里声明成volatile变量不是最佳选择，cpu_relax既可以防止编译器优化使得对my_variable读取每次从内存中取，也可以降低CPU功耗
            
            [蜗窝讨论区](http://www.wowotech.net/forum/viewtopic.php?id=21)
            
            [对优化说不 - Linux中的Barrier](https://zhuanlan.zhihu.com/p/96001570)
            
            [内存屏障相关--barrier(),mb(),smp_mb(),乱序,内存一致性_Adam040606的博客-CSDN博客_smp_mb](https://blog.csdn.net/Adam040606/article/details/50898070)
            
            > CPU越过内存屏障后，将刷新自己对存储器的缓冲状态(从内存中读/合法CPU Cache缓存读)。这条语句实际上不生成任何代码，但可使gcc在barrier()之后刷新寄存器对变量的分配
            > 
            > 
            > ```c
            > 1. barrier()就是compiler提供的屏障，作用是告诉compiler内存中的值已经改变，
            > 之前对内存的缓存（缓存到寄存器）都需要抛弃，barrier()之后的内存操作需要重新从内存load，
            > 而不能使用之前寄存器缓存的值。
            > 2. 并且可以防止compiler优化barrier()前后的内存访问顺序。
            > barrier()就像是代码中的一道不可逾越的屏障，barrier前的 load/store 操作不能跑到barrier后面；
            > 同样，barrier后面的 load/store 操作不能在barrier之前。
            > 
            > #define barrier() __asm__ __volatile__("": : :"memory")
            > 
            > int a = 5, b = 6;
            > barrier();
            > a = b;
            >  
            > // 在line 3，GCC不会用存放b的寄存器给a赋值，而是invalidate b的
            > // Cache line，重新读内存中的b值，赋值给a
            > ```
            > 
        - 未使用锁保护资源的多线程并发 -- READ_ONCE WRITE_ONCE
            
            [READ_ONCE()](https://zhuanlan.zhihu.com/p/102753962)
            
            **语义：对资源访问临时转为volatile变量**
            
                 使得资源首先不会被优化为常量，对资源的访问不再从寄存缓存中读取，而是从内存中读取
            
            ```c
            // 通过暂时将相关变量转换为volatile类型来工作
            #define ACCESS_ONCE(x) (*(volatile typeof(x) *)&(x))
            
            for (;;) {
                    struct task_struct *owner;
            
                    owner = ACCESS_ONCE(lock->owner);
                    if (owner && !mutex_spin_on_owner(lock, owner))
                            break;
                    /* ... */
            
            // 编译器可能会得出以下结论：由于所讨论的代码实际上并未修改lock-> owner，
            // 因此没有必要在每次循环中实际获取其值 然后，编译器可能会将代码重新排列为
            owner = ACCESS_ONCE(lock->owner);
            for (;;) {
                    if (owner && !mutex_spin_on_owner(lock, owner))
                            break;
            ```
            
            仅在**没有持有锁（或显式障碍）**的情况下**访问共享数据**的地方，才需要类似ACCESS_ONCE()的构造
            
            > 编译器能进行这样的优化很重要的原因是：并发不是C语言的内置特性，编译器会任务代码执行是单线程的，因此由于并发使用的代码带来的同步和共享资源的共享、使用和保护需要我们建立在上层去做，ACCESS_ONCE就是这样上层同步共享资源，防止因被编译器看作单线程使用而被优化的一种机制
            > 
            
            [编译乱序](https://zhuanlan.zhihu.com/p/102370222)
            
        
        [[RFC/PATCH] doc: volatile considered evil](https://lwn.net/Articles/233482/)
        
        [kernel-sanitizers/READ_WRITE_ONCE.md at master · google/kernel-sanitizers](https://github.com/google/kernel-sanitizers/blob/master/other/READ_WRITE_ONCE.md)
        

---

# 参考：

[https://www.zhihu.com/question/66896665/answer/248114968](https://www.zhihu.com/question/66896665/answer/248114968)

[https://stackoverflow.com/questions/72552/why-does-volatile-exist](https://stackoverflow.com/questions/72552/why-does-volatile-exist)

[https://stackoverflow.com/questions/246127/why-is-volatile-needed-in-c](https://stackoverflow.com/questions/246127/why-is-volatile-needed-in-c)

[https://stackoverflow.com/questions/19923352/whats-the-difference-of-the-usage-of-volatile-between-c-c-and-c-java#:~:text=ensure that such accesses aren't optimised away by the compiler](https://stackoverflow.com/questions/19923352/whats-the-difference-of-the-usage-of-volatile-between-c-c-and-c-java#:~:text=ensure%20that%20such%20accesses%20aren%27t%20optimised%20away%20by%20the%20compiler).

[https://zh.wikipedia.org/wiki/Volatile变量](https://zh.wikipedia.org/wiki/Volatile%E5%8F%98%E9%87%8F)

[https://componenthouse.com/2016/12/28/comparing-the-volatile-keyword-in-java-c-and-cpp/](https://componenthouse.com/2016/12/28/comparing-the-volatile-keyword-in-java-c-and-cpp/)