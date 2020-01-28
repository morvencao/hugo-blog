---
title: "Linux网络数据包的接收与发送过程"
date: 2020-01-27
type: "notes"
draft: false
---

早在农历新年之前，就构思着将近半年重拾的网络基础整理成成一个系列，正好赶上武汉疫情在春节假期爆发，闲来无事，于是开始这一系列的笔记。

我的整体思路是：

- 第一篇笔记会简单介绍Linux网络数据包接收和发送过程，但不涉及TCP/IP协议栈的细节知识，如果有需要了解这些基础知识的读者，我推荐去阅读[阮一峰老师的互联网协议的系列文章](http://www.ruanyifeng.com/blog/2012/05/internet_protocol_suite_part_i.html)。
- 接下来，我会总结常用的Linux虚拟设备，同时会结合Linux自带的新网络工具包[iproute2](https://en.wikipedia.org/wiki/Iproute2)来操作这些常用的网络设备。
- 有了前面的基础知识，我们再来了解几种常用的容器网络实现远离。
- 最后，我们深入探讨一下K8s平台中主流的网络实现。

其实，严格上来说，这种学习思路其实很“急功近利”，但是，对于不太了解网络基础知识和Linux内核原理的人来说，这反而是一种很有效的学习方式。但是，私以为学习过程不只应该由浅入深，更应该螺旋向前迭代，温故而知新，才能获益良多。

----------------------------------------

## 数据包的接收过程

废话不多说，我们先开始第一篇笔记的内容。

为了简化起见，我们以一个UDP数据包在物理网卡上处理流程来介绍Linux网络数据包的接收和发送过程，我会尽量忽略一些不相关的细节。

### 从网卡到内存

我们知道，每个网络设备（网卡）需要有驱动才能工作，驱动需要在内核启动时加载到内核中才能工作。事实上，从逻辑上看，驱动是负责衔接网络设备和内核网络栈的中间模块，每当网络设备接收到新的数据包时，就会触发中断，而对应的中断处理程序正是加载到内核中的驱动程序。

下面这张图详细的展示了数据包如何从网络设备进入内存，并被处于内核中的驱动程序和网络栈处理的：

![network-receive-data-1.jpg](https://i.loli.net/2020/01/27/8ZsXoQ1emVSzJlc.jpg)


1. 数据包进入物理网卡。如果目的地址不是该网络设备，且该来网络设备没有开启[混杂模式](https://unix.stackexchange.com/questions/14056/what-is-kernel-ip-forwarding)，该包会被网络设备丢弃。
2. 物理网卡将数据包通过DMA的方式写入到指定的内存地址，该地址由网卡驱动分配并初始化。
3. 物理网卡通过硬件中断（IRQ）通知CPU，有新的数据包到达物理网卡需要处理。
4. CPU根据中断表，调用已经注册的中断函数，这个中断函数会调到驱动程序（NIC Driver）中相应的函数
5. 驱动先禁用网卡的中断，表示驱动程序已经知道内存中有数据了，告诉物理网卡下次再收到数据包直接写内存就可以了，不要再通知CPU了，这样可以提高效率，避免CPU不停的被中断。
6. 启动软中断继续处理数据包。这样的原因是硬中断处理程序执行的过程中不能被中断，所以如果它执行时间过长，会导致CPU没法响应其它硬件的中断，于是内核引入软中断，这样可以将硬中断处理函数中耗时的部分移到软中断处理函数里面来慢慢处理。

### 内核处理数据包

上一步中网络设备驱动程序会通过软触发内核网络模块中的软中断处理函数，内核处理数据包的流程如下图所示：

![network-receive-data-2.jpg](https://i.loli.net/2020/01/27/y2SZleoIwtbxDLs.jpg)

7. 对于第6步中驱动发出的软中断，内核中的`ksoftirqd`进程会调用网络模块的相应软中断所对应的处理函数，这里其实就是调用`net_rx_action`函数。
8. 接下来`net_rx_action`调用网卡驱动里的`poll`函数来一个个地处理数据包。
9. 而`poll`函数会让驱动会读取网卡写到内存中的数据包。事实上，内存中数据包的格式只有驱动知道。
10. 驱动程序将内存中的数据包转换成内核网络模块能识别的`skb`(socket buffer)格式，然后调用`napi_gro_receive`函数
11. `napi_gro_receive`会处理[GRO](https://lwn.net/Articles/358910/)相关的内容，也就是将可以合并的数据包进行合并，这样就只需要调用一次协议栈。然后判断是否开启了RPS，如果开启了，将会调用`enqueue_to_backlog`。
12. `enqueue_to_backlog`函数会将数据包放入`input_pkt_queue`结构体中，然后返回。
> Note: 如果`input_pkt_queue`满了的话，该数据包将会被丢弃，这个queue的大小可以通过`net.core.netdev_max_backlog`来配置
13. 接下来CPU会在软中断上下文中处理自己`input_pkt_queue`里的网络数据（调用`__netif_receive_skb_core`函数）
14. 如果没开启[RPS](https://github.com/torvalds/linux/blob/v3.13/Documentation/networking/scaling.txt#L99-L222)，`napi_gro_receive`会直接调用`__netif_receive_skb_core`函数。
15. 紧接着CPU会根据是不是有`AF_PACKET`类型的socket（原始套接字），如果有的话，拷贝一份数据给它(`tcpdump`抓包就是抓的这里的包)。
16. 将数据包交给内核协议栈处理。
17. 当内存中的所有数据包被处理完成后（`poll`函数执行完成），重新启用网卡的硬中断，这样下次网卡再收到数据的时候就会通知CPU。

### 内核协议栈

内核网络协议栈此时接收到的数据包其实是三层(IP网络层)数据包，因此，数据包首先会进入到IP网络层层，然后进入传输层处理。

#### IP网络层

![network-receive-data-3.jpg](https://i.loli.net/2020/01/27/oxfqD6Upiw7Blbt.jpg)

- ip_rcv: `ip_rcv`函数是IP网络层处理模块的入口函数，该函数首先判断属否需要丢弃该数据包（目的mac地址不是当前网卡，并且网卡设置了混杂模式），如果需要进一步处理就然后调用注册在netfilter中的`NF_INET_PRE_ROUTING`这条链上的处理函数。
- NF_INET_PRE_ROUTING: netfilter放在协议栈中的钩子函数，可以通过iptables来注入一些数据包处理函数，用来修改或者丢弃数据包，如果数据包没被丢弃，将继续往下走。
> `NF_INET_PRE_ROUTING`等netfilter链上的处理逻辑可以通iptables来设置，详情请移步: https://morven.life/notes/the_knowledge_of_iptables/
- routing: 进行路由处理，如果是目的IP不是本地IP，且没有开启`ip forward`功能，那么数据包将被丢弃，如果开启了`ip forward`功能，那将进入`ip_forward`函数。
- ip_forward: 该函数会先调用`netfilter`注册的`NF_INET_FORWARD`链上的相关函数，如果数据包没有被丢弃，那么将继续往后调用`dst_output_sk`函数。
- dst_output_sk: 该函数会调用IP网络层的相应函数将该数据包发送出去，这一步将会在下一章节发送数据包中详细介绍。
- ip_local_deliver: 如果上面路由处理发现发现目的IP是本地IP，那么将会调用`ip_local_deliver`函数，该函数先调用`NF_INET_LOCAL_IN`链上的相关函数，如果通过，数据包将会向下发送到UDP层。

### 传输层

![network-receive-data-4.jpg](https://i.loli.net/2020/01/27/Hcyw6pFJDLVtZ8j.jpg)

- udp_rcv: 该函数是UDP处理层模块的入口函数，它首先调用`__udp4_lib_lookup_skb`函数，根据目的IP和端口找对应的`socket`，如果没有找到相应的`socket`，那么该数据包将会被丢弃，否则继续。
- sock_queue_rcv_skb: 该函数一是负责检查这个socket的receive buffer是不是满了，如果满了的话就丢弃该数据包；二是调用`sk_filter`看这个包是否是满足条件的包，如果当前socket上设置了filter，且该包不满足条件的话，这个数据包也将被丢弃。
- __skb_queue_tail: 该函数将数据包放入socket接收队列的末尾。
- sk_data_ready: 通知socket数据包已经准备好。
- 调用完sk_data_ready之后，一个数据包处理完成，等待应用层程序来读取。

> Note: 上面所述的所有执行过程都在软中断的上下文中执行。

----------------------------------------

## 数据包的发送过程

从逻辑上看，Linux网络数据包的发送过程和接收过程正好相反，我们仍旧以一个UDP数据包通过物理网卡发送的过程为例来讲解：

### 应用层

应用层处理过程的起点是应用程序调用Linux网络接口创建socket（所谓socket基本就是ip+端口组成的基本结构体），下面这张图详细的展示了应用层如何构建socket并发送给下层UDP：

![network-send-data-1.jpg](https://i.loli.net/2020/01/27/XFTDfq37OQm9ZAa.jpg)

- socket(...): 调用该函数来创建一个socket结构体，并初始化相应的操作函。
- sendto(sock, ...): 应用层程序调用该函数开始发送数据包，该函数数会调用后面的`inet_sendmsg`。
- inet_sendmsg: 该函数主要是检查当前socket有没有绑定源端口，如果没有的话，调用`inet_autobind`分配一个，然后调用UDP层的函数。
- inet_autobind: 该函数会调用socket上绑定的`get_port`函数获取一个可用的端口。

### 传输层

![network-send-data-2.jpg](https://i.loli.net/2020/01/27/Eojq5mIZty3zJK7.jpg)

- udp_sendmsg: 该函数是UDP传输层模块发送数据包的入口。该函数中先调用`ip_route_output_flow`获取路由信息（主要包括源IP和网卡），然后调用`ip_make_skb`构造skb结构体，最后将网卡的信息和该skb关联。
- ip_route_output_flow: 该函数主要处理路由信息，它会根据路由表和目的IP，找到这个数据包应该从哪个设备发送出去，如果该socket没有绑定源IP，该函数还会根据路由表找到一个最合适的源IP给它。 如果该socket已经绑定了源IP，但根据路由表，从这个源IP对应的网卡没法到达目的地址，则该包会被丢弃，于是数据发送失败将返回错误。该函数最后会将找到的设备和源IP塞进flowi4结构体并返回给`udp_sendmsg`。
- ip_make_skb: 该函数的功能是构造skb包，构造好的skb包里面已经分配了IP包头(包括源IP信息)，同时该函数会调用`__ip_append_dat`，如果需要分片的话，会在`__ip_append_data`函数中进行分片，同时还会在该函数中检查socket的send buffer是否已经用光，如果被用光的话，返回ENOBUFS。
- udp_send_skb(skb, fl4): 该函数主要是往skb里面填充UDP的包头，同时处理checksum，然后交给IP网络层层的相应函数。

### IP网络层

![network-send-data-3.jpg](https://i.loli.net/2020/01/28/dRVXQnmpxbZoqwG.jpg)

- ip_send_skb: IP网络层模块发送数据包的入口，该函数主要是调用后面的一些列函数。
- __ip_local_out_sk: 用来设置IP报文头的长度和checksum，然后调用下面netfilter的钩子链`NF_INET_LOCAL_OUT`。
- NF_INET_LOCAL_OUT: netfilter的钩子函数，可以通过iptables来配置处理函数链；如果该数据包没被丢弃，则继续往下走。
- dst_output_sk: 该函数根据skb里面的信息，调用相应的output函数`ip_output`。
- ip_output: 将上一层`udp_sendmsg`得到的网卡信息写入skb，然后调用        `NF_INET_POST_ROUTING`的钩子链。
- NF_INET_POST_ROUTING: 在这一步主要在配置了SNAT，从而导致该skb的路由信息发生变化。
- ip_finish_output: 这里会判断经过了上一步后，路由信息是否发生变化，如果发生变化的话，需要重新调用`dst_output_sk`（重新调用这个函数时，可能就不会再走到`ip_output`，而是走到被netfilter指定的output函数里，这里有可能是`xfrm4_transport_output`），否则接着往下走。
- ip_finish_output2: 根据目的IP到路由表里面找到下一跳(nexthop)的地址，然后调用`__ipv4_neigh_lookup_noref`去arp表里面找下一跳的neigh信息，没找到的话会调用`__neigh_create`构造一个空的neigh结构体。
- dst_neigh_output: 该函数调用`neigh_resolve_output`获取neigh信息，并将neigh信息里面的mac地址填到skb中，然后调用`dev_queue_xmit`发送数据包。
- neigh_resolve_output: 该函数里面会发送arp请求，得到下一跳的mac地址，然后将mac地址填到skb中并调用`dev_queue_xmit`。

### 内核处理数据包

![network-send-data-4.jpg](https://i.loli.net/2020/01/28/bmCiIl3EBXnsSfY.jpg)

- dev_queue_xmit: 内核模块开始处理发送数据包的入口函数，该函数会先获取设备对应的qdisc，如果没有的话（如loopback或者IP tunnels），就直接调用`dev_hard_start_xmit`，否则数据包将经过`traffic control`模块进行处理。
- traffic control：该模块主要对数据包进行过滤和排序，如果队列满了的话，数据包会被丢掉，详情请参考: http://tldp.org/HOWTO/Traffic-Control-HOWTO/intro.html
- dev_hard_start_xmit: 该函数先拷贝一份skb给“packet taps”(tcpdump的数据就从来自于此），然后调用`ndo_start_xmit`函数。如果`dev_hard_start_xmit`返回错误的话，调用它的函数会把skb放到一个地方，然后抛出软中断`NET_TX_SOFTIRQ`，然后交给软中断处理程序`net_tx_action`稍后重试。
- ndo_start_xmit：该函数绑定到具体驱动发送数据的处理函数。

> Note: `ndo_start_xmit`会指向具体网卡驱动的发送数据包的函数，这一步之后，数据包发送任务就交给网络设备驱动了，不同的网络设备驱动有不同的处理方式，但是大致流程基本一致：

1. 将skb放入网卡自己的发送队列
2. 通知网卡发送数据包
3. 网卡发送完成后发送中断给CPU
4. 收到中断后进行skb的清理工作

## 总结

理解了Linux网络数据包的接收和发送流程，我们就可以知道在哪些地方监控和修改数据包，哪些情况下数据包可能被丢弃，特别是了解了netfilter中相应钩子函数的位置，对于了解iptables的用法有一定的帮助，同时也会帮助我们更好的理解Linux下的网络虚拟设备。

## 参考

- https://blog.packagecloud.io/eng/2016/06/22/monitoring-tuning-linux-networking-stack-receiving-data/
- https://blog.packagecloud.io/eng/2016/10/11/monitoring-tuning-linux-networking-stack-receiving-data-illustrated/
