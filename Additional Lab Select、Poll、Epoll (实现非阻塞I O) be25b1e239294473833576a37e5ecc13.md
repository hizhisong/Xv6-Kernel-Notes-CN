# Additional Lab: Select、Poll、Epoll (实现非阻塞I/O)

Status: Completed
板块: Lab, 同步

# 阻塞与非阻塞I/O 同步与异步I/O

对于文件读写来说，按照当用户发出read读操作后：

1. 进程是等待内核准备好数据，将数据从内核态拷贝到用户态后再继续执行 — **同步I/O**
    
    ![同步阻塞 — 阻塞时CPU没有被浪费，进程只不过让出CPU自己去睡眠了](Additional%20Lab%20Select%E3%80%81Poll%E3%80%81Epoll%20(%E5%AE%9E%E7%8E%B0%E9%9D%9E%E9%98%BB%E5%A1%9EI%20O)%20be25b1e239294473833576a37e5ecc13/Untitled.png)
    
    同步阻塞 — 阻塞时CPU没有被浪费，进程只不过让出CPU自己去睡眠了
    
    ![同步非阻塞 -- 非阻塞 IO 的语义是：试一试，若能完成 IO 就完成；完不成就算了](Additional%20Lab%20Select%E3%80%81Poll%E3%80%81Epoll%20(%E5%AE%9E%E7%8E%B0%E9%9D%9E%E9%98%BB%E5%A1%9EI%20O)%20be25b1e239294473833576a37e5ecc13/Untitled%201.png)
    
    同步非阻塞 -- 非阻塞 IO 的语义是：试一试，若能完成 IO 就完成；完不成就算了
    
    在 Linux 系统中，无论采用阻塞 IO 还是非阻塞 IO，若 IO 已经准备好了，那么会立刻返回．
    
    **阻塞和非阻塞的区别**
    
    仅限于 **IO 尚未准备就绪**的情况下（例如写管道缓冲区已满、读 socket 但尚无数据到达）．这类场景，在在使用**非阻塞** IO 的系统调用时，**系统调用会立刻返回，并通过返回值和 errno** 告诉调用者出现了错误．但若是使用阻塞 IO 的系统调用，则会继续等待直至 IO 完成．(阻塞和非阻塞read)
    
2. 或者是发出读操作后直接返回进程**继续做**其他事宜等待**被通知**数据已准备好并被拷贝到用户进程了 — **异步I/O**
    
    ![Untitled](Additional%20Lab%20Select%E3%80%81Poll%E3%80%81Epoll%20(%E5%AE%9E%E7%8E%B0%E9%9D%9E%E9%98%BB%E5%A1%9EI%20O)%20be25b1e239294473833576a37e5ecc13/Untitled%202.png)
    
    ![Untitled](Additional%20Lab%20Select%E3%80%81Poll%E3%80%81Epoll%20(%E5%AE%9E%E7%8E%B0%E9%9D%9E%E9%98%BB%E5%A1%9EI%20O)%20be25b1e239294473833576a37e5ecc13/Untitled%203.png)
    
    被通知 不需要主动检测(轮询) 数据都被主动拷贝到用户了
    

---

**SUM**

**对同步I/O：**

1. 在  **内核准备数据**  和  将数据从内核态拷贝到用户态  时都等待不返回，这两个过程都被阻塞 ==>**阻塞I/O**
    
    ![Untitled](Additional%20Lab%20Select%E3%80%81Poll%E3%80%81Epoll%20(%E5%AE%9E%E7%8E%B0%E9%9D%9E%E9%98%BB%E5%A1%9EI%20O)%20be25b1e239294473833576a37e5ecc13/Untitled.png)
    
2. 在内核准备数据时返回进程没有被阻塞，中间过程需要自主的轮询内核数据是否已准备好，仅当数据准备好要被拷贝到内核时段，进程被阻塞 ==>**非阻塞I/O**
    
    ![Untitled](Additional%20Lab%20Select%E3%80%81Poll%E3%80%81Epoll%20(%E5%AE%9E%E7%8E%B0%E9%9D%9E%E9%98%BB%E5%A1%9EI%20O)%20be25b1e239294473833576a37e5ecc13/Untitled%201.png)
    

# 基于同步中的非阻塞I/O -- select poll  epoll [内核准备数据时非阻塞(用户read时阻塞)]

