> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s?__biz=Mzg2MzU3Mjc3Ng==&mid=2247483737&idx=1&sn=7ef3afbb54289c6e839eed724bb8a9d6&chksm=ce77c71ef9004e08e3d164561e3a2708fc210c05408fa41f7fe338d8e85f39c1ad57519b614e&scene=178&cur_album_id=2559805446807928833#rd) bin@bin的技术小屋 2022-01-10 11:12

从今天开始我们来聊聊Netty的那些事儿，我们都知道Netty是一个高性能异步事件驱动的网络框架。

它的设计异常优雅简洁，扩展性高，稳定性强。拥有非常详细完整的用户文档。

同时内置了很多非常有用的模块基本上做到了开箱即用，用户只需要编写短短几行代码，就可以快速构建出一个具有`高吞吐`，`低延时`，`更少的资源消耗`，`高性能（非必要的内存拷贝最小化）`等特征的高并发网络应用程序。

本文我们来探讨下支持Netty具有`高吞吐`，`低延时`特征的基石----netty的`网络IO模型`。

由Netty的`网络IO模型`开始，我们来正式揭开本系列Netty源码解析的序幕：

网络包接收流程
-------

![图片](https://mmbiz.qpic.cn/mmbiz_png/sOIZXFW0vUY4ypDTgZibnVV3K2XJZVLcEAVqTQC2CLFPicUHhicvVfVnFGUOEwWI1ueTic9xAtwibD0iaGVmSMxkuJcA/640?wx_fmt=png)网络包收发过程.png

*   当`网络数据帧`通过网络传输到达网卡时，网卡会将网络数据帧通过`DMA的方式`放到`环形缓冲区RingBuffer`中。
    

> `RingBuffer`是网卡在启动的时候`分配和初始化`的`环形缓冲队列`。当`RingBuffer满`的时候，新来的数据包就会被`丢弃`。我们可以通过`ifconfig`命令查看网卡收发数据包的情况。其中`overruns`数据项表示当`RingBuffer满`时，被`丢弃的数据包`。如果发现出现丢包情况，可以通过`ethtool命令`来增大RingBuffer长度。

*   当`DMA操作完成`时，网卡会向CPU发起一个`硬中断`，告诉`CPU`有网络数据到达。CPU调用网卡驱动注册的`硬中断响应程序`。网卡硬中断响应程序会为网络数据帧创建内核数据结构`sk_buffer`，并将网络数据帧`拷贝`到`sk_buffer`中。然后发起`软中断请求`，通知`内核`有新的网络数据帧到达。
    

> `sk_buff`缓冲区，是一个维护网络帧结构的`双向链表`，链表中的每一个元素都是一个`网络帧`。虽然 TCP/IP 协议栈分了好几层，但上下不同层之间的传递，实际上只需要操作这个数据结构中的指针，而`无需进行数据复制`。

*   内核线程`ksoftirqd`发现有软中断请求到来，随后调用网卡驱动注册的`poll函数`，`poll函数`将`sk_buffer`中的`网络数据包`送到内核协议栈中注册的`ip_rcv函数`中。
    

> `每个CPU`会绑定`一个ksoftirqd`内核线程`专门`用来处理`软中断响应`。2个 CPU 时，就会有 `ksoftirqd/0` 和 `ksoftirqd/1`这两个内核线程。

> **这里有个事情需要注意下：** 网卡接收到数据后，当`DMA拷贝完成`时，向CPU发出`硬中断`，这时`哪个CPU`上响应了这个`硬中断`，那么在网卡`硬中断响应程序`中发出的`软中断请求`也会在`这个CPU绑定的ksoftirqd线程`中响应。所以如果发现Linux软中断，CPU消耗都`集中在一个核上`的话，那么就需要调整硬中断的`CPU亲和性`，来将硬中断`打散`到`不通的CPU核`上去。

*   在`ip_rcv函数`中也就是上图中的`网络层`，`取出`数据包的`IP头`，判断该数据包下一跳的走向，如果数据包是发送给本机的，则取出传输层的协议类型（`TCP`或者`UDP`)，并`去掉`数据包的`IP头`，将数据包交给上图中得`传输层`处理。
    

> 传输层的处理函数：`TCP协议`对应内核协议栈中注册的`tcp_rcv函数`，`UDP协议`对应内核协议栈中注册的`udp_rcv函数`。

*   当我们采用的是`TCP协议`时，数据包到达传输层时，会在内核协议栈中的`tcp_rcv函数`处理，在tcp_rcv函数中`去掉`TCP头，根据`四元组（源IP，源端口，目的IP，目的端口）`查找`对应的Socket`，如果找到对应的Socket则将网络数据包中的传输数据拷贝到`Socket`中的`接收缓冲区`中。如果没有找到，则发送一个`目标不可达`的`icmp`包。
    
*   内核在接收网络数据包时所做的工作我们就介绍完了，现在我们把视角放到应用层，当我们程序通过系统调用`read`读取`Socket接收缓冲区`中的数据时，如果接收缓冲区中`没有数据`，那么应用程序就会在系统调用上`阻塞`，直到Socket接收缓冲区`有数据`，然后`CPU`将`内核空间`（Socket接收缓冲区）的数据`拷贝`到`用户空间`，最后系统调用`read返回`，应用程序`读取`数据。
    

性能开销
----

从内核处理网络数据包接收的整个过程来看，内核帮我们做了非常之多的工作，最终我们的应用程序才能读取到网络数据。

随着而来的也带来了很多的性能开销，结合前面介绍的网络数据包接收过程我们来看下网络数据包接收的过程中都有哪些性能开销：

*   应用程序通过`系统调用`从`用户态`转为`内核态`的开销以及系统调用`返回`时从`内核态`转为`用户态`的开销。
    
*   网络数据从`内核空间`通过`CPU拷贝`到`用户空间`的开销。
    
*   内核线程`ksoftirqd`响应`软中断`的开销。
    
*   `CPU`响应`硬中断`的开销。
    
*   `DMA拷贝`网络数据包到`内存`中的开销。
    

网络包发送流程
-------

