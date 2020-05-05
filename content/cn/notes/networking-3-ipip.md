---
title: "揭秘IPIP隧道"
date: 2020-02-12
type: "notes"
draft: false
---

上一篇笔记中，我们在介绍网络设备的时候了解了一种典型的通过TUN/TAP设备来实现VPN的原理，但是并没有实践TUN/TAP虚拟网络设备具体在linux中怎么发挥实际的功能。这篇笔记我们就来看看在云计算领域中一种非常典型的IPIP隧道如何TUN设备来实现。

## IPIP隧道

上一篇笔记中我们也提到了，TUN网络设备能将三层（IP）网络包封装在另外一个三层网络包之中，看起来通过TUN设备发送出来的数据包会像会这样：

```
MAC: xx:xx:xx:xx:xx:xx
IP Header: <new destination IP>
IP Body:
  IP: <original destination IP>
  TCP: stuff
  HTTP: stuff
```

这就是典型的IPIP隧道数据包的结构。Linux原生支持好几种不同的IPIP隧道类型，但都依赖于TUN网络设备，我们可以通过命令`ip tunnel help`来查看IPIP隧道的相关类型以及操作：

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

1. ipip: 普通的IPIP隧道，就是在报文的基础上再封装成一个IPv4报文
2. gre: 通用路由封装（Generic Routing Encapsulation），定义了在任意一种网络层协议上封装其他任意一种网络层协议的机制，所以对于IPv4和IPv6都适用
3. sit: sit模式主要用于IPv4报文封装IPv6报文，即IPv6 over IPv4
4. isatap: 站内自动隧道寻址协议（Intra-Site Automatic Tunnel Addressing Protocol），类似于sit也是用于IPv6的隧道封装
5. vti: 即虚拟隧道接口（Virtual Tunnel Interface），是一种IPsec隧道技术

还有一些有用的参数：
- ttl N 设置进入隧道数据包的TTL为N（N是一个1—255之间的数字，0是一个特殊的值，表示这个数据包的TTL值是继承(inherit)的），ttl参数的缺省值是为inherit
- tos T/dsfield T 设置进入通道数据包的TOS域，缺省是inherit
- [no]pmtudisc 在这个隧道上禁止或者打开路径最大传输单元发现(Path MTU Discovery)，默认打开的

> Note: nopmtudisc选项和固定的ttl是不兼容的，如果使用了固定的ttl参数，系统会打开路径最大传输单元发现( Path MTU Discovery)功能

## one-to-one

我们首先以最基本的one-to-one的IPIP隧道模式为例来介绍如何在linux中搭建IPIP隧道来实现两个不同子网之间的通信。

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

其中有处于在同一个网段`172.16.0.0/16`的两台主机A和B，因此可以直接联通。我们需要做的是分别在两台主机上创建两个不同的子网：

> Note: 实际上，这两台主机A和B不必要处于同一个子网，只要处于同一个三层网络之中，也就是说能通过三层网络路由得到就可以完成IPIP隧道的搭建。

```
A: 10.42.1.0/24
B: 10.42.2.0/24
```

为了简化，我们先在A节点上创建bridge网络设备`mybr0`，并且设置IP地址为`10.42.1.0/24`子网的网关地址，然后启用`mybr0`：
```
# ip link add name mybr0 type bridge
# ip addr add 10.42.1.1/24 dev mybr0
# ip link set dev mybr0 up
```

类似地，然后在B节点上分别执行类似的操作：

B:

```
# ip link add name mybr0 type bridge
# ip addr add 10.42.2.1/24 dev mybr0
# ip link set dev mybr0 up
```

接下来，我们分别在A和B两台节点上

1. 创建对应的TUN网络设备
2. 设置对应的local和remote地址为node节点的可路由地址
3. 设置对应的网关地址分别为我们即将创建的TUN网络设备
4. 启用TUN网络设备来创建IPIP隧道

> Note: 步骤3是为了节省简化我们创建子网的步骤，直接设置网关地址就可以不用创建额外的网络设备。

A: 
```
# modprobe ipip
# ip tunnel add tunl0 mode ipip remote 172.16.232.194 local 172.16.232.172
# ip addr add 10.42.1.1/24 dev tunl0
# ip link set tunl0 up
```

上面的命令我们创建了新的隧道设备`tunl0`并且设置了隧道的`remote`和`local`的IP地址，这是IPIP数据包的外层地址；对于内层地址，我们分别设置两个子网地址，这样，IPIP数据包会看起来如下如所示：