[epoll惊群效应深度剖析](https://segmentfault.com/a/1190000039676522)

<aside>
💡 select、poll、epoll表现出同步非阻塞行为与timeout值息息相关，如果他们timeout值设定为某个特殊值，将会使自身行为变为同步阻塞(阻塞时进程会让出CPU去睡眠)

</aside>

<aside>
💡 **IO多路复用**
假设一个进程保持了 10000 条连接，那么如何发现哪条连接上有数据可读了、哪条连接可写了 ，我们不需要建立10000个线程，我们复用这一个进程

如果采用每个线程建立一个链接，一是资源很快被耗完，二是我们会采用循环遍历的方式来发现 IO 事件，但这种方式太低级了。我们希望有一种更高效的机制，在很多连接中的某条上有 IO 事件发生的时候直接快速把它找出来。

其实这个事情 Linux 操作系统已经替我们都做好了，它就是我们所熟知的 **IO 多路复用** 机制。这里的复用指的就是对进程的复用，一个进程可以管理多个连接发现、处理多个连接上的事件
在 Linux 上多路复用方案有 select、poll、epoll。它们三个中 epoll 的性能表现是最优秀的，能支持的并发量也最大

倘若一个 IO 密集型应用（例如一个服务器）那么可能需要同时处理大量的 IO 请求，当然遍历并轮询所有的 IO 是否就绪是一个做法．内核也提供了相关的设施用来完成**遍历并轮询**的操作（**select、poll**） ，但这在同步 IO 中也不是一个最好的做法．

内核还提供了 epoll 这种机制，当内核通过中断机制得知有 IO 事件发生时通知应用程序．这样便**避免了遍历**之苦也提高了 IO 的效率．这也就是 IO 的**多路复用**了

</aside>

我们熟悉的 select/poll/epoll 内核提供给⽤户态的多路复⽤系统调⽤， **进程可以通过⼀个系统调⽤函数从内核中获取多个事件（一个进程拥有多个socket）。**
select/poll/epoll 是如何获取⽹络事件的呢？在获取事件时，先把所有连接（⽂件描述符）传给内核，再由内核返回产⽣了事件的连接，然后在⽤户态中再处理这些连接对应的请求即可。

## select、poll、epoll相同点

都是非阻塞I/O，在内核准备数据时，进程都没有被阻塞挂起，因此用户进程都要使用他们**询问**内核数据是否已经准备完毕，用户进程可以去取了

## select、poll机制

基本原理：

在用户的一次轮询中：

  用户程序将所有已连接socket文件描述符**拷贝**给内核，内核遍历拷贝检测socket集合，有外部读写事件的就绪socket则将其中的socket标记，再把所有的已连接文件描述符整体拷贝给用户空间，用户空间遍历拷贝socket集合，找到被标记读写事件的socket

区别：

1. select内部由数组+位图维护已连接socket集合、poll内部由链表维护已连接socket集合
2. select有监控I/Ofd文件集合大小，而poll没有(链表实现)

### select

```c
int select (int n, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, 
																											struct timeval *timeout);
```

- select 函数监视的文件描述符分3类，分别是writefds、readfds、和exceptfds(异常)
- 调用后select函数会阻塞，直到有描述副就绪（有数据 可读、可写、或者有except），或者超时（timeout指定等待时间，如果立即返回设为null即可），函数返回。
- 当select函数返回后，可以 通过遍历fdset，来找到就绪的描述符。
- timeout控制着select的阻塞行为：当timeout为NULL使得select没有感知到就绪事件时成为同步阻塞IO(也就是在内核准备数据阶段阻塞着)
    
    ![Linux/UNIX系统编程手册](Additional%20Lab%20Select%E3%80%81Poll%E3%80%81Epoll%20(%E5%AE%9E%E7%8E%B0%E9%9D%9E%E9%98%BB%E5%A1%9EI%20O)%20be25b1e239294473833576a37e5ecc13/Untitled%204.png)
    
    Linux/UNIX系统编程手册
    

### poll

同select一样，将**所有**已连接的socket文件描述符组成的**链表 拷贝** 给 内核，内核遍历这些socket看谁上面有事件发生待处理，进行标记后再 **拷贝** 回用户空间

```c
int poll (struct pollfd *fds, unsigned int nfds, int timeout);

// 要监视的文件、请求观察的事件、发生的事件
struct pollfd {
    int fd; /* file descriptor */
    short events; /* requested events to watch */
    short revents; /* returned events witnessed */
};
```

![Untitled](Additional%20Lab%20Select%E3%80%81Poll%E3%80%81Epoll%20(%E5%AE%9E%E7%8E%B0%E9%9D%9E%E9%98%BB%E5%A1%9EI%20O)%20be25b1e239294473833576a37e5ecc13/Untitled%205.png)

> select和poll都需要在返回后，通过遍历文件描述符来获取已经就绪的socket。事实上，同时连接的大量客户端在一时刻可能只有很少的处于就绪状态，因此随着监视的描述符数量的增长，其效率也会线性下降。
> 

## epoll机制

设想一个场景：有100万用户同时与一个进程保持着TCP连接，而每一时刻只有几十个或几百个TCP连接是活跃的(接收TCP包)，也就是说在每一时刻进程只需要处理这100万连接中的一小部分连接。那么，如何才能高效的处理这种场景呢？进程是否在每次询问操作系统收集有事件发生的TCP连接时，把这100万个连接告诉操作系统，然后由操作系统找出其中有事件发生的几百个连接呢？实际上，在Linux2.4版本以前，那时的select或者poll事件驱动方式是这样做的。

这里有个非常明显的问题，即在某一时刻，进程收集有事件的连接时，其实这100万连接中的大部分都是没有事件发生的。因此如果每次收集事件时，都把100万连接的套接字传给操作系统(这首先是用户态内存到内核态内存的大量复制)，而由操作系统内核寻找这些连接上有没有未处理的事件，将会是巨大的资源浪费，然后select和poll就是这样做的，因此它们最多只能处理几千个并发连接。

而epoll不这样做，它在Linux内核中申请了一个简易的文件系统，把原先的一个select或poll调用**分成**了3部分：

`int epoll_create(int size);`

`int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);`

`int epoll_wait(int epfd, struct epoll_event *events,int maxevents, int timeout);`

1. 调用epoll_create建立一个epoll对象(在epoll文件系统中给这个句柄分配资源)；
2. 调用epoll_ctl向epoll对象中添加这100万个连接的套接字；
3. 调用epoll_wait收集发生事件的连接。

**这样只需要在进程启动时建立1个epoll对象，并在需要的时候向它添加或删除连接就可以了，因此，在实际收集事件时，epoll_wait的效率就会非常高，因为调用epoll_wait时并没有向它传递这100万个连接(select poll会这样做)，内核也不需要去遍历全部的连接。(内核有个就绪事件队列/链表)**

```c
int epoll_create(int size)；// 创建一个epoll的实例(返回的文件描述符表示epoll实例)
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)；
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);

struct epoll_event {
  __uint32_t events;  /* Epoll events */
  epoll_data_t data;  /* User data variable */
};

// events可以是以下几个宏的集合：-- 通过与运算相连
/*
EPOLLIN ：表示对应的文件描述符可以读（包括对端SOCKET正常关闭）；
EPOLLOUT：表示对应的文件描述符可以写；
EPOLLPRI：表示对应的文件描述符有紧急的数据可读（这里应该表示有带外数据到来）；
EPOLLERR：表示对应的文件描述符发生错误；
EPOLLHUP：表示对应的文件描述符被挂断；
EPOLLET： 将EPOLL设为边缘触发(Edge Triggered)模式，这是相对于水平触发(Level Triggered)来说的。
EPOLLONESHOT：只监听一次事件，当监听完这次事件之后，如果还需要继续监听这个socket的话，
需要再次把这个socket加入到EPOLL队列里
*/
```

1. `int epoll_create(int size)；` 
    - 创建一个epoll实例，返回的文件描述符表示epoll实例，使用完毕后需close
    - size是监控文件数参考值，不是限制
2. `int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)；`
对epoll实例操作，让其**增加、删除、修改**监控某文件发生了什么事件
    - epfd: epoll实例的文件描述符
    - op：增删改事件的操作 — `EPOLL_CTL_ADD` `EPOLL_CTL_DEL` `EPOLL_CTL_MOD`
    - fd: 要被监控的文件
    - event：被监控文件的事件描述
3. `int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);`
    
    **在timeout事件内，一直阻塞直到：有文件描述符就绪、被信号打断、时间到了才返回**
    
    - 等待epfd上的io事件，最多返回maxevents个事件
    - events用来从内核得到事件的集合，maxevents告之内核这个events集合有多大
    - timeout：-1表示永久阻塞(使得epoll成为同步阻塞)，0表示立即返回就算没有事件就绪
    - 返回值：>0 就绪的文件描述符数
        
                      = 0 时间到
        
                 -1 error
        

![Untitled](Additional%20Lab%20Select%E3%80%81Poll%E3%80%81Epoll%20(%E5%AE%9E%E7%8E%B0%E9%9D%9E%E9%98%BB%E5%A1%9EI%20O)%20be25b1e239294473833576a37e5ecc13/Untitled%206.png)

struct eventpoll {
　　...
　　/*红黑树的根节点，这棵树中存储着所有添加到epoll中的事件，
　　也就是这个epoll监控的事件*/
　　struct rb_root rbr;
　　/*双向链表rdllist保存着将要通过epoll_wait返回给用户的、满足条件的事件*/
　　struct list_head rdllist;
　　...
};

我们在调用epoll_create时，内核除了帮我们在epoll文件系统里建了个file结点，在内核buffer里建了个红黑树用于存储以后epoll_ctl传来的socket外，还会再建立一个rdllist双向链表，用于存储准备就绪的事件，**当epoll_wait调用时，仅仅观察这个rdllist双向链表里有没有数据即可**。有数据就返回，没有数据就sleep(阻塞时间具体待设置)，等到timeout时间到后即使链表没数据也返回。所以，epoll_wait非常高效。

所有添加到epoll中的事件都会与设备(如网卡)驱动程序**建立回调关系**，也就是说相应事件的发生时会调用这里的回调方法。这个回调方法在内核中叫做**ep_poll_callback**，它会把这样的事件放到上面的rdllist双向链表中(每一个epoll监控的事件即在红黑树上也在事件就绪时在双向链表上)

> **执行 socket 就绪回调函数**
> 
> 
> IO事件就绪时，软中断来临，调用 socket 等待队列项 注册函数 ep_poll_callback，软中断接着就会调用它。
> 
> ```c
> //file: fs/eventpoll.c
> static int ep_poll_callback(wait_queue_t *wait,unsigned mode,int sync,void *key)
> {
> //获取 wait 对应的 epitem
> structepitem *epi =ep_item_from_wait(wait);
> 
> //获取 epitem 对应的 eventpoll 结构体
> structeventpoll *ep =epi->ep;
> 
> //1. 将当前epitem 添加到 eventpoll 的就绪队列中(双向链表)
>     list_add_tail(&epi->rdllink, &ep->rdllist);
> 
> //2. 查看 eventpoll 的等待队列上是否有在等待
> if (waitqueue_active(&ep->wq))
>         wake_up_locked(&ep->wq);
> ```
> 
> 在 ep_poll_callback 根据等待任务队列项上的额外的 base 指针可以找到 epitem， 进而也可以找到 eventpoll 对象。
> 
> 首先它做的第一件事就是**把自己的 epitem 添加到 epoll 的就绪队列中(双向链表)**。
> 
> 接着它又会查看 eventpoll 对象上的等待队列里是否有等待项（epoll_wait 执行的时候会设置）。
> 
> 如果没执行软中断的事情就做完了。如果有等待项，那就查找到等待项里设置的回调函数。
> 
> ![Untitled](Additional%20Lab%20Select%E3%80%81Poll%E3%80%81Epoll%20(%E5%AE%9E%E7%8E%B0%E9%9D%9E%E9%98%BB%E5%A1%9EI%20O)%20be25b1e239294473833576a37e5ecc13/Untitled%207.png)
> 
> 其中软中断回调的时候回调函数整理一下：
> 
> sock_def_readable：sock 对象初始化时设置的
> 
> => ep_poll_callback : epoll_ctl 时添加到 socket 上的
> 
> => default_wake_function: epoll_wait 是设置到 epoll 上的
> 
> 总结下，epoll 相关的函数里内核运行环境分两部分：
> 
> - 用户进程内核态。进行调用 epoll_wait 等函数时会将进程陷入内核态来执行。这部分代码负责查看接收队列，以及负责把当前进程阻塞掉，让出 CPU。
> - 硬软中断上下文。在这些组件中，将包从网卡接收过来进行处理，然后放到 socket 的接收队列。对于 epoll 来说，再找到 socket 关联的 epitem，并把它添加到 epoll 对象的就绪链表中。这个时候再捎带检查一下 epoll 上是否有被阻塞的进程，如果有唤醒之。

当调用epoll_wait检查是否有发生事件的连接时，只是检查eventpoll对象中的rdllist双向链表是否有epitem元素而已，如果rdllist链表不为空，则这里的事件复制到用户态内存（使用共享内存**mmap**提高效率）中，同时将事件数量返回给用户。因此epoll_wait效率非常高。epoll_ctl在向epoll对象中添加、修改、删除事件时，从rbr红黑树中查找事件也非常快，也就是说epoll是非常高效的，它可以轻易地处理百万级别的并发连接。

【总结】：

一颗红黑树，一张准备就绪句柄链表，少量的内核cache，就帮我们解决了大并发下的socket处理问题。

- 执行epoll_create()时，创建了红黑树和就绪链表；
- 执行epoll_ctl()时，如果增加socket句柄，则检查在红黑树中是否存在，存在立即返回，不存在则添加到树干上，然后向内核**注册回调函数**，**用于**当中断事件来临时向准备就绪链表中插入数据；
- 执行epoll_wait()时立刻返回准备就绪链表里的数据即可。

![Untitled](Additional%20Lab%20Select%E3%80%81Poll%E3%80%81Epoll%20(%E5%AE%9E%E7%8E%B0%E9%9D%9E%E9%98%BB%E5%A1%9EI%20O)%20be25b1e239294473833576a37e5ecc13/Untitled%208.png)

[epoll原理详解及epoll反应堆模型_~青萍之末~的博客-CSDN博客_epoll详解](https://blog.csdn.net/daaikuaichuan/article/details/83862311)

### Epoll的LT和ET模式

**LT模式(水平触发 默认)**：当epoll_wait检测到描述符事件发生并将此事件通知应用程序，应用程序可以不立即处理该事件。下次调用epoll_wait时，**会**再次响应应用程序并通知此事件。

**ET模式(边缘触发)**：当epoll_wait检测到描述符事件发生并将此事件通知应用程序，应用程序必须立即处理该事件。如果不处理，下次调用epoll_wait时，**不会**再次响应应用程序并通知此事件

![Untitled](Additional%20Lab%20Select%E3%80%81Poll%E3%80%81Epoll%20(%E5%AE%9E%E7%8E%B0%E9%9D%9E%E9%98%BB%E5%A1%9EI%20O)%20be25b1e239294473833576a37e5ecc13/Untitled%209.png)

### 实例代码 — LT模式

```c
// server.c
// 对于TCP连接的建立和传输数据是使用的完全不同的两个队列，一个socket专门用来接收建立连接请求
// 当TCP链接建立后，将给该链接分配一个新的socket负责数据读写

**epfd = epoll_create(MAXEVENTS);**
ev.data.fd = sockfd;
ev.events = EPOLLIN | EPOLLRDHUP | EPOLLERR;

**epoll_ctl(epfd,EPOLL_CTL_ADD,sockfd,&ev);  // 将用于建立连接的socket加入监控**

connect++;

for (;;)   // server无限循环
{
		**alreadyFDNum = epoll_wait(epfd, events, MAXEVENTS, -1); //等待响应**
    for (int i = 0; i < alreadyFDNum; i++)
    {
        connects++;
// ================= 处理事件 =================
// ================= 1. 建立连接 ==============
        if (events[i].data.fd == sockfd)
        {
            if (connects > MAXEVENTS)
            {
                my_err("达到最大连接数...", __LINE__);
                continue;
            }
            connfd = accept(events[i].data.fd, (struct sockaddr *)&cliaddr, &clen);
            printf("客户端[IP：%s],[port: %d]已链接...\n",
                   inet_ntop(AF_INET, &cliaddr.sin_addr.s_addr, ip, sizeof(ip)),
                   ntohs(cliaddr.sin_port));
            if (connfd <= 0)
            {
                my_err("accpet", __LINE__);
                continue;
            }
            ev.data.fd = connfd;
            ev.events = EPOLLIN | EPOLLRDHUP | EPOLLERR;
            epoll_ctl(epfd, EPOLL_CTL_ADD, connfd, &ev); //新增套接字
        }
// ========= 2.用户非正常挂断或程序出错 ===========
				else if((events[i].events & EPOLLIN) && (events[i].events & EPOLLRDHUP) || (events[i].events & EPOLLIN) && (events[i].events & EPOLLERR))
				{
						epoll_ctl(epfd,EPOLL_CTL_DEL,events[i].data.fd,0); //删去套接字
						close(events[i].data.fd);
						connects--;
		    }
// ========= 3. 用户正常发来消息请求 =============
		    else if (events[i].events & EPOLLIN)
		    {
		        //...工作
		    }
}
```

```c
// client.c

int main()
{
    int connfd;
    pthread_t tid1, tid2;
    struct sockaddr_in servaddr;
    if ((connfd = socket(AF_INET, SOCK_STREAM, 0)) < 0)
    { //初始化socket套接字
        my_err("socket", __LINE__);
    }
    bzero(&servaddr, sizeof(struct sockaddr_in));
    servaddr.sin_family = AF_INET;                                      // ipv4协议
    servaddr.sin_port = htons(PORT);                                    //填入端口号
    inet_pton(AF_INET, "127.0.0.1", (void *)&servaddr.sin_addr.s_addr); //转为网络地址格式
    printf("正在与服务器建立连接...\n");
    if ((connect(connfd, (struct sockaddr *)&servaddr, sizeof(servaddr))) < 0)
    { //与服务器链接
        my_err("connect", __LINE__);
    }
    else
        printf("与服务器链接成功！！！\n");

    pthread_create(&ptd1, NULL, send_mission, (void *)&connfd); //发送消息的线程
    pthread_create(&ptd2, NULL, recv_mission, (void *)&connfd); //接收消息的线程

		pthread_join(&ptd1, NULL);
		pthread_join(&ptd2, NULL);

    return 0;
}
```

```c
#define IPADDRESS   "127.0.0.1"
#define PORT        8787
#define MAXSIZE     1024
#define LISTENQ     5
#define FDSIZE      1000
#define EPOLLEVENTS 100

//添加事件
static void add_event(int epollfd,int fd,int state){
    struct epoll_event ev;
    ev.events = state;
    ev.data.fd = fd;
    epoll_ctl(epollfd,EPOLL_CTL_ADD,fd,&ev);
}

//删除事件
static void delete_event(int epollfd,int fd,int state) {
    struct epoll_event ev;
    ev.events = state;
    ev.data.fd = fd;
    epoll_ctl(epollfd,EPOLL_CTL_DEL,fd,&ev);
}

//修改事件
static void modify_event(int epollfd,int fd,int state){     
    struct epoll_event ev;
    ev.events = state;
    ev.data.fd = fd;
    epoll_ctl(epollfd,EPOLL_CTL_MOD,fd,&ev);
}
{
	listenfd = socket_bind(IPADDRESS,PORT);
	
	struct epoll_event events[EPOLLEVENTS];
	
	//创建一个描述符
	epollfd = epoll_create(FDSIZE);
	
	//添加监听描述符事件
	add_event(epollfd,listenfd,EPOLLIN);
	
	//循环等待
	for ( ; ; ){
	    //该函数返回已经准备好的描述符事件数目
	    ret = epoll_wait(epollfd,events,EPOLLEVENTS,-1);
	    //处理接收到的连接
	    handle_events(epollfd,events,ret,listenfd,buf);
	}
}

//事件处理函数
static void handle_events(int epollfd,struct epoll_event *events,int num,int listenfd,char *buf)
{
     int i;
     int fd;
     //进行遍历;这里只要遍历已经准备好的io事件。num并不是当初epoll_create时的FDSIZE。
     for (i = 0;i < num; i++)
     {
         fd = events[i].data.fd;
        //根据描述符的类型和事件类型进行处理
         if ((fd == listenfd) &&(events[i].events & EPOLLIN))
            handle_accpet(epollfd,listenfd);
         else if (events[i].events & EPOLLIN)
            do_read(epollfd,fd,buf);
         else if (events[i].events & EPOLLOUT)
            do_write(epollfd,fd,buf);
     }
}

//处理接收到的连接
static void handle_accpet(int epollfd,int listenfd){
     int clifd;     
     struct sockaddr_in cliaddr;     
     socklen_t  cliaddrlen;     
     clifd = accept(listenfd,(struct sockaddr*)&cliaddr,&cliaddrlen); // new socket fd 
     if (clifd == -1)         
     perror("accpet error:");     
     else {         
         printf("accept a new client: %s:%d\n",inet_ntoa(cliaddr.sin_addr),cliaddr.sin_port);                       //添加一个客户描述符和事件         
         add_event(epollfd,clifd,EPOLLIN);     
     } 
}

//读处理
static void do_read(int epollfd,int fd,char *buf){
    int nread;
    nread = read(fd,buf,MAXSIZE);
    if (nread == -1)     {         
        perror("read error:");         
        close(fd); //记住close fd        
        delete_event(epollfd,fd,EPOLLIN); //删除监听 
    }
    else if (nread == 0)     {         
        fprintf(stderr,"client close.\n");
        close(fd); //记住close fd       
        delete_event(epollfd,fd,EPOLLIN); //删除监听 
    }     
    else {         
        printf("read message is : %s",buf);        
        //修改当前读事件发生的文件描述符对应的事件，由读改为写         
        modify_event(epollfd,fd,EPOLLOUT);     
    } 
}

//写处理
static void do_write(int epollfd,int fd,char *buf) {     
    int nwrite;     
    nwrite = write(fd,buf,strlen(buf));     
    if (nwrite == -1){         
        perror("write error:");        
        close(fd);   //记住close fd       
        delete_event(epollfd,fd,EPOLLOUT);  //删除监听    
    }else{
				// 修改当前写事件发生的文件描述符对应的事件，由写改为读 
        modify_event(epollfd,fd,EPOLLIN);  
    }    
    memset(buf,0,MAXSIZE); 
}
```

## ET or LT

EPOLL事件有两种模型：

Level Triggered (LT) 水平触发
.socket接收缓冲区不为空 有数据可读 读事件一直触发
.socket发送缓冲区不满 可以继续写入数据 写事件一直触发
符合思维习惯，epoll_wait返回的事件就是socket的状态

Edge Triggered (ET) 边沿触发
.socket的接收缓冲区状态变化时触发读事件，即空的接收缓冲区刚接收到数据时触发读事件
.socket的发送缓冲区状态变化时触发写事件，即满的缓冲区刚空出空间时触发读事件
仅在状态变化时触发事件

**LT的处理过程：**
. accept一个连接，添加到epoll中监听EPOLLIN事件
. 当EPOLLIN事件到达时，read fd中的数据并处理
. 当需要写出数据时，把数据write到fd中；如果数据较大，无法一次性写出，那么在epoll中监听EPOLLOUT事件
. 当EPOLLOUT事件到达时，继续把数据write到fd中；如果数据写出完毕，那么在epoll中关闭EPOLLOUT事件

**ET的处理过程：**
. accept一个一个连接，添加到epoll中监听EPOLLIN|EPOLLOUT事件
. 当EPOLLIN事件到达时，read fd中的数据并处理，read需要一直读，直到返回EAGAIN为止
. 当EPOLLOUT事件到达时，继续把数据write到fd中，直到数据全部写完，或者write返回EAGAIN

从ET的处理过程中可以看到，ET的要求是需要一直读写，直到返回EAGAIN，否则就会遗漏事件。而LT的处理过程中，直到返回EAGAIN不是硬性要求，但通常的处理过程都会读写直到返回EAGAIN，但LT比ET多了一个开关EPOLLOUT事件的步骤

LT的编程与poll/select接近，符合一直以来的习惯，不易出错
ET的编程可以做到更加简洁，某些场景下更加高效，但另一方面容易遗漏事件，容易产生bug

## Sum

在 select/poll中，进程只有在调用一定的方法后，内核才对所有监视的文件描述符进行扫描，而epoll事先通过epoll_ctl()来注册一 个文件描述符，一旦基于某个文件描述符就绪时，内核会采用类似callback的回调机制，迅速激活这个文件描述符，当进程调用epoll_wait() 时便得到通知。(此处去掉了遍历文件描述符，而是通过监听回调的的机制。这正是epoll的魅力所在。)

---

## IO多路复用总结

**IO多路复用**是指内核一旦发现进程指定的一个或者多个IO条件准备读取，它就通知该进程。IO多路复用适用如下场合：

（1）当客户处理多个描述字时（一般是交互式输入和网络套接口），必须使用I/O复用。

（2）当一个客户同时处理多个套接口时，而这种情况是可能的，但很少出现。

（3）如果一个TCP服务器既要处理监听套接口，又要处理已连接套接口，一般也要用到I/O复用。

（4）如果一个服务器即要处理TCP，又要处理UDP，一般要使用I/O复用。

（5）如果一个服务器要处理多个服务或多个协议，一般要使用I/O复用

- 与多进程和多线程技术相比，I/O多路复用技术的最大优势是系统开销小，系统不必创建进程/线程，也不必维护这些进程/线程，从而大大减小了系统的开销。
- 多路复用有三种方式
    - select
    - poll
    - epoll
- 三种IO复用应用场景和区别？
    - IO多路复用的本质时通过一种机制，复用的是进程，让单个进程可以监视多个文件描述符，一旦某个描述符就绪，能够通知程序进行相应的读写操作。
    - select：
        - select()的机制中提供一种fd_set的数据结构,就是一个维护了socket的数组
        - select:在网络编程中统一的操作顺序是创建socket－>绑定端口－>监听－>accept->write/read,当有客户端连接到来时,select会把该连接的文件描述符放到fd_set集合中每次调用select，都需要把fd_set集合从用户态拷贝到内核态，如果fd_set集合很大时，那这个开销也很大，内核对被监控的fd_set集合大小做了限制，并且这个是通过宏控制的，大小不可改变(限制为1024)。除此之外在每次遍历这些描述符之前，系统还需要把这些描述符集合从用户空间copy到内核，然后再把有事件到达的文件描述符copy回去，如果此时没有一个描述符有事件发生，这些copy操作和遍历操作都是无用功，可见slect随着连接数量的增多，效率大大降低。可见如果在高并发的场景下select并不适用
    - poll的机制与select类似，与select在本质上没有多大差别，管理多个描述符也是进行轮询，根据描述符的状态进行处理，但是poll没有最大文件描述符数量的限制。poll和select同样存在一个缺点就是，包含大量文件描述符的数组被整体复制于用户态和内核的地址空间之间，而不论这些文件描述符是否就绪，它的开销随着文件描述符数量的增加而线性增大。
    - epoll: epoll是基于事件驱动的IO方式，它的优点就是在于它的效率和并发量。相对于select来说，epoll没有描述符个数限制，使用一个文件描述符管理多个描述符，将用户关心的文件描述符事件存放在内核的一个事件表中，这样用户空间和内核空间的copy只需一次,在内核中是以红黑树节点去存储的。
        - 提高程序在大量并发连接中只有少量活跃的情况下的系统CPU利用率，获取事件的时候，它无须遍历整个被侦听的描述符集，只要遍历那些被内核IO事件异步唤醒而加入Ready队列的描述符集合
        - int epoll_create(int size);
            - 创建一个epoll的句柄，内核帮我们在epoll文件系统里建了个file结点，size用来告诉内核这个监听的数目一共有多大。这个参数不同于select()中的第一个参数，给出最大监听的fd+1的值。需要注意的是，当创建好epoll句柄后，它就是会占用一个fd值，在linux下如果查看/proc/进程id/fd/，是能够看到这个fd的，所以在使用完epoll后，必须调用close()关闭，否则可能导致fd被耗尽。
        - epoll_ctl 的事件注册函数，注册要监听的事件类型
            - 内核用红黑树来存储以后epoll_ctl传来的socket id，当有新的socket连接来时，先遍历红黑书中有没有这个socket存在，如果有就立即返回，没有就插入，然后给内核中断处理程序注册一个回调函数，每当有事件发生时就通过回调函数把这些文件描述符放到事先准备好的用来存储就绪事件的链表中
        - epoll_wait时，会把准备就绪的socket 描述符链表拷贝到用户态内存，然后清空list链表，最后检查这些socket
        - epoll的优势非常明显，几乎没有描述符数量的限制，并发支持完美，不会随着socket的增加而降低效率，也不用在内核空间和用户空间之间做无效的copy操作
    - 两种触发方式
        - 水平触发（LT）：默认工作模式，即当epoll检测到某描述符事件就绪并通知应用程序时，应用程序可以不立即处理该事件；下次调用时，会再次通知此事件,如果这些socket上确实有未处理的事件时，该句柄会再次被放回到刚刚清空的准备就绪链表，保证所有的事件都得到正确的处理，
            - 如果到timeout时间后链表中没有数据也立刻返回，因此在并发需求量高的场景中我们即使要监控数百万计的句柄，大多数一次也只返回很少量的准备就绪句柄。由此可见epoll仅需要从内核态copy少量的句柄到用户态，这样就
        - 边缘触发（ET）： 当epoll检测到某描述符事件就绪并通知应用程序时，应用程序必须立即处理该事件。如果不处理，下次调用不会再次通知此事件。（直到你做了某些操作导致该描述符变成未就绪状态了，也就是说边缘触发只在状态由未就绪变为就绪时只通知一次）
            - 当epoll_wait检测到描述符事件发生并将此事件通知应用程序，应用程序必须立即处理该事件。如果不处理，下次调用epoll_wait时，不会再次响应应用程序并通知此事件。
        - 比较
            - 相比较而与，边缘触发减少了事件的触发次数，效率更高一些。但是需要用户及时去处理事件。
    - epoll的优势非常明显，几乎没有描述符数量的限制，并发支持完美，不会随着socket的增加而降低效率，也不用在内核空间和用户空间之间做无效的copy操作
    - 在并发量低，socket都比较活跃的情况下，select的性能不会比epoll差，而且不用像epoll一样再内核中维护一个红黑树去管理socket
    - 但是当并发量较大时，epoll是Linux目前大规模网络并发程序开发的首选模型，目前流行的高性能web服务器Nginx正式依赖于epoll提供的高效网络套接字轮询服务
    - 应用场景：一个游戏服务器，tcp server负责接收客户端的连接，dbserver负责处理数据信息，一个webserver负责处理服务器的web请求，gameserver负责游戏的逻辑处理，所有这些服务都和另外一个gateserver相连，gateserver负责服务器间的通信和转发（进程间通信），只要游戏服务器在服务状态，这些连接几乎不会断开（异常情况可能会断开），并且这些连接数量一般不会很多。这种情况使用select，因为每时每刻这些连接的socket都有事件发生（比如：服务期间的心跳信息，还有大型网络游戏的同步信息（一般每秒在20-30次）），最重要的是，这种场景下，并发量也不会很大。如果此时用epoll，为此所建立的文件系统，红黑书和链表对于此来说就是杀鸡用牛刀，效率反而不 高。当然这里的tcp server负责大量的客户端的连接，毫无疑问epoll是首选，它接受大量的客户端连接，收到客户端的消息之后把消息转发发给select网络模型的gateserver，gateserver再转发给gameserver进行逻辑处理，最后返回给客户端就over了。因此在如果在并发量低，socket都比较活跃的情况下，select就不见得比epoll慢了(就像我们常常说快排比插入排序快，但是在特定情况下这并不成立)
    - 应用场景
        - 在LT模式下，无论fd是否有事件发生，或者还有一些事件没有处理完，每次调用epoll_wait时，总会得到该fd让你处理（只要有没事件没有处理，会一直通知你处理，直到你处理完为止，这样就保证了数据的不丢失）。
        - 在ET模式下，当有事件发生时，系统只会通知你一次，即在调用epoll_wait返回fd后，不管这个事件你处理还是没处理，处理完没有处理完，当再次调用epoll_wait时，都不会再返回该fd，这样的话程序员要自己保证在时间发生时要及时有效的处理完该事件。
        - 如果是并发量比较大，而且每个连接通信的数据量比较大的情况，为了不饥饿掉其他连接，采用LT触发模式注册可读事件，通信框架中限制每个连接接受的最大数据为1M，假设一个连接中总共会有5M，那当可读事件触发时，就只会读取1M，然后把这份数据缓存在应用层，然后处理其他连接后再回来处理接下来的第二个1M，重复上述逻辑，知道5M数据都取完毕，再把这些数据批量抛给业务逻辑；这样保证了每个业务的吞吐量；2.如果对每个用户实时性要求比较高，就每次尽最大努力读完5M数据后再处理其他可读事件
- 使用Linux epoll模型，水平触发模式；当socket可写时，会不停的触发 socket 可写的事件，如何处理？
    - 第一种最普遍的方式：需要向 socket 写数据的时候才把 socket 加入 epoll ，等待可写事件。接受到可写事件后，调用 write 或者 send 发送数据。当所有数据都写完后，把 socket 移出 epoll。这种方式的缺点是，即使发送很少的数据，也要把 socket 加入 epoll，写完后在移出 epoll，有一定操作代价。
    - 开始不把 socket 加入 epoll，需要向 socket 写数据的时候，直接调用 write 或者 send 发送数据。如果返回 EAGAIN，把 socket 加入 epoll，在 epoll 的驱动下写数据，全部数据发送完毕后，再移出 epoll。这种方式的优点是：数据不多的时候可以避免 epoll 的事件处理，提高效率
- 使用Linux epoll模型，水平触发模式；当socket可写时，会不停的触发 socket 可写的事件，如何处理？
    - 第一种最普遍的方式：需要向 socket 写数据的时候才把 socket 加入 epoll ，等待可写事件。接受到可写事件后，调用 write 或者 send 发送数据。当所有数据都写完后，把 socket 移出 epoll。这种方式的缺点是，即使发送很少的数据，也要把 socket 加入 epoll，写完后在移出 epoll，有一定操作代价。
    - 开始不把 socket 加入 epoll，需要向 socket 写数据的时候，直接调用 write 或者 send 发送数据。如果返回 EAGAIN，把 socket 加入 epoll，在 epoll 的驱动下写数据，全部数据发送完毕后，再移出 epoll。这种方式的优点是：数据不多的时候可以避免 epoll 的事件处理，提高效率