![图片](https://mmbiz.qpic.cn/mmbiz_png/sOIZXFW0vUY4ypDTgZibnVV3K2XJZVLcEUK82351HYuWhJCicKvej94Cico9gLw6PJFPQBzmSibCplZAJXLoTz9O7w/640?wx_fmt=png)网络包发送过程.png

*   当我们在应用程序中调用`send`系统调用发送数据时，由于是系统调用所以线程会发生一次用户态到内核态的转换，在内核中首先根据`fd`将真正的Socket找出，这个Socket对象中记录着各种协议栈的函数地址，然后构造`struct msghdr`对象，将用户需要发送的数据全部封装在这个`struct msghdr`结构体中。
    
*   调用内核协议栈函数`inet_sendmsg`，发送流程进入内核协议栈处理。在进入到内核协议栈之后，内核会找到Socket上的具体协议的发送函数。
    

> 比如：我们使用的是`TCP协议`，对应的`TCP协议`发送函数是`tcp_sendmsg`，如果是`UDP协议`的话，对应的发送函数为`udp_sendmsg`。

*   在`TCP协议`的发送函数`tcp_sendmsg`中，创建内核数据结构`sk_buffer`,将`struct msghdr`结构体中的发送数据`拷贝`到`sk_buffer`中。调用`tcp_write_queue_tail`函数获取`Socket`发送队列中的队尾元素，将新创建的`sk_buffer`添加到`Socket`发送队列的尾部。
    

> `Socket`的发送队列是由`sk_buffer`组成的一个`双向链表`。

> 发送流程走到这里，用户要发送的数据总算是从`用户空间`拷贝到了`内核`中，这时虽然发送数据已经`拷贝`到了内核`Socket`中的`发送队列`中，但并不代表内核会开始发送，因为`TCP协议`的`流量控制`和`拥塞控制`，用户要发送的数据包`并不一定`会立马被发送出去，需要符合`TCP协议`的发送条件。如果`没有达到发送条件`，那么本次`send`系统调用就会直接返回。

*   如果符合发送条件，则开始调用`tcp_write_xmit`内核函数。在这个函数中，会循环获取`Socket`发送队列中待发送的`sk_buffer`，然后进行`拥塞控制`以及`滑动窗口的管理`。
    
*   将从`Socket`发送队列中获取到的`sk_buffer`重新`拷贝一份`，设置`sk_buffer副本`中的`TCP HEADER`。
    

> `sk_buffer` 内部其实包含了网络协议中所有的 `header`。在设置 `TCP HEADER`的时候，只是把指针指向 `sk_buffer`的合适位置。后面再设置 `IP HEADER`的时候，在把指针移动一下就行，避免频繁的内存申请和拷贝，效率很高。

![图片](https://mmbiz.qpic.cn/mmbiz_png/sOIZXFW0vUY4ypDTgZibnVV3K2XJZVLcEpYMIdLl6Jch1BmZ4hwRpDHIXJIhmKibfPeEib9Y8SDx2YzXibkVWkic3Yg/640?wx_fmt=png)sk_buffer.png

> **为什么不直接使用`Socket`发送队列中的`sk_buffer`而是需要拷贝一份呢？**因为`TCP协议`是支持`丢包重传`的，在没有收到对端的`ACK`之前，这个`sk_buffer`是不能删除的。内核每次调用网卡发送数据的时候，实际上传递的是`sk_buffer`的`拷贝副本`，当网卡把数据发送出去后，`sk_buffer`拷贝副本会被释放。当收到对端的`ACK`之后，`Socket`发送队列中的`sk_buffer`才会被真正删除。

*   当设置完`TCP头`后，内核协议栈`传输层`的事情就做完了，下面通过调用`ip_queue_xmit`内核函数，正式来到内核协议栈`网络层`的处理。
    
    > 通过`route`命令可以查看本机路由配置。
    
    > 如果你使用 `iptables`配置了一些规则，那么这里将检测`是否命中`规则。如果你设置了非常`复杂的 netfilter 规则`，在这个函数里将会导致你的线程 `CPU 开销`会`极大增加`。
    

*   将`sk_buffer`中的指针移动到`IP头`位置上，设置`IP头`。
    
*   执行`netfilters`过滤。过滤通过之后，如果数据大于 `MTU`的话，则执行分片。
    
*   检查`Socket`中是否有缓存路由表，如果没有的话，则查找路由项，并缓存到`Socket`中。接着在把路由表设置到`sk_buffer`中。
    

*   内核协议栈`网络层`的事情处理完后，现在发送流程进入了到了`邻居子系统`，`邻居子系统`位于内核协议栈中的`网络层`和`网络接口层`之间，用于发送`ARP请求`获取`MAC地址`，然后将`sk_buffer`中的指针移动到`MAC头`位置，填充`MAC头`。
    
*   经过`邻居子系统`的处理，现在`sk_buffer`中已经封装了一个完整的`数据帧`，随后内核将`sk_buffer`交给`网络设备子系统`进行处理。`网络设备子系统`主要做以下几项事情：
    

*   选择发送队列（`RingBuffer`）。因为网卡拥有多个发送队列，所以在发送前需要选择一个发送队列。
    
*   将`sk_buffer`添加到发送队列中。
    
*   循环从发送队列（`RingBuffer`）中取出`sk_buffer`，调用内核函数`sch_direct_xmit`发送数据，其中会调用`网卡驱动程序`来发送数据。
    

> 以上过程全部是用户线程的内核态在执行，占用的CPU时间是系统态时间(`sy`)，当分配给用户线程的`CPU quota`用完的时候，会触发`NET_TX_SOFTIRQ`类型的软中断，内核线程`ksoftirqd`会响应这个软中断，并执行`NET_TX_SOFTIRQ`类型的软中断注册的回调函数`net_tx_action`，在回调函数中会执行到驱动程序函数 `dev_hard_start_xmit`来发送数据。

> **注意：当触发`NET_TX_SOFTIRQ`软中断来发送数据时，后边消耗的 CPU 就都显示在 `si`这里了，不会消耗用户进程的系统态时间（`sy`）了。**

> 从这里可以看到网络包的发送过程和接受过程是不同的，在介绍网络包的接受过程时，我们提到是通过触发`NET_RX_SOFTIRQ`类型的软中断在内核线程`ksoftirqd`中执行`内核网络协议栈`接受数据。而在网络数据包的发送过程中是`用户线程的内核态`在执行`内核网络协议栈`，只有当线程的`CPU quota`用尽时，才触发`NET_TX_SOFTIRQ`软中断来发送数据。

> 在整个网络包的发送和接受过程中，`NET_TX_SOFTIRQ`类型的软中断只会在发送网络包时并且当用户线程的`CPU quota`用尽时，才会触发。剩下的接受过程中触发的软中断类型以及发送完数据触发的软中断类型均为`NET_RX_SOFTIRQ`。所以这就是你在服务器上查看 `/proc/softirqs`，一般 `NET_RX`都要比 `NET_TX`大很多的的原因。

*   现在发送流程终于到了网卡真实发送数据的阶段，前边我们讲到无论是用户线程的内核态还是触发`NET_TX_SOFTIRQ`类型的软中断在发送数据的时候最终会调用到网卡的驱动程序函数`dev_hard_start_xmit`来发送数据。在网卡驱动程序函数`dev_hard_start_xmit`中会将`sk_buffer`映射到网卡可访问的`内存 DMA 区域`，最终网卡驱动程序通过`DMA`的方式将`数据帧`通过物理网卡发送出去。
    
*   当数据发送完毕后，还有最后一项重要的工作，就是清理工作。数据发送完毕后，网卡设备会向`CPU`发送一个硬中断，`CPU`调用网卡驱动程序注册的`硬中断响应程序`，在硬中断响应中触发`NET_RX_SOFTIRQ`类型的软中断，在软中断的回调函数`igb_poll`中清理释放 `sk_buffer`，清理`网卡`发送队列（`RingBuffer`），解除 DMA 映射。
    

> 无论`硬中断`是因为`有数据要接收`，还是说`发送完成通知`，从硬中断触发的软中断都是 `NET_RX_SOFTIRQ`。

> 这里释放清理的只是`sk_buffer`的副本，真正的`sk_buffer`现在还是存放在`Socket`的发送队列中。前面在`传输层`处理的时候我们提到过，因为传输层需要`保证可靠性`，所以 `sk_buffer`其实还没有删除。它得等收到对方的 ACK 之后才会真正删除。

性能开销
----

前边我们提到了在网络包接收过程中涉及到的性能开销，现在介绍完了网络包的发送过程，我们来看下在数据包发送过程中的性能开销：

*   和接收数据一样，应用程序在调用`系统调用send`的时候会从`用户态`转为`内核态`以及发送完数据后，`系统调用`返回时从`内核态`转为`用户态`的开销。
    
*   用户线程内核态`CPU quota`用尽时触发`NET_TX_SOFTIRQ`类型软中断，内核响应软中断的开销。
    
*   网卡发送完数据，向`CPU`发送硬中断，`CPU`响应硬中断的开销。以及在硬中断中发送`NET_RX_SOFTIRQ`软中断执行具体的内存清理动作。内核响应软中断的开销。
    
*   内存拷贝的开销。我们来回顾下在数据包发送的过程中都发生了哪些内存拷贝：
    

*   在内核协议栈的传输层中，`TCP协议`对应的发送函数`tcp_sendmsg`会申请`sk_buffer`，将用户要发送的数据`拷贝`到`sk_buffer`中。
    
*   在发送流程从传输层到网络层的时候，会`拷贝`一个`sk_buffer副本`出来，将这个`sk_buffer副本`向下传递。原始`sk_buffer`保留在`Socket`发送队列中，等待网络对端`ACK`，对端`ACK`后删除`Socket`发送队列中的`sk_buffer`。对端没有发送`ACK`，则重新从`Socket`发送队列中发送，实现`TCP协议`的可靠传输。
    
*   在网络层，如果发现要发送的数据大于`MTU`，则会进行分片操作，申请额外的`sk_buffer`，并将原来的sk_buffer`拷贝`到多个小的sk_buffer中。
    

再谈(阻塞，非阻塞)与(同步，异步)
------------------

在我们聊完网络数据的接收和发送过程后，我们来谈下IO中特别容易混淆的概念：`阻塞与同步`，`非阻塞与异步`。

网上各种博文还有各种书籍中有大量的关于这两个概念的解释，但是笔者觉得还是不够形象化，只是对概念的生硬解释，如果硬套概念的话，其实感觉`阻塞与同步`，`非阻塞与异步`还是没啥区别，时间长了，还是比较模糊容易混淆。

所以笔者在这里尝试换一种更加形象化，更加容易理解记忆的方式来清晰地解释下什么是`阻塞与非阻塞`，什么是`同步与异步`。

经过前边对网络数据包接收流程的介绍，在这里我们可以将整个流程总结为两个阶段：

![图片](https://mmbiz.qpic.cn/mmbiz_png/sOIZXFW0vUY4ypDTgZibnVV3K2XJZVLcEgwcdMwZya6aIFeslTuyUGGUtLgBnkQhrP9GhxRPMxNvG3TmKxFG5bQ/640?wx_fmt=png)数据接收阶段.png

*   **数据准备阶段：** 在这个阶段，网络数据包到达网卡，通过`DMA`的方式将数据包拷贝到内存中，然后经过硬中断，软中断，接着通过内核线程`ksoftirqd`经过内核协议栈的处理，最终将数据发送到`内核Socket`的接收缓冲区中。
    
*   **数据拷贝阶段：** 当数据到达`内核Socket`的接收缓冲区中时，此时数据存在于`内核空间`中，需要将数据`拷贝`到`用户空间`中，才能够被应用程序读取。
    

阻塞与非阻塞
------

阻塞与非阻塞的区别主要发生在第一阶段：`数据准备阶段`。

当应用程序发起`系统调用read`时，线程从用户态转为内核态，读取内核`Socket`的接收缓冲区中的网络数据。

### 阻塞

如果这时内核`Socket`的接收缓冲区没有数据，那么线程就会一直`等待`，直到`Socket`接收缓冲区有数据为止。随后将数据从内核空间拷贝到用户空间，`系统调用read`返回。

![图片](https://mmbiz.qpic.cn/mmbiz_png/sOIZXFW0vUY4ypDTgZibnVV3K2XJZVLcE59KUqV0jGMaO1RzueavdIWFn5lpNicVTss7vF5OvdChgR3zjtbGcgzg/640?wx_fmt=png)阻塞IO.png

从图中我们可以看出：**阻塞**的特点是在第一阶段和第二阶段`都会等待`。

### 非阻塞

`阻塞`和`非阻塞`主要的区分是在第一阶段：`数据准备阶段`。

*   在第一阶段，当`Socket`的接收缓冲区中没有数据的时候，`阻塞模式下`应用线程会一直等待。`非阻塞模式下`应用线程不会等待，`系统调用`直接返回错误标志`EWOULDBLOCK`。
    
*   当`Socket`的接收缓冲区中有数据的时候，`阻塞`和`非阻塞`的表现是一样的，都会进入第二阶段`等待`数据从`内核空间`拷贝到`用户空间`，然后`系统调用返回`。
    

![图片](https://mmbiz.qpic.cn/mmbiz_png/sOIZXFW0vUY4ypDTgZibnVV3K2XJZVLcERxuxKJvM0hOVl9fGVF4uP3b5iasjyBg7AQPjwbc9O3w5MjtiaRYM9URQ/640?wx_fmt=png)非阻塞IO.png

从上图中，我们可以看出：**非阻塞**的特点是第一阶段`不会等待`，但是在第二阶段还是会`等待`。

同步与异步
-----

`同步`与`异步`主要的区别发生在第二阶段：`数据拷贝阶段`。

前边我们提到在`数据拷贝阶段`主要是将数据从`内核空间`拷贝到`用户空间`。然后应用程序才可以读取数据。

当内核`Socket`的接收缓冲区有数据到达时，进入第二阶段。

### 同步

`同步模式`在数据准备好后，是由`用户线程`的`内核态`来执行`第二阶段`。所以应用程序会在第二阶段发生`阻塞`，直到数据从`内核空间`拷贝到`用户空间`，系统调用才会返回。

Linux下的 `epoll`和Mac 下的 `kqueue`都属于`同步 IO`。

![图片](https://mmbiz.qpic.cn/mmbiz_png/sOIZXFW0vUY4ypDTgZibnVV3K2XJZVLcEjEQDIAtT2cuNI4aaW1NtREspTEhdh4opJ4bqvamfRm0PAVMC8TjKuQ/640?wx_fmt=png)同步IO.png

### 异步

`异步模式`下是由`内核`来执行第二阶段的数据拷贝操作，当`内核`执行完第二阶段，会通知用户线程IO操作已经完成，并将数据回调给用户线程。所以在`异步模式`下 `数据准备阶段`和`数据拷贝阶段`均是由`内核`来完成，不会对应用程序造成任何阻塞。

基于以上特征，我们可以看到`异步模式`需要内核的支持，比较依赖操作系统底层的支持。

在目前流行的操作系统中，只有Windows 中的 `IOCP`才真正属于异步 IO，实现的也非常成熟。但Windows很少用来作为服务器使用。

而常用来作为服务器使用的Linux，`异步IO机制`实现的不够成熟，与NIO相比性能提升的也不够明显。

但Linux kernel 在5.1版本由Facebook的大神Jens Axboe引入了新的异步IO库`io_uring` 改善了原来Linux native AIO的一些性能问题。性能相比`Epoll`以及之前原生的`AIO`提高了不少，值得关注。

![图片](https://mmbiz.qpic.cn/mmbiz_png/sOIZXFW0vUY4ypDTgZibnVV3K2XJZVLcEgH4TCcSV12UCRFeMUEunGcgFdUG9nZLq2AQ9QjnuQpzpJuKHMTus8w/640?wx_fmt=png)异步IO.png

IO模型
----

在进行网络IO操作时，用什么样的IO模型来读写数据将在很大程度上决定了网络框架的IO性能。所以IO模型的选择是构建一个高性能网络框架的基础。

在《UNIX 网络编程》一书中介绍了五种IO模型：`阻塞IO`,`非阻塞IO`,`IO多路复用`,`信号驱动IO`,`异步IO`，每一种IO模型的出现都是对前一种的升级优化。

下面我们就来分别介绍下这五种IO模型各自都解决了什么问题，适用于哪些场景，各自的优缺点是什么？

阻塞IO（BIO）
---------

![图片](https://mmbiz.qpic.cn/mmbiz_png/sOIZXFW0vUY4ypDTgZibnVV3K2XJZVLcE59KUqV0jGMaO1RzueavdIWFn5lpNicVTss7vF5OvdChgR3zjtbGcgzg/640?wx_fmt=png)阻塞IO.png

经过前一小节对`阻塞`这个概念的介绍，相信大家可以很容易理解`阻塞IO`的概念和过程。

既然这小节我们谈的是`IO`，那么下边我们来看下在`阻塞IO`模型下，网络数据的读写过程。

### 阻塞读

当用户线程发起`read`系统调用，用户线程从用户态切换到内核态，在内核中去查看`Socket`接收缓冲区是否有数据到来。

*   `Socket`接收缓冲区中`有数据`，则用户线程在内核态将内核空间中的数据拷贝到用户空间，系统IO调用返回。
    
*   `Socket`接收缓冲区中`无数据`，则用户线程让出CPU，进入`阻塞状态`。当数据到达`Socket`接收缓冲区后，内核唤醒`阻塞状态`中的用户线程进入`就绪状态`，随后经过CPU的调度获取到`CPU quota`进入`运行状态`，将内核空间的数据拷贝到用户空间，随后系统调用返回。
    

### 阻塞写

当用户线程发起`send`系统调用时，用户线程从用户态切换到内核态，将发送数据从用户空间拷贝到内核空间中的`Socket`发送缓冲区中。

*   当`Socket`发送缓冲区能够容纳下发送数据时，用户线程会将全部的发送数据写入`Socket`缓冲区，然后执行在《网络包发送流程》这小节介绍的后续流程，然后返回。
    
*   当`Socket`发送缓冲区空间不够，无法容纳下全部发送数据时，用户线程让出CPU,进入`阻塞状态`，直到`Socket`发送缓冲区能够容纳下全部发送数据时，内核唤醒用户线程，执行后续发送流程。
    

`阻塞IO`模型下的写操作做事风格比较硬刚，非得要把全部的发送数据写入发送缓冲区才肯善罢甘休。

阻塞IO模型
------

![图片](https://mmbiz.qpic.cn/mmbiz_png/sOIZXFW0vUY4ypDTgZibnVV3K2XJZVLcE15CGWNAUIdAvTbLVsusiaasD4rxcL0k8FShwAj1DguCc5cF8dPzfexA/640?wx_fmt=png)阻塞IO模型.png

由于`阻塞IO`的读写特点，所以导致在`阻塞IO`模型下，每个请求都需要被一个独立的线程处理。一个线程在同一时刻只能与一个连接绑定。来一个请求，服务端就需要创建一个线程用来处理请求。

当客户端请求的并发量突然增大时，服务端在一瞬间就会创建出大量的线程，而创建线程是需要系统资源开销的，这样一来就会一瞬间占用大量的系统资源。

如果客户端创建好连接后，但是一直不发数据，通常大部分情况下，网络连接也`并不`总是有数据可读，那么在空闲的这段时间内，服务端线程就会一直处于`阻塞状态`，无法干其他的事情。CPU也`无法得到充分的发挥`，同时还会`导致大量线程切换的开销`。

适用场景
----

基于以上`阻塞IO模型`的特点，该模型只适用于`连接数少`，`并发度低`的业务场景。

比如公司内部的一些管理系统，通常请求数在100个左右，使用`阻塞IO模型`还是非常适合的。而且性能还不输NIO。

该模型在C10K之前，是普遍被采用的一种IO模型。

非阻塞IO（NIO）
----------

`阻塞IO模型`最大的问题就是一个线程只能处理一个连接，如果这个连接上没有数据的话，那么这个线程就只能阻塞在系统IO调用上，不能干其他的事情。这对系统资源来说，是一种极大的浪费。同时大量的线程上下文切换，也是一个巨大的系统开销。

所以为了解决这个问题，**我们就需要用尽可能少的线程去处理更多的连接。**，`网络IO模型的演变`也是根据这个需求来一步一步演进的。

基于这个需求，第一种解决方案`非阻塞IO`就出现了。我们在上一小节中介绍了`非阻塞`的概念，现在我们来看下网络读写操作在`非阻塞IO`下的特点：

![图片](https://mmbiz.qpic.cn/mmbiz_png/sOIZXFW0vUY4ypDTgZibnVV3K2XJZVLcERxuxKJvM0hOVl9fGVF4uP3b5iasjyBg7AQPjwbc9O3w5MjtiaRYM9URQ/640?wx_fmt=png)非阻塞IO.png

### 非阻塞读

当用户线程发起非阻塞`read`系统调用时，用户线程从`用户态`转为`内核态`，在内核中去查看`Socket`接收缓冲区是否有数据到来。

*   `Socket`接收缓冲区中`无数据`，系统调用立马返回，并带有一个 `EWOULDBLOCK` 或 `EAGAIN`错误，这个阶段用户线程`不会阻塞`，也`不会让出CPU`，而是会继续`轮训`直到`Socket`接收缓冲区中有数据为止。
    
*   `Socket`接收缓冲区中`有数据`，用户线程在`内核态`会将`内核空间`中的数据拷贝到`用户空间`，**注意**这个数据拷贝阶段，应用程序是`阻塞的`，当数据拷贝完成，系统调用返回。
    

### 非阻塞写

前边我们在介绍`阻塞写`的时候提到`阻塞写`的风格特别的硬朗，头比较铁非要把全部发送数据一次性都写到`Socket`的发送缓冲区中才返回，如果发送缓冲区中没有足够的空间容纳，那么就一直阻塞死等，特别的刚。

相比较而言`非阻塞写`的特点就比较佛系，当发送缓冲区中没有足够的空间容纳全部发送数据时，`非阻塞写`的特点是`能写多少写多少`，写不下了，就立即返回。并将写入到发送缓冲区的字节数返回给应用程序，方便用户线程不断的`轮训`尝试将`剩下的数据`写入发送缓冲区中。

非阻塞IO模型
-------

![图片](https://mmbiz.qpic.cn/mmbiz_png/sOIZXFW0vUY4ypDTgZibnVV3K2XJZVLcEZHypSu2HK3FeqU10jRnAwDEkrVZA7kbl0MWpPonZrdOWfXzcngzlvg/640?wx_fmt=png)非阻塞IO模型.png

基于以上`非阻塞IO`的特点，我们就不必像`阻塞IO`那样为每个请求分配一个线程去处理连接上的读写了。

我们可以利用**一个线程或者很少的线程**，去`不断地轮询`每个`Socket`的接收缓冲区是否有数据到达，如果没有数据，`不必阻塞`线程，而是接着去`轮询`下一个`Socket`接收缓冲区，直到轮询到数据后，处理连接上的读写，或者交给业务线程池去处理，轮询线程则`继续轮询`其他的`Socket`接收缓冲区。

这样一个`非阻塞IO模型`就实现了我们在本小节开始提出的需求：**我们需要用尽可能少的线程去处理更多的连接**

适用场景
----

虽然`非阻塞IO模型`与`阻塞IO模型`相比，减少了很大一部分的资源消耗和系统开销。

但是它仍然有很大的性能问题，因为在`非阻塞IO模型`下，需要用户线程去`不断地`发起`系统调用`去轮训`Socket`接收缓冲区，这就需要用户线程不断地从`用户态`切换到`内核态`，`内核态`切换到`用户态`。随着并发量的增大，这个上下文切换的开销也是巨大的。

所以单纯的`非阻塞IO`模型还是无法适用于高并发的场景。只能适用于`C10K`以下的场景。

IO多路复用
------

在`非阻塞IO`这一小节的开头，我们提到`网络IO模型`的演变都是围绕着---**如何用尽可能少的线程去处理更多的连接**这个核心需求开始展开的。

本小节我们来谈谈`IO多路复用模型`，那么什么是`多路`？，什么又是`复用`呢？

我们还是以这个核心需求来对这两个概念展开阐述：

*   **多路**：我们的核心需求是要用尽可能少的线程来处理尽可能多的连接，这里的`多路`指的就是我们需要处理的众多连接。
    
*   **复用**：核心需求要求我们使用`尽可能少的线程`，`尽可能少的系统开销`去处理`尽可能多`的连接（`多路`），那么这里的`复用`指的就是用`有限的资源`，比如用一个线程或者固定数量的线程去处理众多连接上的读写事件。换句话说，在`阻塞IO模型`中一个连接就需要分配一个独立的线程去专门处理这个连接上的读写，到了`IO多路复用模型`中，多个连接可以`复用`这一个独立的线程去处理这多个连接上的读写。
    

好了，`IO多路复用模型`的概念解释清楚了，那么**问题的关键**是我们如何去实现这个`复用`，也就是如何让一个独立的线程去处理众多连接上的读写事件呢？

这个问题其实在`非阻塞IO模型`中已经给出了它的答案，在`非阻塞IO模型`中，利用`非阻塞`的系统IO调用去不断的轮询众多连接的`Socket`接收缓冲区看是否有数据到来，如果有则处理，如果没有则继续轮询下一个`Socket`。这样就达到了用一个线程去处理众多连接上的读写事件了。

**但是**`非阻塞IO模型`最大的问题就是需要不断的发起`系统调用`去轮询各个`Socket`中的接收缓冲区是否有数据到来，`频繁`的`系统调用`随之带来了大量的上下文切换开销。随着并发量的提升，这样也会导致非常严重的性能问题。

**那么如何避免频繁的系统调用同时又可以实现我们的核心需求呢？**

这就需要操作系统的内核来支持这样的操作，我们可以把频繁的轮询操作交给操作系统内核来替我们完成，这样就避免了在`用户空间`频繁的去使用系统调用来轮询所带来的性能开销。

正如我们所想，操作系统内核也确实为我们提供了这样的功能实现，下面我们来一起看下操作系统对`IO多路复用模型`的实现。

select
------

`select`是操作系统内核提供给我们使用的一个`系统调用`，它解决了在`非阻塞IO模型`中需要不断的发起`系统IO调用`去轮询`各个连接上的Socket`接收缓冲区所带来的`用户空间`与`内核空间`不断切换的`系统开销`。

`select`系统调用将`轮询`的操作交给了`内核`来帮助我们完成，从而避免了在`用户空间`不断的发起轮询所带来的的系统性能开销。

![图片](https://mmbiz.qpic.cn/mmbiz_png/sOIZXFW0vUY4ypDTgZibnVV3K2XJZVLcEjOYnCMt84RfZyS5JVtLyrr115wOiauU5b21IvrIsHx4ndVDryGSlfMQ/640?wx_fmt=png)select.png

*   首先用户线程在发起`select`系统调用的时候会`阻塞`在`select`系统调用上。此时，用户线程从`用户态`切换到了`内核态`完成了一次`上下文切换`
    
*   用户线程将需要监听的`Socket`对应的文件描述符`fd`数组通过`select`系统调用传递给内核。此时，用户线程将`用户空间`中的文件描述符`fd`数组`拷贝`到`内核空间`。
    

这里的**文件描述符数组**其实是一个`BitMap`，`BitMap`下标为`文件描述符fd`，下标对应的值为：`1`表示该`fd`上有读写事件，`0`表示该`fd`上没有读写事件。

![图片](https://mmbiz.qpic.cn/mmbiz_png/sOIZXFW0vUY4ypDTgZibnVV3K2XJZVLcED7ATGgNE7Ou6uf2ef80pvzAfiaFI9EakX3FDd1icRiaiaMk5on3c0CrWdA/640?wx_fmt=png)fd数组BitMap.png

**文件描述符fd**其实就是一个`整数值`，在Linux中一切皆文件，`Socket`也是一个文件。描述进程所有信息的数据结构`task_struct`中有一个属性`struct files_struct *files`，它最终指向了一个数组，数组里存放了进程打开的所有文件列表，文件信息封装在`struct file`结构体中，这个数组存放的类型就是`struct file`结构体，`数组的下标`则是我们常说的文件描述符`fd`。

*   当用户线程调用完`select`后开始进入`阻塞状态`，`内核`开始轮询遍历`fd`数组，查看`fd`对应的`Socket`接收缓冲区中是否有数据到来。如果有数据到来，则将`fd`对应`BitMap`的值设置为`1`。如果没有数据到来，则保持值为`0`。
    

> **注意**这里内核会修改原始的`fd`数组！！

*   内核遍历一遍`fd`数组后，如果发现有些`fd`上有IO数据到来，则将修改后的`fd`数组返回给用户线程。此时，会将`fd`数组从`内核空间`拷贝到`用户空间`。
    
*   当内核将修改后的`fd`数组返回给用户线程后，用户线程解除`阻塞`，由用户线程开始遍历`fd`数组然后找出`fd`数组中值为`1`的`Socket`文件描述符。最后对这些`Socket`发起系统调用读取数据。
    

> `select`不会告诉用户线程具体哪些`fd`上有IO数据到来，只是在`IO活跃`的`fd`上打上标记，将打好标记的完整`fd`数组返回给用户线程，所以用户线程还需要遍历`fd`数组找出具体哪些`fd`上有`IO数据`到来。

*   由于内核在遍历的过程中已经修改了`fd`数组，所以在用户线程遍历完`fd`数组后获取到`IO就绪`的`Socket`后，就需要`重置`fd数组，并重新调用`select`传入重置后的`fd`数组，让内核发起新的一轮遍历轮询。
    

### API介绍

当我们熟悉了`select`的原理后，就很容易理解内核给我们提供的`select API`了。

```
 int select(int maxfdp1,fd_set *readset,fd_set *writeset,fd_set *exceptset,const struct timeval *timeout)
```

从`select API`中我们可以看到，`select`系统调用是在规定的`超时时间内`，监听（`轮询`）用户感兴趣的文件描述符集合上的`可读`,`可写`,`异常`三类事件。

*   `maxfdp1 ：` select传递给内核监听的文件描述符集合中数值最大的文件描述符`+1`，目的是用于限定内核遍历范围。比如：`select`监听的文件描述符集合为`{0,1,2,3,4}`，那么`maxfdp1`的值为`5`。
    
*   `fd_set *readset：` 对`可读事件`感兴趣的文件描述符集合。
    
*   `fd_set *writeset：` 对`可写事件`感兴趣的文件描述符集合。
    
*   `fd_set *exceptset：`对`异常事件`感兴趣的文件描述符集合。
    

> 这里的`fd_set`就是我们前边提到的`文件描述符数组`，是一个`BitMap`结构。

*   `const struct timeval *timeout：`select系统调用超时时间，在这段时间内，内核如果没有发现有`IO就绪`的文件描述符，就直接返回。
    

上小节提到，在`内核`遍历完`fd`数组后，发现有`IO就绪`的`fd`，则会将该`fd`对应的`BitMap`中的值设置为`1`，并将修改后的`fd`数组，返回给用户线程。

在用户线程中需要重新遍历`fd`数组，找出`IO就绪`的`fd`出来，然后发起真正的读写调用。

下面介绍下在用户线程中重新遍历`fd`数组的过程中，我们需要用到的`API`：

*   `void FD_ZERO(fd_set *fdset)：`清空指定的文件描述符集合，即让`fd_set`中不在包含任何文件描述符。
    
*   `void FD_SET(int fd, fd_set *fdset)：`将一个给定的文件描述符加入集合之中。
    

> 每次调用`select`之前都要通过`FD_ZERO`和`FD_SET`重新设置文件描述符，因为文件描述符集合会在`内核`中`被修改`。

*   `int FD_ISSET(int fd, fd_set *fdset)：`检查集合中指定的文件描述符是否可以读写。用户线程`遍历`文件描述符集合,调用该方法检查相应的文件描述符是否`IO就绪`。
    
*   `void FD_CLR(int fd, fd_set *fdset)：`将一个给定的文件描述符从集合中删除
    

### 性能开销

虽然`select`解决了`非阻塞IO模型`中频繁发起`系统调用`的问题，但是在整个`select`工作过程中，我们还是看出了`select`有些不足的地方。

*   在发起`select`系统调用以及返回时，用户线程各发生了一次`用户态`到`内核态`以及`内核态`到`用户态`的上下文切换开销。**发生2次上下文`切换`**
    
*   在发起`select`系统调用以及返回时，用户线程在`内核态`需要将`文件描述符集合`从用户空间`拷贝`到内核空间。以及在内核修改完`文件描述符集合`后，又要将它从内核空间`拷贝`到用户空间。**发生2次文件描述符集合的`拷贝`**
    
*   虽然由原来在`用户空间`发起轮询`优化成了`在`内核空间`发起轮询但`select`不会告诉用户线程到底是哪些`Socket`上发生了`IO就绪`事件，只是对`IO就绪`的`Socket`作了标记，用户线程依然要`遍历`文件描述符集合去查找具体`IO就绪`的`Socket`。时间复杂度依然为`O(n)`。
    

> 大部分情况下，网络连接并不总是活跃的，如果`select`监听了大量的客户端连接，只有少数的连接活跃，然而使用轮询的这种方式会随着连接数的增大，效率会越来越低。

*   `内核`会对原始的`文件描述符集合`进行修改。导致每次在用户空间重新发起`select`调用时，都需要对`文件描述符集合`进行`重置`。
    
*   `BitMap`结构的文件描述符集合，长度为固定的`1024`,所以只能监听`0~1023`的文件描述符。
    
*   `select`系统调用 不是线程安全的。
    

以上`select`的不足所产生的`性能开销`都会随着并发量的增大而`线性增长`。

很明显`select`也不能解决`C10K`问题，只适用于`1000`个左右的并发连接场景。

poll
----

`poll`相当于是改进版的`select`，但是工作原理基本和`select`没有本质的区别。

```
int poll(struct pollfd *fds, unsigned int nfds, int timeout)  

```

```
struct pollfd {  
    int   fd;         /* 文件描述符 */  
    short events;     /* 需要监听的事件 */  
    short revents;    /* 实际发生的事件 由内核修改设置 */  
};  

```

`select`中使用的文件描述符集合是采用的固定长度为1024的`BitMap`结构的`fd_set`，而`poll`换成了一个`pollfd`结构没有固定长度的数组，这样就没有了最大描述符数量的限制（当然还会受到系统文件描述符限制）

`poll`只是改进了`select`只能监听`1024`个文件描述符的数量限制，但是并没有在性能方面做出改进。和`select`上本质并没有多大差别。

*   同样需要在`内核空间`和`用户空间`中对文件描述符集合进行`轮询`，查找出`IO就绪`的`Socket`的时间复杂度依然为`O(n)`。
    
*   同样需要将`包含大量文件描述符的集合`整体在`用户空间`和`内核空间`之间`来回复制`，**无论这些文件描述符是否就绪**。他们的开销都会随着文件描述符数量的增加而线性增大。
    
*   `select，poll`在每次新增，删除需要监听的socket时，都需要将整个新的`socket`集合全量传至`内核`。
    

`poll`同样不适用高并发的场景。依然无法解决`C10K`问题。

epoll
-----

通过上边对`select,poll`核心原理的介绍，我们看到`select,poll`的性能瓶颈主要体现在下面三个地方：

*   因为内核不会保存我们要监听的`socket`集合，所以在每次调用`select,poll`的时候都需要传入，传出全量的`socket`文件描述符集合。这导致了大量的文件描述符在`用户空间`和`内核空间`频繁的来回复制。
    
*   由于内核不会通知具体`IO就绪`的`socket`，只是在这些`IO就绪`的socket上打好标记，所以当`select`系统调用返回时，在`用户空间`还是需要`完整遍历`一遍`socket`文件描述符集合来获取具体`IO就绪`的`socket`。
    
*   在`内核空间`中也是通过遍历的方式来得到`IO就绪`的`socket`。
    

下面我们来看下`epoll`是如何解决这些问题的。在介绍`epoll`的核心原理之前，我们需要介绍下理解`epoll`工作过程所需要的一些核心基础知识。

### Socket的创建

服务端线程调用`accept`系统调用后开始`阻塞`，当有客户端连接上来并完成`TCP三次握手`后，`内核`会创建一个对应的`Socket`作为服务端与客户端通信的`内核`接口。

在Linux内核的角度看来，一切皆是文件，`Socket`也不例外，当内核创建出`Socket`之后，会将这个`Socket`放到当前进程所打开的文件列表中管理起来。

下面我们来看下进程管理这些打开的文件列表相关的内核数据结构是什么样的？在了解完这些数据结构后，我们会更加清晰的理解`Socket`在内核中所发挥的作用。并且对后面我们理解`epoll`的创建过程有很大的帮助。

#### 进程中管理文件列表结构

![图片](https://mmbiz.qpic.cn/mmbiz_png/sOIZXFW0vUY4ypDTgZibnVV3K2XJZVLcEDCWo05bLDqtgBwGwAPGHKNGsXCEk7O6LB3ksNKFiaU4qgPouZwQDrFg/640?wx_fmt=png)进程中管理文件列表结构.png

`struct tast_struct`是内核中用来表示进程的一个数据结构，它包含了进程的所有信息。本小节我们只列出和文件管理相关的属性。

其中进程内打开的所有文件是通过一个数组`fd_array`来进行组织管理，数组的下标即为我们常提到的`文件描述符`，数组中存放的是对应的文件数据结构`struct file`。每打开一个文件，内核都会创建一个`struct file`与之对应，并在`fd_array`中找到一个空闲位置分配给它，数组中对应的下标，就是我们在`用户空间`用到的`文件描述符`。

> 对于任何一个进程，默认情况下，文件描述符 `0`表示 `stdin 标准输入`，文件描述符 `1`表示`stdout 标准输出`，文件描述符`2`表示`stderr 标准错误输出`。

进程中打开的文件列表`fd_array`定义在内核数据结构`struct files_struct`中，在`struct fdtable`结构中有一个指针`struct fd **fd`指向`fd_array`。

**由于本小节讨论的是内核网络系统部分的数据结构**，所以这里拿`Socket`文件类型来举例说明：

用于封装文件元信息的内核数据结构`struct file`中的`private_data`指针指向具体的`Socket`结构。

`struct file`中的`file_operations`属性定义了文件的操作函数，不同的文件类型，对应的`file_operations`是不同的，针对`Socket`文件类型，这里的`file_operations`指向`socket_file_ops`。

> 我们在`用户空间`对`Socket`发起的读写等系统调用，进入内核首先会调用的是`Socket`对应的`struct file`中指向的`socket_file_ops`。**比如**：对`Socket`发起`write`写操作，在内核中首先被调用的就是`socket_file_ops`中定义的`sock_write_iter`。`Socket`发起`read`读操作内核中对应的则是`sock_read_iter`。

```
  
static const struct file_operations socket_file_ops = {  
  .owner =  THIS_MODULE,  
  .llseek =  no_llseek,  
  .read_iter =  sock_read_iter,  
  .write_iter =  sock_write_iter,  
  .poll =    sock_poll,  
  .unlocked_ioctl = sock_ioctl,  
  .mmap =    sock_mmap,  
  .release =  sock_close,  
  .fasync =  sock_fasync,  
  .sendpage =  sock_sendpage,  
  .splice_write = generic_splice_sendpage,  
  .splice_read =  sock_splice_read,  
};  

```

#### Socket内核结构

![图片](https://mmbiz.qpic.cn/mmbiz_png/sOIZXFW0vUY4ypDTgZibnVV3K2XJZVLcEV2mJ5ffNML0pVxP679iapicwtrNvRLjAuPJ9EOS0J2VFDtuMicX3Lw3cQ/640?wx_fmt=png)Socket内核结构.png

在我们进行网络程序的编写时会首先创建一个`Socket`，然后基于这个`Socket`进行`bind`，`listen`，我们先将这个`Socket`称作为`监听Socket`。

1.  当我们调用`accept`后，内核会基于`监听Socket`创建出来一个新的`Socket`专门用于与客户端之间的网络通信。并将`监听Socket`中的`Socket操作函数集合`（`inet_stream_ops`）`ops`赋值到新的`Socket`的`ops`属性中。
    

```
const struct proto_ops inet_stream_ops = {  
  .bind = inet_bind,  
  .connect = inet_stream_connect,  
  .accept = inet_accept,  
  .poll = tcp_poll,  
  .listen = inet_listen,  
  .sendmsg = inet_sendmsg,  
  .recvmsg = inet_recvmsg,  
  ......  
}  

```

> 这里需要注意的是，`监听的 socket`和真正用来网络通信的 `Socket`，是两个 Socket，一个叫作`监听 Socket`，一个叫作`已连接的Socket`。

2.  接着内核会为`已连接的Socket`创建`struct file`并初始化，并把Socket文件操作函数集合（`socket_file_ops`）赋值给`struct file`中的`f_ops`指针。然后将`struct socket`中的`file`指针指向这个新分配申请的`struct file`结构体。
    

> 内核会维护两个队列：
> 
> *   一个是已经完成`TCP三次握手`，连接状态处于`established`的连接队列。内核中为`icsk_accept_queue`。
>     
> *   一个是还没有完成`TCP三次握手`，连接状态处于`syn_rcvd`的半连接队列。
>     

3.  然后调用`socket->ops->accept`，从`Socket内核结构图`中我们可以看到其实调用的是`inet_accept`，该函数会在`icsk_accept_queue`中查找是否有已经建立好的连接，如果有的话，直接从`icsk_accept_queue`中获取已经创建好的`struct sock`。并将这个`struct sock`对象赋值给`struct socket`中的`sock`指针。
    

`struct sock`在`struct socket`中是一个非常核心的内核对象，正是在这里定义了我们在介绍`网络包的接收发送流程`中提到的`接收队列`，`发送队列`，`等待队列`，`数据就绪回调函数指针`，`内核协议栈操作函数集合`

*   根据创建`Socket`时发起的系统调用`sock_create`中的`protocol`参数(对于`TCP协议`这里的参数值为`SOCK_STREAM`)查找到对于 tcp 定义的操作方法实现集合 `inet_stream_ops` 和`tcp_prot`。并把它们分别设置到`socket->ops`和`sock->sk_prot`上。
    

> 这里可以回看下本小节开头的《Socket内核结构图》捋一下他们之间的关系。

> `socket`相关的操作接口定义在`inet_stream_ops`函数集合中，负责对上给用户提供接口。而`socket`与内核协议栈之间的操作接口定义在`struct sock`中的`sk_prot`指针上，这里指向`tcp_prot`协议操作函数集合。

```
struct proto tcp_prot = {  
  .name      = "TCP",  
  .owner      = THIS_MODULE,  
  .close      = tcp_close,  
  .connect    = tcp_v4_connect,  
  .disconnect    = tcp_disconnect,  
  .accept      = inet_csk_accept,  
  .keepalive    = tcp_set_keepalive,  
  .recvmsg    = tcp_recvmsg,  
  .sendmsg    = tcp_sendmsg,  
  .backlog_rcv    = tcp_v4_do_rcv,  
   ......  
}  

```

> 之前提到的对`Socket`发起的系统IO调用，在内核中首先会调用`Socket`的文件结构`struct file`中的`file_operations`文件操作集合，然后调用`struct socket`中的`ops`指向的`inet_stream_ops`socket操作函数，最终调用到`struct sock`中`sk_prot`指针指向的`tcp_prot`内核协议栈操作函数接口集合。

![图片](https://mmbiz.qpic.cn/mmbiz_png/sOIZXFW0vUY4ypDTgZibnVV3K2XJZVLcEulX0vLf43hBicEqX8FejWO7Z5ru4ZyWKVmQ5gRSoLruicozQOwotH6fQ/640?wx_fmt=png)系统IO调用结构.png

*   将`struct sock` 对象中的`sk_data_ready` 函数指针设置为 `sock_def_readable`，在`Socket`数据就绪的时候内核会回调该函数。
    
*   `struct sock`中的`等待队列`中存放的是系统IO调用发生阻塞的`进程fd`，以及相应的`回调函数`。**记住这个地方，后边介绍epoll的时候我们还会提到！**
    

4.  当`struct file`，`struct socket`，`struct sock`这些核心的内核对象创建好之后，最后就是把`socket`对象对应的`struct file`放到进程打开的文件列表`fd_array`中。随后系统调用`accept`返回`socket`的文件描述符`fd`给用户程序。
    

### 阻塞IO中用户进程阻塞以及唤醒原理

在前边小节我们介绍`阻塞IO`的时候提到，当用户进程发起系统IO调用时，这里我们拿`read`举例，用户进程会在`内核态`查看对应`Socket`接收缓冲区是否有数据到来。

*   `Socket`接收缓冲区有数据，则拷贝数据到`用户空间`，系统调用返回。
    
*   `Socket`接收缓冲区没有数据，则用户进程让出`CPU`进入`阻塞状态`，当数据到达接收缓冲区时，用户进程会被唤醒，从`阻塞状态`进入`就绪状态`，等待CPU调度。
    

本小节我们就来看下用户进程是如何`阻塞`在`Socket`上，又是如何在`Socket`上被唤醒的。**理解这个过程很重要，对我们理解epoll的事件通知过程很有帮助**

*   首先我们在用户进程中对`Socket`进行`read`系统调用时，用户进程会从`用户态`转为`内核态`。
    
*   在进程的`struct task_struct`结构找到`fd_array`，并根据`Socket`的文件描述符`fd`找到对应的`struct file`，调用`struct file`中的文件操作函数结合`file_operations`，`read`系统调用对应的是`sock_read_iter`。
    
*   在`sock_read_iter`函数中找到`struct file`指向的`struct socket`，并调用`socket->ops->recvmsg`，这里我们知道调用的是`inet_stream_ops`集合中定义的`inet_recvmsg`。
    
*   在`inet_recvmsg`中会找到`struct sock`，并调用`sock->skprot->recvmsg`,这里调用的是`tcp_prot`集合中定义的`tcp_recvmsg`函数。
    

> 整个调用过程可以参考上边的《系统IO调用结构图》

**熟悉了内核函数调用栈后，我们来看下系统IO调用在`tcp_recvmsg`内核函数中是如何将用户进程给阻塞掉的**

![图片](https://mmbiz.qpic.cn/mmbiz_png/sOIZXFW0vUY4ypDTgZibnVV3K2XJZVLcEmK3jYMPOt7rib3Q5g7XFoaibh594gLT4uFERshJOcs9TGdVajaRaVYkA/640?wx_fmt=png)系统IO调用阻塞原理.png

```
int tcp_recvmsg(struct kiocb *iocb, struct sock *sk, struct msghdr *msg,  
  size_t len, int nonblock, int flags, int *addr_len)  
{  
    .................省略非核心代码...............  
   //访问sock对象中定义的接收队列  
  skb_queue_walk(&sk->sk_receive_queue, skb) {  
  
    .................省略非核心代码...............  
  
  //没有收到足够数据，调用sk_wait_data 阻塞当前进程  
  sk_wait_data(sk, &timeo);  
}  

```

```
int sk_wait_data(struct sock *sk, long *timeo)  
{  
 //创建struct sock中等待队列上的元素wait_queue_t  
 //将进程描述符和回调函数autoremove_wake_function关联到wait_queue_t中  
 DEFINE_WAIT(wait);  
  
 // 调用 sk_sleep 获取 sock 对象下的等待队列的头指针wait_queue_head_t  
 // 调用prepare_to_wait将新创建的等待项wait_queue_t插入到等待队列中，并将进程状态设置为可打断 INTERRUPTIBLE  
 prepare_to_wait(sk_sleep(sk), &wait, TASK_INTERRUPTIBLE);  
 set_bit(SOCK_ASYNC_WAITDATA, &sk->sk_socket->flags);  
  
 // 通过调用schedule_timeout让出CPU，然后进行睡眠，导致一次上下文切换  
 rc = sk_wait_event(sk, timeo, !skb_queue_empty(&sk->sk_receive_queue));  
 ...  

```

*   首先会在`DEFINE_WAIT`中创建`struct sock`中等待队列上的等待类型`wait_queue_t`。
    

```
#define DEFINE_WAIT(name) DEFINE_WAIT_FUNC(name, autoremove_wake_function)  
  
#define DEFINE_WAIT_FUNC(name, function)    \  
 wait_queue_t name = {      \  
  .private = current,    \  
  .func  = function,    \  
  .task_list = LIST_HEAD_INIT((name).task_list), \  
 }  

```

等待类型`wait_queue_t`中的`private`用来关联`阻塞`在当前`socket`上的用户进程`fd`。`func`用来关联等待项上注册的回调函数。这里注册的是`autoremove_wake_function`。

*   调用`sk_sleep(sk)`获取`struct sock`对象中的等待队列头指针`wait_queue_head_t`。
    
*   调用`prepare_to_wait`将新创建的等待项`wait_queue_t`插入到等待队列中，并将进程设置为可打断 `INTERRUPTIBL`。
    
*   调用`sk_wait_event`让出CPU，进程进入睡眠状态。
    

用户进程的`阻塞过程`我们就介绍完了，关键是要理解记住`struct sock`中定义的等待队列上的等待类型`wait_queue_t`的结构。后面`epoll`的介绍中我们还会用到它。

**下面我们接着介绍当数据就绪后，用户进程是如何被唤醒的**

在本文开始介绍《网络包接收过程》这一小节中我们提到：

*   当网络数据包到达网卡时，网卡通过`DMA`的方式将数据放到`RingBuffer`中。
    
*   然后向CPU发起硬中断，在硬中断响应程序中创建`sk_buffer`，并将网络数据拷贝至`sk_buffer`中。
    
*   随后发起软中断，内核线程`ksoftirqd`响应软中断，调用`poll函数`将`sk_buffer`送往内核协议栈做层层协议处理。
    
*   在传输层`tcp_rcv 函数`中，去掉TCP头，根据`四元组（源IP，源端口，目的IP，目的端口）`查找对应的`Socket`。
    
*   最后将`sk_buffer`放到`Socket`中的接收队列里。
    

上边这些过程是内核接收网络数据的完整过程，下边我们来看下，当数据包接收完毕后，用户进程是如何被唤醒的。

![图片](https://mmbiz.qpic.cn/mmbiz_png/sOIZXFW0vUY4ypDTgZibnVV3K2XJZVLcE9kTceMWn7KUGpgqK5Sq7nMgTBs4UkZwJ5Mor0WEibKZvLBxLeV0dJJg/640?wx_fmt=png)系统IO调用唤醒原理.png

*   当软中断将`sk_buffer`放到`Socket`的接收队列上时，接着就会调用`数据就绪函数回调指针sk_data_ready`，前边我们提到，这个函数指针在初始化的时候指向了`sock_def_readable`函数。
    
*   在`sock_def_readable`函数中会去获取`socket->sock->sk_wq`等待队列。在`wake_up_common`函数中从等待队列`sk_wq`中找出`一个`等待项`wait_queue_t`，回调注册在该等待项上的`func`回调函数（`wait_queue_t->func`）,创建等待项`wait_queue_t`是我们提到，这里注册的回调函数是`autoremove_wake_function`。
    

> 即使是有多个进程都阻塞在同一个 socket 上，也只唤醒 1 个进程。其作用是为了避免惊群。

*   在`autoremove_wake_function`函数中，根据等待项`wait_queue_t`上的`private`关联的`阻塞进程fd`调用`try_to_wake_up`唤醒阻塞在该`Socket`上的进程。
    

> 记住`wait_queue_t`中的`func`函数指针，在`epoll`中这里会注册`epoll`的回调函数。

现在理解`epoll`所需要的基础知识我们就介绍完了，唠叨了这么多，下面终于正式进入本小节的主题`epoll`了。

### epoll_create创建epoll对象

`epoll_create`是内核提供给我们创建`epoll`对象的一个系统调用，当我们在用户进程中调用`epoll_create`时，内核会为我们创建一个`struct eventpoll`对象，并且也有相应的`struct file`与之关联，同样需要把这个`struct eventpoll`对象所关联的`struct file`放入进程打开的文件列表`fd_array`中管理。

> 熟悉了`Socket`的创建逻辑，`epoll`的创建逻辑也就不难理解了。

> `struct eventpoll`对象关联的`struct file`中的`file_operations 指针`指向的是`eventpoll_fops`操作函数集合。

```
static const struct file_operations eventpoll_fops = {  
     .release = ep_eventpoll_release;  
     .poll = ep_eventpoll_poll,  
}  

```

![图片](https://mmbiz.qpic.cn/mmbiz_png/sOIZXFW0vUY4ypDTgZibnVV3K2XJZVLcEibKwPYGuZaNbHCJ6jeia6TBicsgdh0cicLaIaHaGgLomLzb9h2r3Wfd3Cw/640?wx_fmt=png)eopll在进程中的整体结构.png

```
struct eventpoll {  
  
    //等待队列，阻塞在epoll上的进程会放在这里  
    wait_queue_head_t wq;  
  
    //就绪队列，IO就绪的socket连接会放在这里  
    struct list_head rdllist;  
  
    //红黑树用来管理所有监听的socket连接  
    struct rb_root rbr;  
  
    ......  
}  

```

*   `wait_queue_head_t wq：`epoll中的等待队列，队列里存放的是`阻塞`在`epoll`上的用户进程。在`IO就绪`的时候`epoll`可以通过这个队列找到这些`阻塞`的进程并唤醒它们，从而执行`IO调用`读写`Socket`上的数据。
    

> 这里注意与`Socket`中的等待队列区分！！！

*   `struct list_head rdllist：`epoll中的就绪队列，队列里存放的是都是`IO就绪`的`Socket`，被唤醒的用户进程可以直接读取这个队列获取`IO活跃`的`Socket`。无需再次遍历整个`Socket`集合。
    

> 这里正是`epoll`比`select ，poll`高效之处，`select ，poll`返回的是全部的`socket`连接，我们需要在`用户空间`再次遍历找出真正`IO活跃`的`Socket`连接。而`epoll`只是返回`IO活跃`的`Socket`连接。用户进程可以直接进行IO操作。

*   `struct rb_root rbr :` 由于红黑树在`查找`，`插入`，`删除`等综合性能方面是最优的，所以epoll内部使用一颗红黑树来管理海量的`Socket`连接。
    

> `select`用`数组`管理连接，`poll`用`链表`管理连接。

### epoll_ctl向epoll对象中添加监听的Socket

当我们调用`epoll_create`在内核中创建出`epoll`对象`struct eventpoll`后，我们就可以利用`epoll_ctl`向`epoll`中添加我们需要管理的`Socket`连接了。

1.  首先要在epoll内核中创建一个表示`Socket连接`的数据结构`struct epitem`，而在`epoll`中为了综合性能的考虑，采用一颗红黑树来管理这些海量`socket连接`。所以`struct epitem`是一个红黑树节点。
    

![图片](https://mmbiz.qpic.cn/mmbiz_png/sOIZXFW0vUY4ypDTgZibnVV3K2XJZVLcEItBeLMXJPAJtcYN6yeNKBbIkH8pCgeic4CIpFTJqrBJ4IkAxJH6JaEg/640?wx_fmt=png)struct epitem.png

```
struct epitem  
{  
      //指向所属epoll对象  
      struct eventpoll *ep;   
      //注册的感兴趣的事件,也就是用户空间的epoll_event       
      struct epoll_event event;   
      //指向epoll对象中的就绪队列  
      struct list_head rdllink;    
      //指向epoll中对应的红黑树节点  
      struct rb_node rbn;       
      //指向epitem所表示的socket->file结构以及对应的fd  
      struct epoll_filefd ffd;                    
  }  

```

> 这里重点记住`struct epitem`结构中的`rdllink`以及`epoll_filefd`成员，后面我们会用到。

2.  在内核中创建完表示`Socket连接`的数据结构`struct epitem`后，我们就需要在`Socket`中的等待队列上创建等待项`wait_queue_t`并且注册`epoll的回调函数ep_poll_callback`。
    

通过`《阻塞IO中用户进程阻塞以及唤醒原理》`小节的铺垫，我想大家已经猜到这一步的意义所在了吧！当时在等待项`wait_queue_t`中注册的是`autoremove_wake_function`回调函数。还记得吗？

> epoll的回调函数`ep_poll_callback`正是`epoll`同步IO事件通知机制的核心所在，也是区别于`select，poll`采用内核轮询方式的根本性能差异所在。

![图片](https://mmbiz.qpic.cn/mmbiz_png/sOIZXFW0vUY4ypDTgZibnVV3K2XJZVLcEkerTL14uiakMoYWeslKZrAXfMibzAwSt2CE6LmTbHMI6yK1oolEvCqAQ/640?wx_fmt=png)epitem创建等待项.png

**这里又出现了一个新的数据结构`struct eppoll_entry`，那它的作用是干什么的呢？大家可以结合上图先猜测下它的作用!**

我们知道`socket->sock->sk_wq`等待队列中的类型是`wait_queue_t`，我们需要在`struct epitem`所表示的`socket`的等待队列上注册`epoll`回调函数`ep_poll_callback`。

这样当数据到达`socket`中的接收队列时，内核会回调`sk_data_ready`，在`阻塞IO中用户进程阻塞以及唤醒原理`这一小节中，我们知道这个`sk_data_ready`函数指针会指向`sk_def_readable`函数，在`sk_def_readable`中会回调注册在等待队列里的等待项`wait_queue_t -> func`回调函数`ep_poll_callback`。**在`ep_poll_callback`中需要找到`epitem`**，将`IO就绪`的`epitem`放入`epoll`中的就绪队列中。

而`socket`等待队列中类型是`wait_queue_t`无法关联到`epitem`。所以就出现了`struct eppoll_entry`结构体，它的作用就是关联`Socket`等待队列中的等待项`wait_queue_t`和`epitem`。

```
struct eppoll_entry {   
   //指向关联的epitem  
   struct epitem *base;   
  
  // 关联监听socket中等待队列中的等待项 (private = null  func = ep_poll_callback)  
   wait_queue_t wait;     
  
   // 监听socket中等待队列头指针  
   wait_queue_head_t *whead;   
    .........  
  }; 
```

这样在`ep_poll_callback`回调函数中就可以根据`Socket`等待队列中的等待项`wait`，通过`container_of宏`找到`eppoll_entry`，继而找到`epitem`了。

> `container_of`在Linux内核中是一个常用的宏，用于从包含在某个结构中的指针获得结构本身的指针，通俗地讲就是通过结构体变量中某个成员的首地址进而获得整个结构体变量的首地址。

> 这里需要注意下这次等待项`wait_queue_t`中的`private`设置的是`null`，因为这里`Socket`是交给`epoll`来管理的，阻塞在`Socket`上的进程是也由`epoll`来唤醒。在等待项`wait_queue_t`注册的`func`是`ep_poll_callback`而不是`autoremove_wake_function`，`阻塞进程`并不需要`autoremove_wake_function`来唤醒，所以这里设置`private`为`null`

3.  当在`Socket`的等待队列中创建好等待项`wait_queue_t`并且注册了`epoll`的回调函数`ep_poll_callback`，然后又通过`eppoll_entry`关联了`epitem`后。剩下要做的就是将`epitem`插入到`epoll`中的红黑树`struct rb_root rbr`中。
    

> 这里可以看到`epoll`另一个优化的地方，`epoll`将所有的`socket`连接通过内核中的红黑树来集中管理。每次添加或者删除`socket连接`都是增量添加删除，而不是像`select，poll`那样每次调用都是全量`socket连接`集合传入内核。避免了`频繁大量`的`内存拷贝`。

### epoll_wait同步阻塞获取IO就绪的Socket

1.  用户程序调用`epoll_wait`后，内核首先会查找epoll中的就绪队列`eventpoll->rdllist`是否有`IO就绪`的`epitem`。`epitem`里封装了`socket`的信息。如果就绪队列中有就绪的`epitem`，就将`就绪的socket`信息封装到`epoll_event`返回。
    
2.  如果`eventpoll->rdllist`就绪队列中没有`IO就绪`的`epitem`，则会创建等待项`wait_queue_t`，将用户进程的`fd`关联到`wait_queue_t->private`上，并在等待项`wait_queue_t->func`上注册回调函数`default_wake_function`。最后将等待项添加到`epoll`中的等待队列中。用户进程让出CPU，进入`阻塞状态`。
    

![图片](https://mmbiz.qpic.cn/mmbiz_png/sOIZXFW0vUY4ypDTgZibnVV3K2XJZVLcEG5wWc27ZGXv7sBleibFeuPfEuyfibG9VolWJCiaHlp0DZoYoibpwb3MgNA/640?wx_fmt=png)epoll_wait同步获取数据.png

> 这里和`阻塞IO模型`中的阻塞原理是一样的，只不过在`阻塞IO模型`中注册到等待项`wait_queue_t->func`上的是`autoremove_wake_function`，并将等待项添加到`socket`中的等待队列中。这里注册的是`default_wake_function`，将等待项添加到`epoll`中的等待队列上。

![图片](https://mmbiz.qpic.cn/mmbiz_png/sOIZXFW0vUY4ypDTgZibnVV3K2XJZVLcEl2MECv0ibicDsbibN5ZJaql9ziaJMEFNtzEoxr5LcJuR9vaojRYZ5kyOfA/640?wx_fmt=png)数据到来epoll_wait流程.png

3.  **前边做了那么多的知识铺垫，下面终于到了`epoll`的整个工作流程了：**
    

![图片](https://mmbiz.qpic.cn/mmbiz_png/sOIZXFW0vUY4ypDTgZibnVV3K2XJZVLcEgGR4zpAZARg0IVDCjFjwuukcBZsicw9tmJeO2CCda9x4EwMRZibXptWg/640?wx_fmt=png)epoll_wait处理过程.png

*   当网络数据包在软中断中经过内核协议栈的处理到达`socket`的接收缓冲区时，紧接着会调用socket的数据就绪回调指针`sk_data_ready`，回调函数为`sock_def_readable`。在`socket`的等待队列中找出等待项，其中等待项中注册的回调函数为`ep_poll_callback`。
    
*   在回调函数`ep_poll_callback`中，根据`struct eppoll_entry`中的`struct wait_queue_t wait`通过`container_of宏`找到`eppoll_entry`对象并通过它的`base`指针找到封装`socket`的数据结构`struct epitem`，并将它加入到`epoll`中的就绪队列`rdllist`中。
    
*   随后查看`epoll`中的等待队列中是否有等待项，也就是说查看是否有进程阻塞在`epoll_wait`上等待`IO就绪`的`socket`。如果没有等待项，则软中断处理完成。
    
*   如果有等待项，则回到注册在等待项中的回调函数`default_wake_function`,在回调函数中唤醒`阻塞进程`，并将就绪队列`rdllist`中的`epitem`的`IO就绪`socket信息封装到`struct epoll_event`中返回。
    
*   用户进程拿到`epoll_event`获取`IO就绪`的socket，发起系统IO调用读取数据。
    

再谈水平触发和边缘触发
-----------

网上有大量的关于这两种模式的讲解，大部分讲的比较模糊，感觉只是强行从概念上进行描述，看完让人难以理解。所以在这里，笔者想结合上边`epoll`的工作过程，再次对这两种模式做下自己的解读，力求清晰的解释出这两种工作模式的异同。

经过上边对`epoll`工作过程的详细解读，我们知道，当我们监听的`socket`上有数据到来时，软中断会执行`epoll`的回调函数`ep_poll_callback`,在回调函数中会将`epoll`中描述`socket信息`的数据结构`epitem`插入到`epoll`中的就绪队列`rdllist`中。随后用户进程从`epoll`的等待队列中被唤醒，`epoll_wait`将`IO就绪`的`socket`返回给用户进程，随即`epoll_wait`会清空`rdllist`。

**水平触发**和**边缘触发**最关键的**区别**就在于当`socket`中的接收缓冲区还有数据可读时。**`epoll_wait`是否会清空`rdllist`。**

*   **水平触发**：在这种模式下，用户线程调用`epoll_wait`获取到`IO就绪`的socket后，对`Socket`进行系统IO调用读取数据，假设`socket`中的数据只读了一部分没有全部读完，这时再次调用`epoll_wait`，`epoll_wait`会检查这些`Socket`中的接收缓冲区是否还有数据可读，如果还有数据可读，就将`socket`重新放回`rdllist`。所以当`socket`上的IO没有被处理完时，再次调用`epoll_wait`依然可以获得这些`socket`，用户进程可以接着处理`socket`上的IO事件。
    
*   **边缘触发：** 在这种模式下，`epoll_wait`就会直接清空`rdllist`，不管`socket`上是否还有数据可读。所以在边缘触发模式下，当你没有来得及处理`socket`接收缓冲区的剩下可读数据时，再次调用`epoll_wait`，因为这时`rdlist`已经被清空了，`socket`不会再次从`epoll_wait`中返回，所以用户进程就不会再次获得这个`socket`了，也就无法在对它进行IO处理了。**除非，这个`socket`上有新的IO数据到达**，根据`epoll`的工作过程，该`socket`会被再次放入`rdllist`中。
    

> 如果你在`边缘触发模式`下，处理了部分`socket`上的数据，那么想要处理剩下部分的数据，就只能等到这个`socket`上再次有网络数据到达。

在`Netty`中实现的`EpollSocketChannel`默认的就是`边缘触发`模式。`JDK`的`NIO`默认是`水平触发`模式。

### epoll对select，poll的优化总结

*   `epoll`在内核中通过`红黑树`管理海量的连接，所以在调用`epoll_wait`获取`IO就绪`的socket时，不需要传入监听的socket文件描述符。从而避免了海量的文件描述符集合在`用户空间`和`内核空间`中来回复制。
    

> `select，poll`每次调用时都需要传递全量的文件描述符集合，导致大量频繁的拷贝操作。

*   `epoll`仅会通知`IO就绪`的socket。避免了在用户空间遍历的开销。
    

> `select，poll`只会在`IO就绪`的socket上打好标记，依然是全量返回，所以在用户空间还需要用户程序在一次遍历全量集合找出具体`IO就绪`的socket。

*   `epoll`通过在`socket`的等待队列上注册回调函数`ep_poll_callback`通知用户程序`IO就绪`的socket。避免了在内核中轮询的开销。
    

> 大部分情况下`socket`上并不总是`IO活跃`的，在面对海量连接的情况下，`select，poll`采用内核轮询的方式获取`IO活跃`的socket，无疑是性能低下的核心原因。

根据以上`epoll`的性能优势，它是目前为止各大主流网络框架，以及反向代理中间件使用到的网络IO模型。

利用`epoll`多路复用IO模型可以轻松的解决`C10K`问题。

`C100k`的解决方案也还是基于`C10K`的方案，通过`epoll` 配合线程池，再加上 CPU、内存和网络接口的性能和容量提升。大部分情况下，`C100K`很自然就可以达到。

甚至`C1000K`的解决方法，本质上还是构建在 `epoll` 的`多路复用 I/O 模型`上。只不过，除了 I/O 模型之外，还需要从应用程序到 Linux 内核、再到 CPU、内存和网络等各个层次的深度优化，特别是需要借助硬件，来卸载那些原来通过软件处理的大量功能（`去掉大量的中断响应开销`，`以及内核协议栈处理的开销`）。

信号驱动IO
------

![图片](https://mmbiz.qpic.cn/mmbiz_png/sOIZXFW0vUY4ypDTgZibnVV3K2XJZVLcEPmrUxCAPYRsmPqyGfq7icCaV1FP8bwKOTt0icvGO12ickvLv8Y80K7R5A/640?wx_fmt=png)信号驱动IO.png

大家对这个装备肯定不会陌生，当我们去一些美食城吃饭的时候，点完餐付了钱，老板会给我们一个信号器。然后我们带着这个信号器可以去找餐桌，或者干些其他的事情。当信号器亮了的时候，这时代表饭餐已经做好，我们可以去窗口取餐了。

这个典型的生活场景和我们要介绍的`信号驱动IO模型`就很像。

在`信号驱动IO模型`下，用户进程操作通过`系统调用 sigaction 函数`发起一个 IO 请求，在对应的`socket`注册一个`信号回调`，此时`不阻塞`用户进程，进程会继续工作。当内核数据就绪时，内核就为该进程生成一个 `SIGIO 信号`，通过信号回调通知进程进行相关 IO 操作。

> 这里需要注意的是：`信号驱动式 IO 模型`依然是`同步IO`，因为它虽然可以在等待数据的时候不被阻塞，也不会频繁的轮询，但是当数据就绪，内核信号通知后，用户进程依然要自己去读取数据，在`数据拷贝阶段`发生阻塞。

> 信号驱动 IO模型 相比于前三种 IO 模型，实现了在等待数据就绪时，进程不被阻塞，主循环可以继续工作，所以`理论上`性能更佳。

但是实际上，使用`TCP协议`通信时，`信号驱动IO模型`几乎`不会被采用`。原因如下：

*   信号IO 在大量 IO 操作时可能会因为信号队列溢出导致没法通知
    
*   `SIGIO 信号`是一种 Unix 信号，信号没有附加信息，如果一个信号源有多种产生信号的原因，信号接收者就无法确定究竟发生了什么。而 TCP socket 生产的信号事件有七种之多，这样应用程序收到 SIGIO，根本无从区分处理。
    

但`信号驱动IO模型`可以用在 `UDP`通信上，因为UDP 只有`一个数据请求事件`，这也就意味着在正常情况下 UDP 进程只要捕获 SIGIO 信号，就调用 `read 系统调用`读取到达的数据。如果出现异常，就返回一个异常错误。

* * *

这里插句题外话，大家觉不觉得`阻塞IO模型`在生活中的例子就像是我们在食堂排队打饭。你自己需要排队去打饭同时打饭师傅在配菜的过程中你需要等待。

![图片](https://mmbiz.qpic.cn/mmbiz_png/sOIZXFW0vUY4ypDTgZibnVV3K2XJZVLcEfnPgIKViaoGPghC3DGrmSn8YiaibhwQBrUiaFgticngwAwyqr6hh6exRocA/640?wx_fmt=png)阻塞IO.png

`IO多路复用模型`就像是我们在饭店门口排队等待叫号。叫号器就好比`select,poll,epoll`可以统一管理全部顾客的`吃饭就绪`事件，客户好比是`socket`连接，谁可以去吃饭了，叫号器就通知谁。

![图片](https://mmbiz.qpic.cn/mmbiz_png/sOIZXFW0vUY4ypDTgZibnVV3K2XJZVLcEB28gaSs3gtCLlPLIlibib0kRf9XgKCEbuW8Tvx3dC20Bc8MylahYMbtg/640?wx_fmt=png)IO多路复用.png

##异步IO（AIO）

以上介绍的四种`IO模型`均为`同步IO`，它们都会阻塞在第二阶段`数据拷贝阶段`。

通过在前边小节《同步与异步》中的介绍，相信大家很容易就会理解`异步IO模型`，在`异步IO模型`下，IO操作在`数据准备阶段`和`数据拷贝阶段`均是由内核来完成，不会对应用程序造成任何阻塞。应用进程只需要在`指定的数组`中引用数据即可。

`异步 IO` 与`信号驱动 IO` 的主要区别在于：`信号驱动 IO` 由内核通知何时可以`开始一个 IO 操作`，而`异步 IO`由内核通知 `IO 操作何时已经完成`。

举个生活中的例子：`异步IO模型`就像我们去一个高档饭店里的包间吃饭，我们只需要坐在包间里面，点完餐（`类比异步IO调用`）之后，我们就什么也不需要管，该喝酒喝酒，该聊天聊天，饭餐做好后服务员（`类比内核`）会自己给我们送到包间（`类比用户空间`）来。整个过程没有任何阻塞。

![图片](https://mmbiz.qpic.cn/mmbiz_png/sOIZXFW0vUY4ypDTgZibnVV3K2XJZVLcEfQXKapnOwAK72jTpaKrqfUicX38A8Z5TWopFUyZC0B04NMBRMdiapAFw/640?wx_fmt=png)异步IO.png

`异步IO`的系统调用需要操作系统内核来支持，目前只有`Window`中的`IOCP`实现了非常成熟的`异步IO机制`。

而`Linux`系统对`异步IO机制`实现的不够成熟，且与`NIO`的性能相比提升也不明显。

> 但Linux kernel 在5.1版本由Facebook的大神Jens Axboe引入了新的异步IO库`io_uring` 改善了原来Linux native AIO的一些性能问题。性能相比`Epoll`以及之前原生的`AIO`提高了不少，值得关注。

再加上`信号驱动IO模型`不适用`TCP协议`，所以目前大部分采用的还是`IO多路复用模型`。

IO线程模型
------

在前边内容的介绍中，我们详述了网络数据包的接收和发送过程，并通过介绍5种`IO模型`了解了内核是如何读取网络数据并通知给用户线程的。

前边的内容都是以`内核空间`的视角来剖析网络数据的收发模型，本小节我们站在`用户空间`的视角来看下如果对网络数据进行收发。

相对`内核`来讲，`用户空间的IO线程模型`相对就简单一些。这些`用户空间`的`IO线程模型`都是在讨论当多线程一起配合工作时谁负责接收连接，谁负责响应IO 读写、谁负责计算、谁负责发送和接收，仅仅是用户IO线程的不同分工模式罢了。

Reactor
-------

`Reactor`是利用`NIO`对`IO线程`进行不同的分工：

*   使用前边我们提到的`IO多路复用模型`比如`select,poll,epoll,kqueue`,进行IO事件的注册和监听。
    
*   将监听到`就绪的IO事件`分发`dispatch`到各个具体的处理`Handler`中进行相应的`IO事件处理`。
    

通过`IO多路复用技术`就可以不断的监听`IO事件`，不断的分发`dispatch`，就像一个`反应堆`一样，看起来像不断的产生`IO事件`，因此我们称这种模式为`Reactor`模型。

下面我们来看下`Reactor模型`的三种分类：

### 单Reactor单线程

![图片](https://mmbiz.qpic.cn/mmbiz_png/sOIZXFW0vUY4ypDTgZibnVV3K2XJZVLcEwlZPO0207A07smWv3l1Gq7rHOMAmRuqePPg2KdO7BPDbSQDrguxz6g/640?wx_fmt=png)单Reactor单线程

`Reactor模型`是依赖`IO多路复用技术`实现监听`IO事件`，从而源源不断的产生`IO就绪事件`，在Linux系统下我们使用`epoll`来进行`IO多路复用`，我们以Linux系统为例：

*   单`Reactor`意味着只有一个`epoll`对象，用来监听所有的事件，比如`连接事件`，`读写事件`。
    
*   `单线程`意味着只有一个线程来执行`epoll_wait`获取`IO就绪`的`Socket`，然后对这些就绪的`Socket`执行读写，以及后边的业务处理也依然是这个线程。
    

`单Reactor单线程`模型就好比我们开了一个很小很小的小饭馆，作为老板的我们需要一个人干所有的事情，包括：迎接顾客（`accept事件`），为顾客介绍菜单等待顾客点菜(`IO请求`)，做菜（`业务处理`），上菜（`IO响应`），送客（`断开连接`）。

### 单Reactor多线程

随着客人的增多（`并发请求`），显然饭馆里的事情只有我们一个人干（`单线程`）肯定是忙不过来的，这时候我们就需要多招聘一些员工（`多线程`）来帮着一起干上述的事情。

于是就有了`单Reactor多线程`模型：

![图片](https://mmbiz.qpic.cn/mmbiz_png/sOIZXFW0vUY4ypDTgZibnVV3K2XJZVLcEo6zebSVIsdVBZIBDtOM5OEOwJxgZfMicrYkyfnz6hLIkiaK8MN4I5Kfw/640?wx_fmt=png)单Reactor多线程

*   这种模式下，也是只有一个`epoll`对象来监听所有的`IO事件`，一个线程来调用`epoll_wait`获取`IO就绪`的`Socket`。
    
*   但是当`IO就绪事件`产生时，这些`IO事件`对应处理的业务`Handler`，我们是通过线程池来执行。这样相比`单Reactor单线程`模型提高了执行效率，充分发挥了多核CPU的优势。
    

### 主从Reactor多线程

做任何事情都要区分`事情的优先级`，我们应该`优先高效`的去做`优先级更高`的事情，而不是一股脑不分优先级的全部去做。

当我们的小饭馆客人越来越多（`并发量越来越大`），我们就需要扩大饭店的规模，在这个过程中我们发现，`迎接客人`是饭店最重要的工作，我们要先把客人迎接进来，不能让客人一看人多就走掉，只要客人进来了，哪怕菜做的慢一点也没关系。

于是，`主从Reactor多线程`模型就产生了：

![图片](https://mmbiz.qpic.cn/mmbiz_png/sOIZXFW0vUY4ypDTgZibnVV3K2XJZVLcEubXjFtTT6Ac0xQxicxT4jwia2OOjiafa2FhzzYh1VGelwHmj8OBdKwukg/640?wx_fmt=png)主从Reactor多线程

*   我们由原来的`单Reactor`变为了`多Reactor`。`主Reactor`用来优先`专门`做优先级最高的事情，也就是迎接客人（`处理连接事件`），对应的处理`Handler`就是图中的`acceptor`。
    
*   当创建好连接，建立好对应的`socket`后，在`acceptor`中将要监听的`read事件`注册到`从Reactor`中，由`从Reactor`来监听`socket`上的`读写`事件。
    
*   最终将读写的业务逻辑处理交给线程池处理。
    

> **注意**：这里向`从Reactor`注册的只是`read事件`，并没有注册`write事件`，因为`read事件`是由`epoll内核`触发的，而`write事件`则是由用户业务线程触发的（`什么时候发送数据是由具体业务线程决定的`），所以`write事件`理应是由`用户业务线程`去注册。

> 用户线程注册`write事件`的时机是只有当用户发送的数据`无法一次性`全部写入`buffer`时，才会去注册`write事件`，等待`buffer重新可写`时，继续写入剩下的发送数据、如果用户线程可以一股脑的将发送数据全部写入`buffer`，那么也就无需注册`write事件`到`从Reactor`中。

`主从Reactor多线程`模型是现在大部分主流网络框架中采用的一种`IO线程模型`。我们本系列的主题`Netty`就是用的这种模型。

Proactor
--------

`Proactor`是基于`AIO`对`IO线程`进行分工的一种模型。前边我们介绍了`异步IO模型`，它是操作系统内核支持的一种全异步编程模型，在`数据准备阶段`和`数据拷贝阶段`全程无阻塞。

`ProactorIO线程模型`将`IO事件的监听`，`IO操作的执行`，`IO结果的dispatch`统统交给`内核`来做。

![图片](https://mmbiz.qpic.cn/mmbiz_png/sOIZXFW0vUY4ypDTgZibnVV3K2XJZVLcEGKYKib71IWfbpGh4URSHfiaPhTiaWfQVZuJzfg4PwDXm7E6iacaMUrnsVg/640?wx_fmt=png)proactor.png

**`Proactor模型`组件介绍：**

*   `completion handler` 为用户程序定义的异步IO操作回调函数，在异步IO操作完成时会被内核回调并通知IO结果。
    
*   `Completion Event Queue` 异步IO操作完成后，会产生对应的`IO完成事件`，将`IO完成事件`放入该队列中。
    
*   `Asynchronous Operation Processor` 负责`异步IO`的执行。执行完成后产生`IO完成事件`放入`Completion Event Queue` 队列中。
    
*   `Proactor` 是一个事件循环派发器，负责从`Completion Event Queue`中获取`IO完成事件`，并回调与`IO完成事件`关联的`completion handler`。
    
*   `Initiator` 初始化异步操作（`asynchronous operation`）并通过`Asynchronous Operation Processor`将`completion handler`和`proactor`注册到内核。
    

**`Proactor模型`执行过程：**

*   用户线程发起`aio_read`，并告诉`内核`用户空间中的读缓冲区地址，以便`内核`完成`IO操作`将结果放入`用户空间`的读缓冲区，用户线程直接可以读取结果（`无任何阻塞`）。
    
*   `Initiator` 初始化`aio_read`异步读取操作（`asynchronous operation`）,并将`completion handler`注册到内核。
    

> 在`Proactor`中我们关心的`IO完成事件`：内核已经帮我们读好数据并放入我们指定的读缓冲区，用户线程可以直接读取。在`Reactor`中我们关心的是`IO就绪事件`：数据已经到来，但是需要用户线程自己去读取。

*   此时用户线程就可以做其他事情了，无需等待IO结果。而内核与此同时开始异步执行IO操作。当`IO操作`完成时会产生一个`completion event`事件，将这个`IO完成事件`放入`completion event queue`中。
    
*   `Proactor`从`completion event queue`中取出`completion event`，并回调与`IO完成事件`关联的`completion handler`。
    
*   在`completion handler`中完成业务逻辑处理。
    

Reactor与Proactor对比
------------------

*   `Reactor`是基于`NIO`实现的一种`IO线程模型`，`Proactor`是基于`AIO`实现的`IO线程模型`。
    
*   `Reactor`关心的是`IO就绪事件`，`Proactor`关心的是`IO完成事件`。
    
*   在`Proactor`中，用户程序需要向内核传递`用户空间的读缓冲区地址`。`Reactor`则不需要。这也就导致了在`Proactor`中每个并发操作都要求有独立的缓存区，在内存上有一定的开销。
    
*   `Proactor` 的实现逻辑复杂，编码成本较 `Reactor`要高很多。
    
*   `Proactor` 在处理`高耗时 IO`时的性能要高于 `Reactor`，但对于`低耗时 IO`的执行效率提升`并不明显`。
    

Netty的IO模型
----------

在我们介绍完`网络数据包在内核中的收发过程`以及五种`IO模型`和两种`IO线程模型`后，现在我们来看下`netty`中的IO模型是什么样的。

在我们介绍`Reactor IO线程模型`的时候提到有三种`Reactor模型`：`单Reactor单线程`，`单Reactor多线程`，`主从Reactor多线程`。

这三种`Reactor模型`在`netty`中都是支持的，但是我们常用的是`主从Reactor多线程模型`。

而我们之前介绍的三种`Reactor`只是一种模型，是一种设计思想。实际上各种网络框架在实现中并不是严格按照模型来实现的，会有一些小的不同，但大体设计思想上是一样的。

下面我们来看下`netty`中的`主从Reactor多线程模型`是什么样子的？

![图片](https://mmbiz.qpic.cn/mmbiz_png/sOIZXFW0vUY4ypDTgZibnVV3K2XJZVLcEqBHAhkKJCkVgiaazsXibAeyzHXtCy8fB3JPwWlq0LL8kWQG6OVwFYDgA/640?wx_fmt=png)netty中的reactor.png

*   `Reactor`在`netty`中是以`group`的形式出现的，`netty`中将`Reactor`分为两组，一组是`MainReactorGroup`也就是我们在编码中常常看到的`EventLoopGroup bossGroup`,另一组是`SubReactorGroup`也就是我们在编码中常常看到的`EventLoopGroup workerGroup`。
    
*   `MainReactorGroup`中通常只有一个`Reactor`，专门负责做最重要的事情，也就是监听连接`accept`事件。当有连接事件产生时，在对应的处理`handler acceptor`中创建初始化相应的`NioSocketChannel`（代表一个`Socket连接`）。然后以`负载均衡`的方式在`SubReactorGroup`中选取一个`Reactor`，注册上去，监听`Read事件`。
    

> `MainReactorGroup`中只有一个`Reactor`的原因是，通常我们服务端程序只会`绑定监听`一个端口，如果要`绑定监听`多个端口，就会配置多个`Reactor`。

*   `SubReactorGroup`中有多个`Reactor`，具体`Reactor`的个数可以由系统参数 `-D io.netty.eventLoopThreads`指定。默认的`Reactor`的个数为`CPU核数 * 2`。`SubReactorGroup`中的`Reactor`主要负责监听`读写事件`，每一个`Reactor`负责监听一组`socket连接`。将全量的连接`分摊`在多个`Reactor`中。
    
*   一个`Reactor`分配一个`IO线程`，这个`IO线程`负责从`Reactor`中获取`IO就绪事件`，执行`IO调用获取IO数据`，执行`PipeLine`。
    

> `Socket连接`在创建后就被`固定的分配`给一个`Reactor`，所以一个`Socket连接`也只会被一个固定的`IO线程`执行，每个`Socket连接`分配一个独立的`PipeLine`实例，用来编排这个`Socket连接`上的`IO处理逻辑`。这种`无锁串行化`的设计的目的是为了防止多线程并发执行同一个socket连接上的`IO逻辑处理`，防止出现`线程安全问题`。同时使系统吞吐量达到最大化

> 由于每个`Reactor`中只有一个`IO线程`，这个`IO线程`既要执行`IO活跃Socket连接`对应的`PipeLine`中的`ChannelHandler`，又要从`Reactor`中获取`IO就绪事件`，执行`IO调用`。所以`PipeLine`中`ChannelHandler`中执行的逻辑不能耗时太长，尽量将耗时的业务逻辑处理放入单独的业务线程池中处理，否则会影响其他连接的`IO读写`，从而近一步影响整个服务程序的`IO吞吐`。

*   当`IO请求`在业务线程中完成相应的业务逻辑处理后，在业务线程中利用持有的`ChannelHandlerContext`引用将响应数据在`PipeLine`中反向传播，最终写回给客户端。
    

`netty`中的`IO模型`我们介绍完了，下面我们来简单介绍下在`netty`中是如何支持前边提到的三种`Reactor模型`的。

### 配置单Reactor单线程

```
EventLoopGroup eventGroup = new NioEventLoopGroup(1);  
ServerBootstrap serverBootstrap = new ServerBootstrap();   
serverBootstrap.group(eventGroup);  

```

### 配置多Reactor线程

```
EventLoopGroup eventGroup = new NioEventLoopGroup();  
ServerBootstrap serverBootstrap = new ServerBootstrap();   
serverBootstrap.group(eventGroup);  

```

### 配置主从Reactor多线程

```
EventLoopGroup bossGroup = new NioEventLoopGroup(1);   
EventLoopGroup workerGroup = new NioEventLoopGroup();  
ServerBootstrap serverBootstrap = new ServerBootstrap();   
serverBootstrap.group(bossGroup, workerGroup);  

```

* * *

总结
--

本文是一篇信息量比较大的文章，用了`25`张图，`22336`个字从内核如何处理网络数据包的收发过程开始展开，随后又在`内核角度`介绍了经常容易混淆的`阻塞与非阻塞`，`同步与异步`的概念。以这个作为铺垫，我们通过一个`C10K`的问题，引出了五种`IO模型`，随后在`IO多路复用`中以技术演进的形式介绍了`select,poll,epoll`的原理和它们综合的对比。最后我们介绍了两种`IO线程模型`以及`netty`中的`Reactor模型`。

感谢大家听我唠叨到这里，哈哈，现在大家可以揉揉眼，伸个懒腰，好好休息一下了。