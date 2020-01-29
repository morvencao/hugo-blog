---
title: "IPIP隧道实践"
date: 2020-01-29
type: "notes"
draft: false
---

上一篇笔记中，我们在介绍TUN/TAP网络设备的时候介绍了一种典型的VPN实现拓扑，但是并没有实践TUN/TAP虚拟网络设备具体在linux中怎么发挥实际的功能。这篇笔记我们就来看看在云计算领域中一种非常典型的IPIP隧道试图和通过TUN设备来实现。

## IPIP隧道

上一篇笔记中我们也提到了，TUN设备能将三层（IP）网络包封装在另外一个三层网络包之中，看起来通过TUN设备发送出来的数据包会像会这样：

```
MAC: xx:xx:xx:xx:xx:xx
IP Header: <new destination IP>
IP Body:
  IP: <original destination IP>
  TCP: stuff
  HTTP: stuff
```

这就是典型的IPIP隧道数据包的结构。Linux原生支持好几种不同的IPIP隧道类型，但都依赖于TUN虚拟网络设备，我们可以通过命令`ip tunnel help`来查看IPIP隧道的相关类型以及操作：

```
# ip tunnel help
Usage: ip tunnel { add | change | del | show | prl | 6rd } [ NAME ]
          [ mode { ipip | gre | sit | isatap | vti } ] [ remote ADDR ] [ local ADDR ]
          [ [i|o]seq ] [ [i|o]key KEY ] [ [i|o]csum ]
          [ prl-default ADDR ] [ prl-nodefault ADDR ] [ prl-delete ADDR ]
          [ 6rd-prefix ADDR ] [ 6rd-relay_prefix ADDR ] [ 6rd-reset ]
          [ ttl TTL ] [ tos TOS ] [ [no]pmtudisc ] [ dev PHYS_DEV ]

Where: NAME := STRING
       ADDR := { IP_ADDRESS | any }
       TOS  := { STRING | 00..ff | inherit | inherit/STRING | inherit/00..ff }
       TTL  := { 1..255 | inherit }
       KEY  := { DOTTED_QUAD | NUMBER }
```

其中`mode`代表不同的IPIP隧道类型，Linux原生共支持5种IPIP隧道：

1. ipip: 普通的IPIP隧道，就是在报文的基础上再封装一个IPv4报文
2. gre: 通用路由封装（Generic Routing Encapsulation），定义了在任意一种网络层协议上封装其他任意一种网络层协议的机制，所以对于IPv4和IPv6都适用
3. sit: sit模式主要用于IPv4报文封装IPv6报文，即IPv6 over IPv4
4. isatap: 站内自动隧道寻址协议（Intra-Site Automatic Tunnel Addressing Protocol），类似于sit也是用于IPv6的隧道封装
5. vti: 即虚拟隧道接口（Virtual Tunnel Interface），是一种IPsec隧道技术

还有一些有用的参数：
- ttl N 设置进入通道数据包的TTL为N（N是一个1—255之间的数字，0是一个特殊的值，表示这个数据包的TTL值是继承(inherit)的），ttl参数的缺省值是为inherit
- tos T/dsfield T 设置进入通道数据包的TOS域，缺省是inherit
- [no]pmtudisc 在这个通道上禁止路径最大传输单元发现(Path MTU Discovery)，默认打开的

> Note: nopmtudisc选项和固定的ttl是不兼容的，如果使用了固定的ttl参数，系统会打开路径最大传输单元发现( Path MTU Discovery)功能

## ipip模式实践

我们以最基本的ipip模式为例来介绍如何在linux中搭建IPIP隧道来实现不同子网之间的通信。

开始之前需要注意的是，并不是所有的linux发行版都会默认加载`ipip.ko`模块，可以通过`lsmod | grep ipip`查看内核是否加载该模块；若没有则用`modprobe ipip`先加载；如果一切正常则应该显示：

```
# lsmod | grep ipip
# modprobe ipip
# lsmod | grep ipip
ipip                   20480  0
tunnel4                16384  1 ipip
ip_tunnel              24576  1 ipip
```

现在就可开始搭建IPIP隧道了，我们的网络拓扑如下图所示：

