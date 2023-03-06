# CacheLine的映射

板块: 内存管理

将 虚拟地址 映射成 物理地址 有多种方法

## ****直接映射缓存****

### 结构组成

<aside>
💡 映射算法：内存块号(不是页) % CacheLine数
>> 将内存分为多块

</aside>

例：

内存共被划分为 32 个内存块， CPU Cache 共有 8 个 CPU Line，假设 CPU 想要访问第 15 号内存块，如果 15 号内存块中的数据已经缓存在 CPU Line 中的话，则是⼀定映射在 7 号 CPU Line中，因为 15 % 8 的值是 7。

> **内存需要被划分成多少个大块？**
总内存大小 / 本层Cache总大小

这样就保证每个块内所有的地址空间都有一个CacheLine与之对应
> 

![Untitled](CacheLine%E7%9A%84%E6%98%A0%E5%B0%84%201e6b54b74dc04ea995ed4453b1bfbf79/Untitled.png)

![Untitled](CacheLine%E7%9A%84%E6%98%A0%E5%B0%84%201e6b54b74dc04ea995ed4453b1bfbf79/Untitled%201.png)

因此可能会出现多个内存块映射到同一CacheLine中，我们对CacheLine进行标记：

![Untitled](CacheLine%E7%9A%84%E6%98%A0%E5%B0%84%201e6b54b74dc04ea995ed4453b1bfbf79/Untitled%202.png)

缓存将虚拟地址划分为四部分：

1. **Tag -- 组标记：标记这个CacheLine中数据来源于哪一组**
2. **Index -- 索引：自然序列，区分不同的Cache Line**
3. **Offset -- CPU从Cache中取内容的单位是一个字(64位CPU为一个字为64位)，根据Offset取CacheLine中的Block字（CacheLine内寻址）**

CacheLine中多出一个：

1. Valid bit -- 有效位 ：标记对应的 Cache Line 中的数据是否是有效的，如果有效位是 0，无论 Cache Line 中是否有数据， CPU 都会直接访问内存，重新加载数据

### 查找流程

1. 根据内存地址中索引信息，计算在 CPU Cahe 中的索引，也就是找出对应的 CPU Line 的地址；
    1. 虚拟地址的 索引域 找到对应CacheLine
2. 找到对应CacheLine后，判断 CacheLine 中的有效位，确认 CacheLine中数据是否是有效的，如果是⽆效的， CPU 就会直接访问内存，并重新加载数据，如果数据有效，则往下执⾏；
3. 对⽐内存地址中组标记和 v 中的Tag（确定是哪个内存块在本CacheLine中对应的数据），确认 CacheLine 中的数据是我们要访问的内存数据，如果不是的话， CPU 就会直接访问内存，并重新加载数据，如果是的话，则往下执⾏；
4. 根据内存地址中偏移量信息，从CacheLine的数据块中，读取对应的字

### 直接映射缓存的问题 -- Cache颠簸 (cache thrashing)

连续访问内存中的一组位置，这些位置是同一Cache Index但属于不同内存块的地址，那么就会导致访问这组地址时每次访问都会产生Cache Missing

## 全相连映射 -- 解决Cache颠簸问题手段之一

![Untitled](CacheLine%E7%9A%84%E6%98%A0%E5%B0%84%201e6b54b74dc04ea995ed4453b1bfbf79/Untitled%203.png)

![Untitled](CacheLine%E7%9A%84%E6%98%A0%E5%B0%84%201e6b54b74dc04ea995ed4453b1bfbf79/Untitled%204.png)

主存中的一个地址可被**随意**映射进**任意**cache line

当寻找一个地址是否已经被cache时，需要**遍历每一个**cache line来寻找，这个代价很高。

因此在虚拟地址中没有必要保留Index域，Index域在直接映射中是用来区分虚拟地址对应到哪个CacheLine的

现在只需要用Tag来区分就好，当作是唯一索引

参考：

[主存到Cache直接映射、全相联映射和组相联映射_cany1000的博客-CSDN博客_全相联映射](https://blog.csdn.net/dongyanxia1000/article/details/53392315)

[Cache的基本原理](https://zhuanlan.zhihu.com/p/102293437)

[Cache组织方式](https://zhuanlan.zhihu.com/p/107096130)

小林Coding系统 -- CPU Cache 的数据结构和读取过程是什么样的？