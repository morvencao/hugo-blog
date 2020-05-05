---
title: "Linux虚拟网络设备"
date: 2020-01-30
type: "notes"
draft: false
---

随着容器逐步取代虚拟机，成为现在云基础架构的标准，这些容器的网络管理模块都离不开Linux虚拟网络设备。事实上，了解常用的Linux虚拟网络设备对于我们理解容器网络以及其他依赖于容器的基础架构网络实现都大有裨益。现在开始，我们就来看看常见的Linux虚拟网络设备有哪些以及它们的典型使用场景。

## 虚拟网络设备

通过上一篇笔记我们也知道，网络设备的驱动程序并不直接与内核中的协议栈交互，而是通过内核的网络设备管理模块。这样做的好处是，驱动程序不需要了解协议栈的细节，协议栈也不需要针对特定驱动处理数据包。

对于内核网络设备管理模块来说，虚拟设备和物理设备没有区别，都是网络设备，都能配置IP，甚至从逻辑上来看，虚拟网络设备和物理网络设备并没有什么区别，它们都类似于管道，从任意一端接收到的数据将从另外一端发送出去。比如物理网卡的两端分别是协议栈于外面的物理网络，从外面物理网络接收到的数据包会转发给协议栈，相反，应用程序通过协议栈发送过来的数据包会通过物理网卡发送到外面的物理网络。但是对于具体将数据包发送到哪里，怎么发送，不同的网络设备有不同的驱动实现，与内核设备管理模块以及协议栈没什么关系。

总的来说，虚拟网络设备与物理网络设备没有什么区别，它们的一端连接着内核协议栈，而另一端的行为是什么取决于不同虚拟网络设备的驱动实现。

### TUN/TAP

TUN/TAP虚拟网络设备一端连着协议栈，另外一端不是物理网络，而是另外一个处于用户空间的应用程序。也就是说，协议栈发给TUN/TAP的数据包能被这个应用程序读取到，当然应用程序能直接向TUN/TAP发送数据包。

一个典型的TUN/TAP的例子如下图所示：

