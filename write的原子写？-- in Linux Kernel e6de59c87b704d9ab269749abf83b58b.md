# write的原子写？-- in Linux Kernel

Comment: 已解决 待整理
板块: IO与文件系统

多个进程使用write系统调用写一个文件时，write能保证的是什么？
在这几种情况下：

1. 如果是同一进程的两个线程写同一个文件描述符
2. 父子进程各自不同的文件描述符/一个进程内dup的不同两个文件描述符，但指向同一个system open file table中某一项
3. 两个进程各自打开同一文件创建的两个不同文件描述符，并各自指向的system open file table中的不同项，但这两个不同项又指向同一inode

I. 会确保两个write都写入指定字节大小的数据到文件吗？(一个线程要写1G数据 另一个也写1G 能保证全部写到吗)

II. 会保证一次write调用对文件写的独占吗？
(假如线程A写'aaaa...aaa'线程B写'bbbb...bbb'会出现'aaababbaab..这样的序列吗)

III. 如果无法做到一次性不中断写完指定大小字节数据，那最终表现是该次写操作被撤销，相当于没执行，还是将已经执行了的留在文件里并对应相应的文件大小

IV. 如何理解系统调用的原子性

![Untitled](write%E7%9A%84%E5%8E%9F%E5%AD%90%E5%86%99%EF%BC%9F--%20in%20Linux%20Kernel%20e6de59c87b704d9ab269749abf83b58b/Untitled.png)

![Untitled](write%E7%9A%84%E5%8E%9F%E5%AD%90%E5%86%99%EF%BC%9F--%20in%20Linux%20Kernel%20e6de59c87b704d9ab269749abf83b58b/Untitled%201.png)

![Untitled](write%E7%9A%84%E5%8E%9F%E5%AD%90%E5%86%99%EF%BC%9F--%20in%20Linux%20Kernel%20e6de59c87b704d9ab269749abf83b58b/Untitled.jpeg)