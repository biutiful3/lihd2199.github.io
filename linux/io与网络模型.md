---
title: "linux内核前言（五）Io与网络模型"
date: 2021-09-06T20:44:10+08:00
draft: false
audthor: lihongda
categories: ["linux内核"]
tags: ["网络模型","IO"]
---

### 常见的io模型介绍：
#### 阻塞

在linux中，默认情况下所有的socket都是blocking，一个典型的读操作流程大概是这样：

当用户进程调用了recvfrom这个系统调用，kernel就开始了IO的第一个阶段：准备数据（对于网络IO来说，很多时候数据在一开始还没有到达。比如，还没有收到一个完整的UDP包。这个时候kernel就要等待足够的数据到来）。这个过程需要等待，也就是说数据被拷贝到操作系统内核的缓冲区中是需要一个过程的。而在用户进程这边，整个进程会被阻塞（当然，是进程自己选择的阻塞）。当kernel一直等到数据准备好了，它就会将数据从kernel中拷贝到用户内存，然后kernel返回结果，用户进程才解除block的状态，重新运行起来。

所以，blocking IO的特点就是在IO执行的两个阶段都被block了。

#### 非阻塞IO

linux下，可以通过设置socket使其变为non-blocking。当对一个non-blocking socket执行读操作时，流程是这个样子：

当用户进程发出read操作时，如果kernel中的数据还没有准备好，那么它并不会block用户进程，而是立刻返回一个error。从用户进程角度讲 ，它发起一个read操作后，并不需要等待，而是马上就得到了一个结果。用户进程判断结果是一个error时，它就知道数据还没有准备好，于是它可以再次发送read操作。一旦kernel中的数据准备好了，并且又再次收到了用户进程的system call，那么它马上就将数据拷贝到了用户内存，然后返回。

所以，nonblocking IO的特点是用户进程需要**不断的主动询问**kernel数据好了没有。

#### 多线程处理

当有多个socket消息需要处理，阻塞IO搞不定，有一种可能是多个进程/线程，每当有一个连接建立（accept socket)，都会启动一个线程去处理新建立的连接。但是，这种模型性能不太好，创建多进程、多线程时会有开销。

经典的C10K问题，意思是 在一台服务器上维护1w个连接，需要建立1w个进程或者线程。那么如果维护1亿用户在线，则需要1w台服务器。

IO多路复用，则是解决以上问题的场景。

总结：多进程、多线程模型企图把每一个fd放到不同的线程/进程处理，避免阻塞的问题，从而引入了进程创建\撤销，调度的开销。能否在一个线程内搞定所有IO? -- 这就是多路复用的作用。

#### 多路复用

##### select

```c
int select(int maxfd,fd_set *rdset,fd_set *wrset,fd_set *exset,struct timeval *timeout);
```

###### select函数的调用过程

![img](/img/select.png)

（1）使用copy_from_user从用户空间拷贝fd_set到内核空间

（2）注册回调函数__pollwait

（3）遍历所有fd，调用其对应的poll方法（对于socket，这个poll方法是sock_poll，sock_poll根据情况会调用到tcp_poll,udp_poll或者datagram_poll）

（4）以tcp_poll为例，其核心实现就是__pollwait，也就是上面注册的回调函数。

（5）__pollwait的主要工作就是把current（当前进程）挂到设备的等待队列中，不同的设备有不同的等待队列，对于tcp_poll来说，其等待队列是sk->sk_sleep（注意把进程挂到等待队列中并不代表进程已经睡眠了）。在设备收到一条消息（网络设备）或填写完文件数据（磁盘设备）后，会唤醒设备等待队列上睡眠的进程，这时current便被唤醒了。

（6）poll方法返回时会返回一个描述读写操作是否就绪的mask掩码，根据这个mask掩码给fd_set赋值。

（7）如果遍历完所有的fd，还没有返回一个可读写的mask掩码，则会调用schedule_timeout是调用select的进程（也就是current）进入睡眠。当设备驱动发生自身资源可读写后，会唤醒其等待队列上睡眠的进程。如果超过一定的超时时间（schedule_timeout指定），还是没人唤醒，则调用select的进程会重新被唤醒获得CPU，进而重新遍历fd，判断有没有就绪的fd。

（8）把fd_set从内核空间拷贝到用户空间。
###### select睡眠和唤醒过程

　　select巧妙的利用等待队列机制让用户进程适当在没有资源可读/写时睡眠，有资源可读/写时唤醒。

###### select睡眠过程

　　select会循环遍历它所监测的fd_　set内的所有文件描述符对应的驱动程序的poll函数。