![network-device-tun-tap.jpg](https://i.loli.net/2020/01/28/yDTFvEohmQfiWz5.jpg)

上图中我们配置了一个物理网卡，IP为`18.12.0.92`，而tun0为一个TUN/TAP设备，IP配置为`10.0.0.12`。数据包的流向为：

1. 应用程序A通过socket A发送了一个数据包，假设这个数据包的目的IP地址是`10.0.0.22`
2. socket A将这个数据包丢给协议栈
3. 协议栈根据本地路由规则和数据包的目的IP，将数据包由给tun0设备发送出去
4. tun0收到数据包之后，将数据包转发给给了用户空间的应用程序B
5. 应用程序B收到数据包之后构造一个新的数据包，将原来的数据包嵌入在新的数据包（IPIP包）中，最后通过socket B将数据包转发出去
> Note: 新数据包的源地址变成了eth0的地址，而目的IP地址则变成了另外一个地址`18.13.0.91`.
6. socket B将数据包发给协议栈
7. 协议栈根据本地路由规则和数据包的目的IP，决定将这个数据包要通过eth0发送出去，于是将数据包转发给eth0
8. eth0通过物理网络将数据包发送出去

我们看到发送给`10.0.0.22`的网络数据包通过在用户空间的应用程序B，利用`18.12.0.92`发到远端网络的`18.13.0.91`，网络包到达`18.13.0.91`后，读取里面的原始数据包，读取里面的原始数据包，再转发给本地的`10.0.0.22`。这就是[VPN](https://en.wikipedia.org/wiki/Virtual_private_network)的基本原理。

使用TUN/TAP设备我们有机会将协议栈中的部分数据包转发给用户空间的应用程序，让应用程序处理数据包。常用的使用场景包括数据压缩，加密等功能。

> Note: TUN和TAP设备的区别在于，TUN设备是一个虚拟的端到端IP层设备，也就是说用户空间的应用程序通过TUN设备只能读写IP网络数据包（三层），而TAP设备是一个虚拟的链路层设备，通过TAP设备能读写链路层数据包（二层）。在使用`ip`命令创建设备的时候使用`--dev tun`和`--dev tap`来区分。

### veth

veth虚拟网络设备一端连着协议栈，另外一端不是物理网络，而是另一个veth设备，成对的veth设备中一个数据包发送出去后会直接到另一个veth设备上去。每个veth设备都可以被配置IP地址，并参与三层IP网络路由过程。

下面就是一个典型的veth设备对的例子：

![network-device-veth.jpg](https://i.loli.net/2020/01/28/dOA19SeZHPLanYC.jpg)

我们配置物理网卡eth0的IP为`12.124.10.11`， 而成对出现的veth设备分别为veth0和veth1，它们的IP分别是`20.1.0.10`和`20.1.0.11`。

```
# ip link add veth0 type veth peer name veth1
# ip addr add 20.1.0.10/24 dev veth0
# ip addr add 20.1.0.11/24 dev veth1
# ip link set veth0 up
# ip link set veth1 up
```

然后尝试从veth0设备ping另一个设备veth1:

```
# ping -c 2 20.1.0.11 -I veth0
PING 20.1.0.11 (20.1.0.11) from 20.1.0.11 veth0: 28(42) bytes of data.
64 bytes from 20.1.0.11: icmp_seq=1 ttl=64 time=0.034 ms
64 bytes from 20.1.0.11: icmp_seq=2 ttl=64 time=0.052 ms

--- 20.1.0.11 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1500ms
```

> Note: 在有些Ubuntu中有可能ping不通，原因是默认情况下内核网络配置导致veth设备对无法返回ARP返回包。解决办法是：

```
# echo 1 > /proc/sys/net/ipv4/conf/veth1/accept_local
# echo 1 > /proc/sys/net/ipv4/conf/veth0/accept_local
# echo 0 > /proc/sys/net/ipv4/conf/veth0/rp_filter
# echo 0 > /proc/sys/net/ipv4/conf/veth1/rp_filter
# echo 0 > /proc/sys/net/ipv4/conf/all/rp_filter
```

可以尝试使用tcpdump看看在veth设备对上的请求包：

```
# tcpdump -n -i veth1
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on veth1, link-type EN10MB (Ethernet), capture size 458122 bytes
20:24:12.220002 ARP, Request who-has 20.1.0.11 tell 20.1.0.10, length 28
20:24:12.220198 ARP, Request who-has 20.1.0.11 tell 20.1.0.10, length 28
20:24:12.221372 IP 20.1.0.10 > 20.1.0.11: ICMP echo request, id 18174, seq 1, length 64
20:24:13.222089 IP 20.1.0.10 > 20.1.0.11: ICMP echo request, id 18174, seq 2, length 64
```

可以看到在veth1上面只有ICMP echo的请求包，但是没有应答包。仔细想一下，veth1收到ICMP echo请求包后，转交给另一端的协议栈，但是协议栈检查当前的设备列表，发现本地有`20.1.0.10`，于是构造ICMP echo应答包，并转发给lo设备，lo设备收到数据包之后直接交给协议栈，紧接着给交给用户空间的ping进程。

我们可以尝试使用tcpdump抓取lo设备上的数据：

```
# tcpdump -n -i lo
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on lo, link-type EN10MB (Ethernet), capture size 458122 bytes
20:25:49.486019 IP IP 20.1.0.11 > 20.1.0.10: ICMP echo reply, id 24177, seq 1, length 64
20:25:50.4861228 IP IP 20.1.0.11 > 20.1.0.10: ICMP echo reply, id 24177, seq 2, length 64
```

由此可见，对于成对出现的veth设备对，从一个设备出去的数据包会直接发给另外一个设备。在实际的应用场景中，比如容器网络中，成对的veth设备对处于不同的[网络命名空间](https://blog.scottlowe.org/2013/09/04/introducing-linux-network-namespaces/)中，数据包的转发在不同网络命名空间之间进行，后续在介绍容器网络的时候会详细说明。

### bridge

[bridge](https://wiki.linuxfoundation.org/networking/bridge)一般叫网桥，它也是一种虚拟网络设备，所以具有虚拟网络设备的特征，可以配置IP、MAC地址等。与其他去你网络设备不同的是，bridge是一个虚拟交换机，和物理交换机有类似的功能。bridge一端连接着协议栈，另外一端有多个端口，数据在各个端口间转发是基于MAC地址。

bridge可以工作在二层(链路层)，也可以工作在三层（IP网路层）。默认工作在二层。默认情况下，其工作在二层，可以在同一子网内的的不同主机间转发以太网报文；当给bridge分配了IP地址，也就开启了该bridge的三层工作模式。在Linux下，你可以用[iproute2](https://wiki.linuxfoundation.org/networking/iproute2)或`brctl`命令对bridge进行管理。

创建bridge与创建其他虚拟网络设备类似，只需要制定type为`bridge`:

```
# ip link add name br0 type bridge
# ip link set br0 up
```

![network-device-bridge-1.jpg](https://i.loli.net/2020/01/28/JljuUAEyNRfb6Dq.jpg)

但是这样创建出来的bridge一端连接着协议栈，其他端口什么也没有连接，因此我们需要将其他设备连接到该bridge才能有实际的功能。

```
# ip link add veth0 type veth peer name veth1
# ip addr add 20.1.0.10/24 dev veth0
# ip addr add 20.1.0.11/24 dev veth1
# ip link set veth0 up
# ip link set veth1 up
# 将veth0连接到br0
# ip link set dev veth0 master br0
# 通过bridge link命令可以看到bridge上连接了哪些设备
# bridge link
6: veth0 state UP : <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 master br0 state forwarding priority 32 cost 2
```

![network-device-bridge-2.jpg](https://i.loli.net/2020/01/28/NCUXi4l7zBo83ev.jpg)

事实上，一旦br0和veth0连接后，它们之间将变成双向通道，但是内核协议栈和veth0之间变成了单通道，协议栈能发数据给veth0，但veth0从外面收到的数据不会转发给协议栈
，同时br0的MAC地址变成了veth0的MAC地址。我们可以验证一下：

```
# ping -c 1 -I veth0 20.1.0.11
PING 20.1.0.11 (20.1.0.11) from 20.1.0.10 veth0: 56(84) bytes of data.
From 20.1.0.10 icmp_seq=1 Destination Host Unreachable

--- 20.1.0.11 ping statistics ---
1 packets transmitted, 0 received, +1 errors, 100% packet loss, time 0ms
```

如果我们使用tcpdump在br0上抓包就会发现：

```
# tcpdump -n -i br0
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on br0, link-type EN10MB (Ethernet), capture size 262144 bytes
21:45:48.225459 ARP, Reply 20.1.0.10 is-at a2:85:26:b3:72:6c, length 28
```

可以看到veth0收到应答包后没有给协议栈，而是直接转发给br0，这样协议栈得不到veth1的mac地址，从而ping不通。br0在veth0和协议栈之间数据包给拦截了。但是如果我们给br配置IP，会怎么样呢？

```
# ip addr del 20.1.0.10/24 dev veth0
# ip addr add 20.1.0.10/24 dev br0
```

这样，网络结构就变成了下面这样：

![network-device-bridge-3.jpg](https://i.loli.net/2020/01/28/nuKtLZyaRhXqjDp.jpg)

这时候再通过br0来ping一下veth1，会发现结果可以通：
```
# ping -c 1 -I br0 20.1.0.11
PING 20.1.0.11 (20.1.0.11) from 20.1.0.10 br0: 56(84) bytes of data.
64 bytes from 20.1.0.11: icmp_seq=1 ttl=64 time=0.121 ms

--- 20.1.0.11 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.121/0.121/0.121/0.000 ms
```

其实当去掉veth0的IP地址，而给br0配置了IP之后，协议栈在路由的时候不会将数据包发给veth0，为了表达更直观，我们协议栈和veth0之间的连接线去掉，这时候的veth0相当于一根网线。

在现实中，bridge常用的使用场景：

**虚拟机**

典型的虚拟机网络实现就是通过TUN/TAP将虚拟机内的网卡同宿主机的br0连接起来，这时br0和物理交换机的效果类似，虚拟机发出去的数据包先到达br0，然后由br0交给eth0发送出去，这样做数据包都不需要经过host机器的协议栈，运行效率非常高。

![network-device-vm.jpg](https://i.loli.net/2020/01/28/u4GDayrE3c6kPxs.jpg)

**容器**

而对于容器网络来说，每个容器的网络设备单独的网络命名空间中，所以很好地不同容器的协议栈，我们在接下来的笔记中进一步讨论不同的容器实现。

![network-device-docker.jpg](https://i.loli.net/2020/01/28/IPSwuRVbgTDKnhB.jpg)

## 参考

- https://backreference.org/2010/03/26/tuntap-interface-tutorial/
- https://blog.scottlowe.org/2013/09/04/introducing-linux-network-namespaces/
- https://www.ibm.com/developerworks/cn/linux/1310_xiawc_networkdevice/
- http://ifeanyi.co/posts/linux-namespaces-part-1/
- http://www.opencloudblog.com/?p=66