![network-ipip-2.jpg](https://i.loli.net/2020/01/29/95RPciVgDqtSmNb.jpg)

B:

```
# modprobe ipip
# ip tunnel add tunl0 mode ipip remote 172.16.232.172 local 172.16.232.194
# ip addr add 10.42.2.1/24 dev tunl0
# ip link set tunl0 up
```

为了保证我们通过创建的IPIP隧道来访问两个不同主机上的子网，我们需要手动添加如下静态路由：

A: 

```
# ip route add 10.42.2.0/24 dev tunl0
```

B:

```
# ip route add 10.42.1.0/24 dev tunl0
```

现在主机AB的路由表如下所示：

A:

```
# ip route show
default via 172.16.200.51 dev ens3
10.42.1.0/24 dev tunl0 proto kernel scope link src 10.42.1.1
10.42.2.0/24 dev tunl0 scope link
172.16.0.0/16 dev ens3 proto kernel scope link src 172.16.232.172
```

B:

```
# ip route show
default via 172.16.200.51 dev ens3
10.42.1.0/24 dev tunl0 scope link
10.42.2.0/24 dev tunl0 proto kernel scope link src 10.42.2.1
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
# tcpdump -n -i tunl0
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on tunl0, link-type RAW (Raw IP), capture size 262144 bytes
01:32:05.486835 IP 10.42.1.1 > 10.42.2.1: ICMP echo request, id 3460, seq 1, length 64
01:32:05.486868 IP 10.42.2.1 > 10.42.1.1: ICMP echo reply, id 3460, seq 1, length 64
01:32:06.509617 IP 10.42.1.1 > 10.42.2.1: ICMP echo request, id 3460, seq 2, length 64
01:32:06.509668 IP 10.42.2.1 > 10.42.1.1: ICMP echo reply, id 3460, seq 2, length 64
```

到此为止，我们的实验是成功的。但是需要注意的是，如果我们使用的是`gre`模式，有可能需要设置防火墙才能让两个子网互通，这种情况在搭建IPv6隧道较为常见。

### one-to-many

上一节中我们通过指定TUN设备的local地址和remote地址创建了一个one-to-one的IPIP隧道，实际上，在创建IPIP隧道的时候完全可以不指定remote地址，只要在TUN设备上增加对应的路由，IPIP隧道就知道如何封装新的IP数据包并发送到路由指定的目标地址。

还是举个栗子来说明，假设我们现在有处于同一个三层网络的3个节点：

```
A: 172.16.165.33
B: 172.16.165.244
C: 172.16.168.113
```

同时在这三个节点上分别attach三个不同的子网：

```
A: 10.42.1.0/24
B: 10.42.2.0/24
C: 10.42.3.0/24
```

与上一小节不同的是，我们没有直接将子网的网关地址设置为TUN设备的IP地址，而是创建额外的bridge网络设备以模拟实际常用的容器网络模型。我们在A节点上创建bridge网络设备`mybr0`，并且设置IP地址为`10.42.1.0/24`子网的网关地址，然后启用`mybr0`：

```
# ip link add name mybr0 type bridge
# ip addr add 10.42.1.1/24 dev mybr0
# ip link set dev mybr0 up
```

类似地，然后在B和C节点上分别执行类似的操作：

B:

```
# ip link add name mybr0 type bridge
# ip addr add 10.42.2.1/24 dev mybr0
# ip link set dev mybr0 up
```

C:

```
# ip link add name mybr0 type bridge
# ip addr add 10.42.3.1/24 dev mybr0
# ip link set dev mybr0 up
```

我们的最终目标是在三个节点之间分别俩俩搭建IPIP隧道来保证这三个不同的子网直接能够互相通信，因此下一步是创建TUN网络设备并且设置路由信息。分别在A和B两台节点上：

1. 创建对应的TUN网络设备并启用
2. 设置TUN网络设备的IP地址
3. 设置到不同子网的路由，指明下一跳的地址


对应的网关地址分别为我们即将创建的TUN网络设备
4. 启用TUN网络设备来创建IPIP隧道

> Note: TUN网络设备的IP地址是对应节点的子网地址，但是子网掩码是32位的，例如A节点上子网地址是`10.42.1.0/24`，A节点上的TUN网络设备的IP地址是`10.42.1.0/32`。这样做的原因是有时候同一个子网(例如`10.42.1.0/24`)的地址会分配相同的MAC地址，因此不能通过二层的链路层直接通信，而如果保证TUN网络设备的IP地址和任何地址都不在同一个子网，也就不存在二层的链路层直接通信了。关于这点请参考calico的实现原理，每个容器会有相同的MAC地址，后面我们有机会在深入探究。

> Note: 还有一点需要注意，给TUN网络设备设置路由的时候指定了`onlink`, 这样做的目的是保证下一跳是直接attach到该TUN网络设备的，这样保证即使节点之间不在同一个子网中也可以搭建IPIP隧道。

A:
```
# modprobe ipip
# ip tunnel add tunl0 mode ipip
# ip link set tunl0 up
# ip addr add 10.42.1.0/32 dev tunl0
# ip route add 10.42.2.0/24 via 172.16.165.244 dev tunl0 onlink
# ip route add 10.42.3.0/24 via 172.16.168.113 dev tunl0 onlink
```

B:
```
# modprobe ipip
# ip tunnel add tunl0 mode ipip
# ip link set tunl0 up
# ip addr add 10.42.2.0/32 dev tunl0
# ip route add 10.42.1.0/24 via 172.16.165.33 dev tunl0 onlink
# ip route add 10.42.3.0/24 via 172.16.168.113 dev tunl0 onlink
```

C:
```
modprobe ipip
ip tunnel add tunl0 mode ipip
ip link set tunl0 up
ip addr add 10.42.3.0/32 dev tunl0
ip route add 10.42.1.0/24 via 172.16.165.33 dev tunl0 onlink
ip route add 10.42.2.0/24 via 172.16.165.244 dev tunl0 onlink
```

到此我们就可以开始验证我们搭建的IPIP隧道是否正常工作：

A:
```
# try to ping IP in 10.42.2.0/24 on Node B
# ping 10.42.2.1 -c 2
PING 10.42.2.1 (10.42.2.1) 56(84) bytes of data.
64 bytes from 10.42.2.1: icmp_seq=1 ttl=64 time=0.338 ms
64 bytes from 10.42.2.1: icmp_seq=2 ttl=64 time=0.302 ms

--- 10.42.2.1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1028ms
rtt min/avg/max/mdev = 0.302/0.320/0.338/0.018 ms
...
# try to ping IP in 10.42.3.0/24 on Node C
# ping 10.42.3.1 -c 2
PING 10.42.3.1 (10.42.3.1) 56(84) bytes of data.
64 bytes from 10.42.3.1: icmp_seq=1 ttl=64 time=0.315 ms
64 bytes from 10.42.3.1: icmp_seq=2 ttl=64 time=0.381 ms

--- 10.42.3.1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1029ms
rtt min/avg/max/mdev = 0.315/0.348/0.381/0.033 ms
```

看起来一切正常，如果反过来从B或者C节点分别ping其他子网，也是可以通的。这就说明我们确实可以创建一对多的IPIP隧道，这中one-to-many的模式在一些典型的多节点网络中创建overlay通信模型中非常有用。

### under the hood

我们再通过tcpdump在分别在B和C的TUN设备抓取数据：

B:

```
# tcpdump -n -i tunl0
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on tunl0, link-type RAW (Raw IP), capture size 262144 bytes
22:38:28.268089 IP 10.42.1.0 > 10.42.2.1: ICMP echo request, id 6026, seq 1, length 64
22:38:28.268125 IP 10.42.2.1 > 10.42.1.0: ICMP echo reply, id 6026, seq 1, length 64
22:38:29.285595 IP 10.42.1.0 > 10.42.2.1: ICMP echo request, id 6026, seq 2, length 64
22:38:29.285629 IP 10.42.2.1 > 10.42.1.0: ICMP echo reply, id 6026, seq 2, length 64
```

C:

```
# tcpdump -n -i tunl0
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on tunl0, link-type RAW (Raw IP), capture size 262144 bytes
22:36:18.236446 IP 10.42.1.0 > 10.42.3.1: ICMP echo request, id 5894, seq 1, length 64
22:36:18.236499 IP 10.42.3.1 > 10.42.1.0: ICMP echo reply, id 5894, seq 1, length 64
22:36:19.265946 IP 10.42.1.0 > 10.42.3.1: ICMP echo request, id 5894, seq 2, length 64
22:36:19.265997 IP 10.42.3.1 > 10.42.1.0: ICMP echo reply, id 5894, seq 2, length 64
```

其实，从创建one-to-many的IPIP隧道的过程中我们就能大致猜到Linux的ipip模块基于路由信息获取IPIP包的内部ip然后再用外部IP封装成新的IP包。至于怎么解封IPIP数据包的呢，我们来看看ipip模块收数据包的过程：

```
void ip_protocol_deliver_rcu(struct net *net, struct sk_buff *skb, int protocol)
{
	const struct net_protocol *ipprot;
	int raw, ret;

resubmit:
	raw = raw_local_deliver(skb, protocol);

	ipprot = rcu_dereference(inet_protos[protocol]);
	if (ipprot) {
		if (!ipprot->no_policy) {
			if (!xfrm4_policy_check(NULL, XFRM_POLICY_IN, skb)) {
				kfree_skb(skb);
				return;
			}
			nf_reset_ct(skb);
		}
		ret = INDIRECT_CALL_2(ipprot->handler, tcp_v4_rcv, udp_rcv,
				      skb);
		if (ret < 0) {
			protocol = -ret;
			goto resubmit;
		}
		__IP_INC_STATS(net, IPSTATS_MIB_INDELIVERS);
	} else {
		if (!raw) {
			if (xfrm4_policy_check(NULL, XFRM_POLICY_IN, skb)) {
				__IP_INC_STATS(net, IPSTATS_MIB_INUNKNOWNPROTOS);
				icmp_send(skb, ICMP_DEST_UNREACH,
					  ICMP_PROT_UNREACH, 0);
			}
			kfree_skb(skb);
		} else {
			__IP_INC_STATS(net, IPSTATS_MIB_INDELIVERS);
			consume_skb(skb);
		}
	}
}
```

From https://github.com/torvalds/linux/blob/master/net/ipv4/ip_input.c#L187-L224

可以看到，ipip模块会根据数据包的协议类型去解封，然后将解封后的skb数据包再做一次解封。以上只是一些非常浅显的分析，如果大家感兴趣，推荐去多看看ipip模块的源代码实现。