![network-ipip-1.jpg](https://i.loli.net/2020/01/29/bexu9Qt26K7FiND.jpg)

其中有两台主机A和B，每台主机都有一个网卡eth1，并且在同一个网段`172.16.0.0/16`，因此可以直接联通。我们需要做的是分别在两台主机上创建两个不同的子网：

```
A: 10.42.1.0/24
B: 10.42.2.0/24
```

并且设置对应的网关地址分别为我们即将创建的TUN网络设备：

A: 
```
# ip tunnel add mytun1 mode ipip remote 172.16.232.194 local 172.16.232.172
# ip addr add 10.42.1.1/24 dev mytun1
# ip link set mytun1 up
```

上面的命令我们创建了新的隧道设备`mytun1`并且设置了隧道的`remote`和`local`的IP地址，这是IPIP数据包的外层地址；对于内层地址，我们分别设置两个子网地址，这样，IPIP数据包会看起来如下如所示：

![network-ipip-2.jpg](https://i.loli.net/2020/01/29/95RPciVgDqtSmNb.jpg)


B:

```
# ip tunnel add mytun2 mode ipip remote 172.16.232.172 local 172.16.232.194
# ip addr add 10.42.2.1/24 dev mytun2
# ip link set mytun2 up
```

为了保证我们通过创建的IPIP隧道来访问两个不同主机上的子网，我们需要手动添加如下静态路由：

A: 

```
# ip route add 10.42.2.0/24 dev mytun1
```

B:

```
# ip route add 10.42.1.0/24 dev mytun2
```

现在主机AB的路由表如下所示：

A:

```
# ip route show
default via 172.16.200.51 dev ens3
10.42.1.0/24 dev mytun1 proto kernel scope link src 10.42.1.1
10.42.2.0/24 dev mytun1 scope link
172.16.0.0/16 dev ens3 proto kernel scope link src 172.16.232.172
```

B:

```
# ip route show
default via 172.16.200.51 dev ens3
10.42.1.0/24 dev mytun2 scope link
10.42.2.0/24 dev mytun2 proto kernel scope link src 10.42.2.1
172.16.0.0/16 dev ens3 proto kernel scope link src 172.16.232.194
```

到此我们就可以开始验证IPIP隧道是否正常工作：

A:

```
# ping 10.42.2.1 -c 2
PING 10.42.2.1 (10.42.2.1) 56(84) bytes of data.
64 bytes from 10.42.2.1: icmp_seq=1 ttl=64 time=0.269 ms
64 bytes from 10.42.2.1: icmp_seq=2 ttl=64 time=0.303 ms

--- 10.42.2.1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1013ms
rtt min/avg/max/mdev = 0.269/0.286/0.303/0.017 ms
```

B:

```
# ping 10.42.1.1 -c 2
PING 10.42.1.1 (10.42.1.1) 56(84) bytes of data.
64 bytes from 10.42.1.1: icmp_seq=1 ttl=64 time=0.214 ms
64 bytes from 10.42.1.1: icmp_seq=2 ttl=64 time=3.27 ms

--- 10.42.1.1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1021ms
rtt min/avg/max/mdev = 0.214/1.745/3.277/1.532 ms
```

是的，可以ping通，我们通过tcpdump在TUN设备抓取数据：

```
# tcpdump -n -i mytun2
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on mytun2, link-type RAW (Raw IP), capture size 262144 bytes
01:32:05.486835 IP 10.42.1.1 > 10.42.2.1: ICMP echo request, id 3460, seq 1, length 64
01:32:05.486868 IP 10.42.2.1 > 10.42.1.1: ICMP echo reply, id 3460, seq 1, length 64
01:32:06.509617 IP 10.42.1.1 > 10.42.2.1: ICMP echo request, id 3460, seq 2, length 64
01:32:06.509668 IP 10.42.2.1 > 10.42.1.1: ICMP echo reply, id 3460, seq 2, length 64
```

到此为止，我们的实验是成功的。但是需要注意的是，如果我们使用的是`gre`
模式，有可能需要设置防火墙才能让两个子网互通，这种情况在搭建IPv6隧道较为常见。
