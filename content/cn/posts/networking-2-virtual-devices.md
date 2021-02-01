---
title: "Linux 虚拟网络设备"
date: 2020-01-30
categories: ['note', 'tech']
draft: false
---

随着容器逐步取代虚拟机，成为现在云基础架构的标准，这些容器的网络管理模块都离不开 Linux 虚拟网络设备。事实上，了解常用的 Linux 虚拟网络设备对于我们理解容器网络以及其他依赖于容器的基础网络架构实现都大有裨益。现在开始，我们就来看看常见的 Linux 虚拟网络设备有哪些以及它们的典型使用场景。

## 虚拟网络设备

通过上一篇笔记我们也知道，网络设备的驱动程序并不直接与内核中的协议栈交互，而是通过内核的网络设备管理模块作为中间桥梁。这样做的好处是，驱动程序不需要了解网络协议栈的细节，协议栈也不需要针对特定驱动处理数据包。

对于内核网络设备管理模块来说，虚拟设备和物理设备没有区别，都是网络设备，都能配置 IP 地址，甚至从逻辑上来看，虚拟网络设备和物理网络设备都类似于管道，从任意一端接收到的数据将从另外一端发送出去。比如物理网卡的两端分别是协议栈与外面的物理网络，从外面物理网络接收到的数据包会转发给协议栈，相反，应用程序通过协议栈发送过来的数据包会通过物理网卡发送到外面的物理网络。但是对于具体将数据包发送到哪里，怎么发送，不同的网络设备有不同的驱动实现，与内核设备管理模块以及协议栈没关系。

总的来说，虚拟网络设备与物理网络设备没有什么区别，它们的一端连接着内核协议栈，而另一端的行为是什么取决于不同网络设备的驱动实现。

### TUN/TAP

TUN/TAP 虚拟网络设备一端连着协议栈，另外一端不是物理网络，而是另外一个处于用户空间的应用程序。也就是说，协议栈发给 TUN/TAP 的数据包能被这个应用程序读取到，当然应用程序能直接向 TUN/TAP 发送数据包。

一个典型的使用 TUN/TAP 网络设备的例子如下图所示：

