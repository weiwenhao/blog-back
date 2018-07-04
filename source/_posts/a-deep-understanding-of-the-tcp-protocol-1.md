---
title: 深入理解TCP协议（一）
date: 2018-02-09 23:39:24
tags:
---

<!-- more -->

## 三路握手

1. 服务器端通过调用 socket、bind和1isten这3个函数准备好接受外来的连接。称之为被动打开（passive open）

2. 客户端通过调用 connect发起主动打开( active open)。该调用使客户端发送一个SYN(同步)分节给服务器，SYN中包含客户将在(待建立的)连接中发送的数据的初始序列号。**通常SYN分节不携带数据**， 其所在IP数据报只含有一个IP首部、一个TCP首部及可能有的TCP选项。
3. 服务器必须确认(ACK)客户的SYN,同时自己也得发送一个SYN分节，它含有服务器将在同一连接中发送的数据的初始序列号。服务器在单个分节中发送SYN和对客户SYN的ACK(确认)。
4. 客户必须确认服务器的SYN。既发送一个确认SYN的ACK。

这种交换至少需要3个分组,因此称之为TCP的三路握手(thre-way handshake)。
![](http://omjq5ny0e.bkt.clouddn.com/15175486060759.jpg)

可以看到服务端调用accept()函数后，一直处于阻塞状态。直到三路握手完成后，accept函数才返回。然后再阻塞于read状态

## TCP状态转换图
TCP为一个连接定义了11种状态
 
![](http://omjq5ny0e.bkt.clouddn.com/15179303680473.jpg)

其中ESTABLISHED为（客户端或服务端）完成三路握手后的状态，这个最终状态在数据发送期间一直保持。


主动调用close一方经历了TIME_WAIT状态，  在TIME_WAIT状态下会持续 2MSL (时间单位)。
**MSL: maximum segment lifetime**
实现TCP时必须必须为MSL选择一个值，建议在30秒~2分钟。
MSL是任何IP数据报能够在因特网中存活的最长时间。

TIME_WAIT状态有两个存在的理由：
(1)可靠地实现TCP全双工连接的终止
(2)允许老的重复分节在网络中消逝，因为TCP协议规定在TIME_WAIT状态下的一端不允许创建新的TCP连接。从而保证旧连接的任何分节都不会发送到新的连接中去（分节在因特网中的最长存活时间小于TIME_WAIT持续的时间）。



![](http://omjq5ny0e.bkt.clouddn.com/15179304017422.jpg)


上图服务器对客户的请求的确认是伴随其应答发送的,这种做法称为 piggybacking，
它通常在服务器请求并产生的应答的时间少于200ms时发生。

## 并发服务器
![](http://omjq5ny0e.bkt.clouddn.com/15181013775313.jpg)
如图为一个多进程模型的并发服务器

我们发现子进程1和子进程2的connect_socket甚至是listen_socket都使用了服务器端的21端口。

当客户端请求12.106.32.254:21端口时，tcp协议应该将该请求交给三个socket中的哪一个呢？

首先通过上图可以确定的是无法仅仅通过目的端口将到来的请求交给对应的socket。

**书（unix网络编程卷3）中的解释是：**  TCP协议必须查看**套接字对**的所有四个元素，才能确定由哪个socket接受到达的客户端请求。

> 对于tcp请求，我们可以在分节的tcp标头中得到这四个元素（请求双方的id和端口号）
![](http://omjq5ny0e.bkt.clouddn.com/15181892735181.jpg)


**下面是我根据书中的解释做出的臆想**

对于一台并发服务器来说，一个简单的算法是：对于一个到来的分节。首先查看其目的端口(对于单ip服务器来说，再查看分节的目的ip就有些多此一举了)。

然后检索所有监听了该目的端口的socket。假如只检索到一个，那么直接将该分节交给该socket即可。

当检索出了多个socket时 (上图中多个已连接socket的情况)，我们可以根据客户端的ip与端口号进行匹配，当然这一次是客户端的ip与端口都需要参与匹配。
优先匹配具有精确客户端地址的socket 如服务器的子进程2的socket中保存的socket pair为 `{12.106.32.254:21,206.168.112.219:1501}`
其精确的标识了请求端的ip地址和端口号 206.168.112.219:1501
其次再匹配通配的socket。如监听socket

当然要做到上面匹配需要我们服务器端的socket中保存了4个元素。实际上也确实如此

socket中包含的一个inet_sock的结构体如下。其中包含确定一个tcp连接所需的4个元素

```
struct inet_sock   
{   
    struct sock sk;   
#if defined(CONFIG_IPV6) || defined(CONFIG_IPV6_MODULE)   
    struct ipv6_pinfo   *pinet6;   
#endif   
    __u32           daddr;          //IPv4的目的地址。   
    __u32           rcv_saddr;      //IPv4的本地接收地址。   
    __u16           dport;          //目的端口。   
    __u16           num;            //本地端口（主机字节序）。  
    
    …………      
}
```

