# External Interrupts(外部中断) -- Hardware wants attention now.

板块: Trap

- syscall call、exceptions、interrupts, they all have same mechianism -- trap
- 中断 与 系统调用及exception的区别：
    - asynchronous 异步的。当硬件生成中断时，Interrupt handler与当前运行的进程在CPU上没有任何关联。但如果是系统调用的话，系统调用发生在运行进程的context下。
    - concurrency。对于中断来说，CPU和生成中断的设备是并行运行的。
        
        网卡自己独立的处理来自网络的packet，然后在某个时间点产生中断，但是同时，CPU也在运行。所以我们在CPU和设备之间是真正的并行的，我们必须管理这里的并行。
        
    - program device 驱动。外部设备，例如网卡，UART，而这些设备需要被编程。
        
        每个设备都有一个编程手册，就像RISC-V有一个包含了指令和寄存器的手册一样。设备的编程手册包含了它有什么样的寄存器，它能执行什么样的操作，在读写控制寄存器的时候，设备会如何响应。
        

> **本节聊的是External Interrupts，来自外部电路板上的设备发生中断时，CPU会做出什么样的响应？**
而不是time interrupts and software interrupts
> 

## 硬件中断怎么传递到CPU

- 类似于读写内存，通过向相应的设备地址执行load/store指令，我们就可以对例如UART的设备进行编程。
- 所有的设备都会被连接到达处理器，处理器上是通过Platform Level Interrupt Control(PLIC **平台级中断控制器**)来处理设备中断。PLIC会管理来自于外设的中断
- 我们拿SiFive举例
    
    ![Untitled](External%20Interrupts(%E5%A4%96%E9%83%A8%E4%B8%AD%E6%96%AD)%20--%20Hardware%20wants%20attent%2008d8b5d1988b4913978580383828913e/Untitled.png)
    
- 我们放大看PILC，PILC用来**路由分发**外部来的中断到CPU核上让CPU核心对中断进行处理，倘若CPU核心都在忙着处理中断，那么PLIC就要先暂时保留这些中断直到有CPU核可以处理

![Untitled](External%20Interrupts(%E5%A4%96%E9%83%A8%E4%B8%AD%E6%96%AD)%20--%20Hardware%20wants%20attent%2008d8b5d1988b4913978580383828913e/Untitled%201.png)

> **具体处理流程：**
PLIC会通知当前有一个待处理的中断
其中一个CPU核会Claim接收中断，这样PLIC就不会把中断发给其他的CPU处理
CPU核处理完中断之后，CPU会通知PLIC
PLIC将不再保存中断的信息
> 

对于PLIC的路由功能，我们是可以自己对PLIC编程的，选择路由不同中断到去哪个核、哪些中断根据优先级先被路由

（e.g. 中断亲和性(对特定中断进行绑核)）

## UART设备驱动

**设备驱动**：管理设备的代码，所有的驱动都位于内核中

大部分**驱动**都**分为** 上半部/下半部 (top/bottom)

> 通用异步收发传输器（Universal Asynchronous Receiver/Transmitter，通常称为UART）是一种异步收发传输器，是电脑硬件的一部分，将数据透过串列通讯和平行通讯间作传输转换。
这里我们使用Qemu来模拟该硬件，来与键盘和Console进行交互。
> 

### top/bottom部

**bottom**部分通常是Interrupt handler。

当一个中断送到了CPU，并且CPU设置接收这个中断，CPU会调用相应的Interrupt handler。Interrupt handler并不运行在任何特定进程的context中，它只是处理中断。

**top**部分，是用户进程或者内核的其他部分 调用 的 **接口**(与用户进程交互/copyin、copyout)。

对于UART来说，这里有read/write接口，这些接口可以被更高层级的代码调用。

通常情况下，驱动中会有一些队列（或者说buffer），top部分的代码会从队列中读写数据，而Interrupt handler（bottom部分）同时也会向队列中读写数据。

注意：这里的buffer存在于内存中，并且只有一份，所以，所有的CPU核都并行的与这一份数据交互。所以我们才需要lock

**这里的队列可以将 并行运行的`设备` 和 `CPU` 解耦开来。**

![Untitled](External%20Interrupts(%E5%A4%96%E9%83%A8%E4%B8%AD%E6%96%AD)%20--%20Hardware%20wants%20attent%2008d8b5d1988b4913978580383828913e/Untitled%202.png)

## Program Device -- 读写硬件外设映射地址其对应外设寄存器

编程是通过memory mapped I/O完成的 (硬件映射到物理地址上)

在SiFive的手册中，设备地址出现在物理地址的特定区间内，这个区间由主板制造商决定。

操作系统需要知道这些设备位于物理地址空间的具体位置，然后再通过普通的load/store指令对这些地址进行编程。load/store指令实际上的工作就是**读写**设备的**控制寄存器**。

例如，对网卡执行store指令时，CPU会修改网卡的某个控制寄存器，进而导致网卡发送一个packet。所以这里的load/store指令不会读写内存，而是会操作设备。

## 两件事: 终端如何打印出$、键入ls终端如何显示出ls

$ -- 设备将$放入UART中的某个寄存器中，当这个字符被发送时UART生成一个中断

在QEMU中，模拟的线路的另一端会有另一个UART芯片（模拟的），这个UART芯片连接到了虚拟的Console，它会进一步将“$ ”显示在console上。

ls -- 对于“ls”，这是用户输入的字符。键盘连接到了UART的输入线路，当你在键盘上按下一个按键，UART芯片会将按键字符通过串口线发送到另一端的UART芯片。另一端的UART芯片先将数据bit合并成一个Byte，之后再产生一个中断，并告诉处理器说这里有一个来自于键盘的字符。之后Interrupt handler会处理来自于UART的字符