　　驱动程序提供的poll函数首先会将调用select的用户进程插入到该设备驱动对应资源的等待队列(如读/写等待队列)，然后返回一个bitmask告诉select当前资源哪些可用。　

　　当select循环遍历完所有fd_set内指定的文件描述符对应的poll函数后，如果没有一个资源可用(即没有一个文件可供操作)，则select让该进程睡眠，一直等到有资源可用为止，进程被唤醒(或者timeout)继续往下执行。

###### select唤醒过程

　　唤醒该进程的过程通常是在所监测文件的设备驱动内实现的。

　　驱动程序维护了针对自身资源读写的等待队列。当设备驱动发现自身资源变为可读写并且有进程睡眠在该资源的等待队列上时，就会唤醒这个资源等待队列上的进程。

原文链接：https://blog.csdn.net/lixungogogo/article/details/52219951


> ### 文件描述符 fd
>
> 文件描述符（file descriptor）是一个非负整数，从 0 开始。进程使用文件描述符来标识一个打开的文件。
>
> 系统为每一个进程维护了一个文件描述符表，表示该进程打开文件的记录表，而**文件描述符实际上就是这张表的索引**。当进程打开（`open`）或者新建（`create`）文件时，内核会在该进程的文件列表中新增一个表项，同时返回一个文件描述符 —— 也就是新增表项的下标。
>
> 一般来说，每个进程最多可以打开 64 个文件，`fd ∈ 0~63`。在不同系统上，最多允许打开的文件个数不同，Linux 2.4.22 强制规定最多不能超过 1,048,576。
>
> 每个进程默认都有 3 个文件描述符：0 (stdin)、1 (stdout)、2 (stderr)。
>
> [fucking-algorithm/linux进程.md at master · labuladong/fucking-algorithm (github.com)](https://github.com/labuladong/fucking-algorithm/blob/master/技术/linux进程.md)




两个问题：

1. 性能开销大
   1. 调用 `select` 时会陷入内核，这时需要将参数中的 `fd_set` 从用户空间拷贝到内核空间
   2. 内核需要遍历传递进来的所有 `fd_set` 的每一位，不管它们是否就绪
2. 同时能够监听的文件描述符数量太少。受限于 `sizeof(fd_set)` 的大小，在编译内核时就确定了且无法更改。一般是 1024，不同的操作系统不相同

别人写的select的应用，可以看看有助了解：[I/O多路复用之select:多用户聊天室学习与开发](https://www.huaweicloud.com/articles/8355434.html)

```c
int Chat(int sock)
{
	//启动两个线程进行读写
	fd_set refd;
	int Maxfd;
	char talk[20];
	int ret;

	Maxfd=sock;
	while(1)
	{
		FD_ZERO(&refd);
		FD_SET(0,&refd);
		FD_SET(sock,&refd);
		ret=select(Maxfd+1,&refd,0,0,0);
		if (ret==-1)break;
		if(FD_ISSET(0,&refd))
		{
			//scanf("%s",talk);
			memset(talk,0,20);
			fgets(talk,20,stdin);
			send(sock,talk,strlen(talk)+1,0);
		}
		else if(FD_ISSET(sock,&refd))
		{
			memset(talk,0,20);
			recv(sock,talk,20,0);
			printf("%s",talk);
		}
	}
	return 0;
}
#endif
```



##### poll

poll 和 select 几乎没有区别。poll 在用户态通过**数组**方式**传递**文件描述符，在内核会转为**链表**方式**存储**，没有最大数量的限制

从性能开销上看，poll 和 select 的差别不大。

##### epoll

epoll ：在内核中的实现不是通过轮询的方式，而是通过注册callback函数的方式。当某个文件描述符发现变化，就主动通知。成功解决了select的两个问题，“epoll 被称为解决 C10K 问题的利器。”

epoll 是对 select 和 poll 的改进，避免了“性能开销大”和“文件描述符数量少”两个缺点。

简而言之，epoll 有以下几个特点：

- 使用**红黑树**存储文件描述符集合
- 使用**队列**存储就绪的文件描述符
- 每个文件描述符只需在添加时传入一次；通过事件更改文件描述符状态

epoll获取事件的时候，无须遍历整个被监听的描述符集，只要遍历那些被内核I/O事件异步唤醒而加入就绪队列的描述符集合。

epoll_create: 创建epoll池子；

epoll_ctl：向epoll注册事件。告诉内核epoll关心哪些文件，让内核没有健忘症；

epoll_wait：等待就绪事件的到来。专门等哪些文件，第2个参数 是输出参数，包含满足的fd，不需要再遍历所有的fd文件；