![network-device-tun-tap.jpg](https://i.loli.net/2021/02/01/5NYEzLXpmPSg8on.jpg)

上图中我们配置了一个物理网卡，IP 为`18.12.0.92`，而 tun0 为一个 TUN/TAP 设备，IP 配置为`10.0.0.12`。数据包的流向为：

1. 应用程序 A 通过 socket A 发送了一个数据包，假设这个数据包的目的 IP 地址是 `10.0.0.22`
2. socket A 将这个数据包丢给网络协议栈
3. 协议栈根据本地路由规则和数据包的目的 IP，将数据包由给 tun0 设备发送出去
4. tun0 收到数据包之后，将数据包转发给了用户空间的应用程序 B
5. 应用程序 B 收到数据包之后构造一个新的数据包，将原来的数据包嵌入在新的数据包（IPIP 包）中，最后通过 socket B 将数据包转发出去

> Note: 新数据包的源地址变成了 tun0 的地址，而目的 IP 地址则变成了另外一个地址 `18.13.0.91`.

6. socket B 将数据包发给协议栈
7. 协议栈根据本地路由规则和数据包的目的 IP，决定将这个数据包要通过设备 eth0 发送出去，于是将数据包转发给设备 eth0
8. 设备 eth0 通过物理网络将数据包发送出去

我们看到发送给`10.0.0.22`的网络数据包通过在用户空间的应用程序 B，利用`18.12.0.92`发到远端网络的`18.13.0.91`，网络包到达`18.13.0.91`后，读取里面的原始数据包，再转发给本地的`10.0.0.22`。这就是 [VPN](https://en.wikipedia.org/wiki/Virtual_private_network) 的基本实现原理。

使用 TUN/TAP 设备我们有机会将协议栈中的部分数据包转发给用户空间的应用程序，让应用程序处理数据包。常用的使用场景包括数据压缩、加密等功能。

> Note: TUN 和 TAP 设备的区别在于，TUN 设备是一个虚拟的端到端 IP 层设备，也就是说用户空间的应用程序通过 TUN 设备只能读写 IP 网络数据包（三层），而 TAP 设备是一个虚拟的链路层设备，通过 TAP 设备能读写链路层数据包（二层）。如果使用 Linux 网络工具包 [iproute2](https://en.wikipedia.org/wiki/Iproute2) 来创建网络设备 TUN/TAP 设备
则需要指定 `--dev tun` 和 `--dev tap` 来区分。

### veth

veth 虚拟网络设备一端连着协议栈，另外一端不是物理网络，而是另一个 veth 设备，成对的 veth 设备中一个数据包发送出去后会直接到另一个 veth 设备上去。每个 veth 设备都可以配置 IP 地址，并参与三层 IP 网络的路由过程。

下面就是一个典型的使用 veth 设备对的例子：

![network-device-veth.jpg](https://i.loli.net/2020/01/28/dOA19SeZHPLanYC.jpg)

我们配置物理网卡 eth0 的 IP 地址为`12.124.10.11`，这里 veth 设备对分别为 veth0 和 veth1，它们的 IP 分别是`20.1.0.10`和`20.1.0.11`：

```bash
# ip link add veth0 type veth peer name veth1
# ip addr add 20.1.0.10/24 dev veth0
# ip addr add 20.1.0.11/24 dev veth1
# ip link set veth0 up
# ip link set veth1 up
```

然后尝试从 veth0 设备 ping 另一个设备 veth1：

```bash
# ping -c 2 20.1.0.11 -I veth0
PING 20.1.0.11 (20.1.0.11) from 20.1.0.11 veth0: 28(42) bytes of data.
64 bytes from 20.1.0.11: icmp_seq=1 ttl=64 time=0.034 ms
64 bytes from 20.1.0.11: icmp_seq=2 ttl=64 time=0.052 ms

--- 20.1.0.11 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1500ms
```

> Note: 在有些版本的 Ubuntu 中有可能会 ping 不通，原因是默认情况下内核网络配置导致 veth 设备对无法返回 ARP 包，解决办法是配置 veth 设备可以返回 ARP 包：

```bash
# echo 1 > /proc/sys/net/ipv4/conf/veth1/accept_local
# echo 1 > /proc/sys/net/ipv4/conf/veth0/accept_local
# echo 0 > /proc/sys/net/ipv4/conf/veth0/rp_filter
# echo 0 > /proc/sys/net/ipv4/conf/veth1/rp_filter
# echo 0 > /proc/sys/net/ipv4/conf/all/rp_filter
```

可以尝试使用 tcpdump 命令看看在 veth 设备对上的请求包：

```bash
# tcpdump -n -i veth1
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on veth1, link-type EN10MB (Ethernet), capture size 458122 bytes
20:24:12.220002 ARP, Request who-has 20.1.0.11 tell 20.1.0.10, length 28
20:24:12.220198 ARP, Request who-has 20.1.0.11 tell 20.1.0.10, length 28
20:24:12.221372 IP 20.1.0.10 > 20.1.0.11: ICMP echo request, id 18174, seq 1, length 64
20:24:13.222089 IP 20.1.0.10 > 20.1.0.11: ICMP echo request, id 18174, seq 2, length 64
```

可以看到在 veth1 上面只有 ICMP echo 的请求包，但是没有应答包。仔细想一下，veth1 收到 ICMP echo请求包后，转交给另一端的协议栈，但是协议栈检查当前的设备列表，发现本地有`20.1.0.10`，于是构造 ICMP echo 应答包，并转发给 lo 设备，lo 设备收到数据包之后直接交给协议栈，紧接着给交给用户空间的 ping 进程。

我们可以尝试使用 tcpdump 抓取 lo 设备上的数据：

```bash
# tcpdump -n -i lo
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on lo, link-type EN10MB (Ethernet), capture size 458122 bytes
20:25:49.486019 IP IP 20.1.0.11 > 20.1.0.10: ICMP echo reply, id 24177, seq 1, length 64
20:25:50.4861228 IP IP 20.1.0.11 > 20.1.0.10: ICMP echo reply, id 24177, seq 2, length 64
```

由此可见，对于成对出现的 veth 设备对，从一个设备出去的数据包会直接发给另外一个设备。在实际的应用场景中，比如容器网络中，成对的 veth 设备对处于不同的[网络命名空间](https://blog.scottlowe.org/2013/09/04/introducing-linux-network-namespaces/)中，数据包的转发在不同网络命名空间之间进行，后续在介绍容器网络的时候会详细说明。

### bridge

[bridge](https://wiki.linuxfoundation.org/networking/bridge) 一般叫作“网桥”，也是一种虚拟网络设备，所以具有虚拟网络设备的特征，可以配置 IP、MAC 地址等。与其他网络设备不同的是，bridge 是一个虚拟交换机，和物理交换机有类似的功能。bridge 一端连接着协议栈，另外一端有多个端口，数据在各个端口间转发数据包是基于 MAC 地址。

bridge 可以工作在二层(链路层)，也可以工作在三层（IP 网路层）。默认情况下，其工作在二层，可以在同一子网内的的不同主机间转发以太网报文；当给 bridge 分配了 IP 地址，也就开启了该 bridge 的三层工作模式。在 Linux 下，你可以用 [iproute2](https://wiki.linuxfoundation.org/networking/iproute2) 或 `brctl` 命令对 bridge 进行管理。

创建 bridge 与创建其他虚拟网络设备类似，只需要指定 type 参数为 `bridge`：

```bash
# ip link add name br0 type bridge
# ip link set br0 up
```

![network-device-bridge-1.jpg](https://i.loli.net/2020/01/28/JljuUAEyNRfb6Dq.jpg)

但是这样创建出来的 bridge 一端连接着协议栈，其他端口什么也没有连接，因此我们需要将其他设备连接到该 bridge 才能有实际的功能：

```bash
# ip link add veth0 type veth peer name veth1
# ip addr add 20.1.0.10/24 dev veth0
# ip addr add 20.1.0.11/24 dev veth1
# ip link set veth0 up
# ip link set veth1 up
# 将 veth0 连接到 br0
# ip link set dev veth0 master br0
# 通过 bridge link 命令可以看到 bridge 上连接了哪些设备
# bridge link
6: veth0 state UP : <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 master br0 state forwarding priority 32 cost 2
```

![network-device-bridge-2.jpg](https://i.loli.net/2020/01/28/NCUXi4l7zBo83ev.jpg)

事实上，一旦 br0 和 veth0 连接之后，它们之间将变成双向通道，但是内核协议栈和 veth0 之间变成了单通道，协议栈能发数据给 veth0，但 veth0 从外面收到的数据不会转发给协议栈，同时 br0 的 MAC 地址变成了 veth0 的 MAC 地址。我们可以验证一下：

```bash
# ping -c 1 -I veth0 20.1.0.11
PING 20.1.0.11 (20.1.0.11) from 20.1.0.10 veth0: 56(84) bytes of data.
From 20.1.0.10 icmp_seq=1 Destination Host Unreachable

--- 20.1.0.11 ping statistics ---
1 packets transmitted, 0 received, +1 errors, 100% packet loss, time 0ms
```

如果我们使用 tcpdump 在 br0 上抓包就会发现：

```bash
# tcpdump -n -i br0
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on br0, link-type EN10MB (Ethernet), capture size 262144 bytes
21:45:48.225459 ARP, Reply 20.1.0.10 is-at a2:85:26:b3:72:6c, length 28
```

可以看到 veth0 收到应答包后没有给协议栈，而是直接转发给 br0，这样协议栈得不到 veth1 的 MAC 地址，从而 ping 不通。br0 在 veth0 和协议栈之间将数据包给拦截了。但是如果我们给 br0 配置 IP，会怎么样呢？

```bash
# ip addr del 20.1.0.10/24 dev veth0
# ip addr add 20.1.0.10/24 dev br0
```

这样，网络结构就变成了下面这样：

![network-device-bridge-3.jpg](https://i.loli.net/2020/01/28/nuKtLZyaRhXqjDp.jpg)

这时候再通过 br0 来 ping 一下 veth1，会发现结果可以通：

```bash
# ping -c 1 -I br0 20.1.0.11
PING 20.1.0.11 (20.1.0.11) from 20.1.0.10 br0: 56(84) bytes of data.
64 bytes from 20.1.0.11: icmp_seq=1 ttl=64 time=0.121 ms

--- 20.1.0.11 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.121/0.121/0.121/0.000 ms
```

其实当去掉 veth0 的 IP，而给 br0 配置了 IP 之后，协议栈在路由的时候不会将数据包发给 veth0，为了表达更直观，我们协议栈和 veth0 之间的连接线去掉，这时候的 veth0 相当于一根网线。

在现实中，bridge 常用的使用场景：

**虚拟机**

典型的虚拟机网络实现就是通过 TUN/TAP 将虚拟机内的网卡同宿主机的 br0 连接起来，这时 br0 和物理交换机的效果类似，虚拟机发出去的数据包先到达 br0，然后由 br0 交给 eth0 发送出去，这样做数据包都不需要经过宿主机的协议栈，运行效率非常高。

![network-device-vm.jpg](https://i.loli.net/2020/01/28/u4GDayrE3c6kPxs.jpg)

**容器**

而对于容器网络来说，每个容器的网络设备单独的网络命名空间中，所以很好地不同容器的协议栈，我们在接下来的笔记中进一步讨论不同的容器网络实现。

![network-device-docker.jpg](https://i.loli.net/2020/01/28/IPSwuRVbgTDKnhB.jpg)

## 参考

- https://backreference.org/2010/03/26/tuntap-interface-tutorial/
- https://blog.scottlowe.org/2013/09/04/introducing-linux-network-namespaces/
- https://www.ibm.com/developerworks/cn/linux/1310_xiawc_networkdevice/
- http://ifeanyi.co/posts/linux-namespaces-part-1/
- http://www.opencloudblog.com/?p=66
