---
title: "从零开始认识iptables"
date: 2017-05-20
categories: ['note', 'tech']
draft: false
---

在使用Linux的过程中，很多人和我一样经常接触iptables，但却只知道它是用来设置Linux防火墙的工具，不知道它具体是怎么工作的。今天，我们就从零开始认识一下Linux下iptables的具体工作原理。

iptables是Linux上常用的防火墙软件netfilter项目的一部分，所以要讲清楚iptables，我们先理一理什么是防火墙？

### 什么是防火墙

简单来说，防火墙是一种网络隔离工具，部署于主机或者网络的边缘，目标是对于进出主机或者本地网络的网络报文根据事先定义好的规则做匹配检测，规则匹配成功则对相应的网络报文做定义好的处理（允许，拒绝，转发，丢弃等）。防火墙根据其管理的范围来分可以将其划分为**主机防火墙**和**网络防火墙**；根据其工作机制来区分又可分为**包过滤型防火墙（netfilter）**和**代理服务器（Proxy）**。我们接下来在这篇笔记中主要说说**包过滤型防火墙（netfilter）。

> Note: 也有人将tcp_warrpers也划分为防火墙的一种，它是根据服务程序软件的名称来处理网络数据包的工具。

### 包过滤型防火墙的工作原理

包过滤型防火墙主要依赖于Linux内核软件netfilter，它是一个Linux内核“安全框架”，而iptables是内核软件netfilter的配置工具，工作于用户空间。iptables/netfilter组合就是Linux平台下的过滤型防火墙，并且这个防火墙软件是免费的，可以用来替代商业防火墙软件，来完成网络数据包的过滤，修改，重定向以及网络地址转换（nat）等功能。

> Note: 在有些Linux发行版上，我们可以使用`systemctl start iptables`来启动iptables服务，但需要指出的是，iptables 并不是也不依赖于守护进程，它只是利用Linux内核提供的功能。


Linux网络管理员通过配置iptables规则以及对应的网路数据包处理逻辑，当网络数据包符合这样的规则时，就做执行预先定义好的相应的处理逻辑。可以简单的总结为：

```
IF network_pkg match rule; THEN
    handler
FI
```

其中规则可以包括匹配数据报文的源地址，目的地址，传输层协议（TCP/UDP/ICMP/..）以及应用层协议（HTTP/FTP/SMTP/..）等，处理逻辑就是根据规则所定义的方法来处理这些数据包，如放行（accept），拒绝（reject），丢弃（drop）等。

而netfilter是工作于内核空间当中的一系列网络（TCP/IP）协议栈的钩子（hook），为内核模块在网络协议栈中的不同位置注册回调函数（callback）。也就是说，在数据包经过网络协议栈的不同位置时做相应的由iptables配置好的处理逻辑。
netfilter中的五个钩子（这里也称为五个关卡）PRE_ROUTING，INPUT，FORWARD，OUTPUT，POST_ROUTING，网络数据包的流向图如下图所示：

![](https://i.loli.net/2019/06/02/5cf36899c47eb56912.png)

1. 当主机/网络服务器网卡收到一个数据包之后进入内核空间的TCP/IP协议栈进行层层解封装
2. 刚刚进入网络层的数据包通过PRE_ROUTING关卡时，要进行一次路由选择，当目标地址为本机地址时，数据进入INPUT，非本地的目标地址进入FORWARD（需要本机内核支持IP_FORWARD），所以目标地址转换通常在这个关卡进行
3. INPUT：经过路由之后送往本地的数据包经过此关卡，所以过滤INPUT包在此点关卡进行
4. FORWARD：经过路由选择之后要转发的数据包经过此关卡，所以网络防火墙通常在此关卡配置
5. OUTPUT：由本地用户空间应用进程产生的数据包过此关卡，所以OUTPUT包过滤在此关卡进行
6. POST_ROUTING：刚刚通过FORWARD和OUTPUT关卡的数据包要通过一次路由选择由哪个接口送往网络中，经过路由之后的数据包要通过POST_ROUTING此关卡，源地址转换通常在此点进行

上面提到的这些处于网络（TCP/IP）协议栈的“关卡”，在iptables的术语里叫做“链（chain）”，内置的链包括上面提到的5个：

1. PreRouting
2. Forward
3. Input
4. Output
5. PostRouting

一般的场景里面，数据包的流向基本是：

- 到本主机某进程的报文：PreRouting -> Input -> Process -> Output -> PostRouting
- 由本主机转发的报文：PreRouting -> Forward -> PostRouting

### iptables的四表五链

iptables默认有五条链（chain），分别对应上面提到的五个关卡，PRE_ROUTING，INPUT，FORWARD，OUTPUT，POST_ROUTING，这五个关卡分别由netfilter的五个钩子函数来触发。但是，为什么叫做“链”呢？
我们知道，iptables/netfilter防火墙对经过的数据包进行“规则”匹配，然后执行相应的“处理”。当报文经过某一个关卡时，这个关卡上的“规则”不止一条，很多条规则会按照顺序逐条匹配，将在此关卡的所有规则组织称“链”就很适合，对于经过的数据包按照顺序逐条匹配“规则”。

![](https://i.loli.net/2019/06/03/5cf4c0e47307857800.png)

另外一个问题是，每一条“链”上的一串规则里面有些功能是相似的，比如，A类规则都是对IP或者端口进行过滤，B类规则都是修改报文，我们考虑能否将这些功能相似的规则放到一起，这样管理iptables规则会更方便。iptables把具有相同功能的规则集合叫做“表”，并且定一个四种表：

1. filter：负责过滤功能；与之对应的内核模块是iptables_filter
2. nat：Network Address Translation，网络地址转换功能，典型的比如SNAT，DNAT；与之对应的内核模块是iptables_nat
3. mangle：解包报文，修改并封包；与之对应的内核模块是iptables_mangle
4. raw：关闭nat表上启用的连接追踪机制；与之对应的内核模块是iptables_raw

这样，Linux网络管理员所定义的iptables“规则”都存在于这四张表中。

但是，需要注意的是，并不是所有的“链”都具有所有类型的“规则”，也就是说，某个特定表中的“规则”注定不能应用到某些“链”中，比如，用作地址转换功能的nat表里面的“规则”据不能存在于FORWARD“链”中。下面，我们就详细的列举iptables的“链表”关系：

我们先说一下，每个“链”（关卡）都拥有那些功能的规则：

| 链 | 表 |
| ---------- | ----------- |
| PreRouting | raw, mangle, nat |
| Forward | mangle, filter |
| Input | mangle, filter, nat|
| Output | raw, mangle, filter, nat |
| PostRouting | mangle, nat |

在实际使用iptables配置规则时，我们往往是以“表”为入口制定“规则”，所以我们将“链表”关系转化成“表链”关系：

| 表 | 链 |
| ---------- | ----------- |
| raw | PreRouting, Output |
| mangle | PreRouting, Forward, Input, Output, PostRouting |
| filter | Forward, Input, Output |
| nat | PreRouting, Input, Output, PostRouting |


还需要注意的一点儿是，因为数据包经过一个关卡的时候，会将“链”中所有的“规则”都按照顺序逐条匹配，为相同功能的“规则”属于同一个“表”。那么，哪些“表”中的规则会放到“链”的最前面执行呢？这时候就涉及一个优先级的问题。
iptables为我们提供了四张“表”，当它们处于同一条“链”的时候，它们的执行优先级关系如下：

```
raw -> mangle -> nat -> filter
```

实际上，网络管理员还可以使用iptables创建自定义的链，将针对某个应用层序所设置的规则放到这个自定义链中，但是自定义的链不能直接使用，只能被某个默认的链当作Action去调用。可以这样说，自定义链是“短”链，这些“短”链并不能直接使用，而是需要和iptables上的内置链一起配合，被内置链“引用”。

### 数据包经过过滤型防火墙的流程

有了前面的介绍我们可以总结出如下图所示的数据包在经过过滤型防火墙的流程图：

![](https://i.loli.net/2019/06/02/5cf388043e45962099.png)

### iptables规则

前面我们提到过，iptables规则由两部分组成，报文的匹配条件和匹配到之后的处理动作。

1. 匹配条件：根据协议报文特征指定匹配条件，基本匹配条件和扩展匹配条件
2. 处理动作：内建处理机制由iptables自身提供的一些处理动作

同时，网络管理员还可以使用iptables创建自定义的链，附加到iptables的四个内置链。

> Note: 报文不会经过自定义链，只能在内置链上通过规则进行引用后生效，也就是说自定义链为规则的一个处理动作的集合。

设置iptables规则时需要考量的要点：

1. 根据要实现哪种功能，判断添加在那张“表”上
2. 根据报文流经的路径，判断添加在那个“链”上
    - 到本主机某进程的报文：PreRouting -> Input -> Process -> Output -> PostRouting
    - 由本主机转发的报文：PreRouting -> Forward -> PostRouting

对于每一条“链”上其“规则”的匹配顺序，排列好检查顺序能有效的提高性能，因此隐含一定的法则：

1. 同类规则（访问同一应用），匹配范围小的放上面
2. 不同类规则（访问不同应用），匹配到报文频率大的放上面
3. 将那些可由一条规则描述的多个规则合并为一个
4. 设置默认策略

同时，也一定要注意，在远程连接主机配置防火墙时注意：

1. 不要把“链”的默认策略修改为拒绝，因为有可能配置失败或者清除所有策略后无法远程到服务器，而是尽量使用规则条目配置默认策略
2. 为防止配置失误策略把自己也拒掉，可在配置策略时设置计划任务定时清除策略，当确定无误后，关闭该计划任务

### iptables语法


iptables语法结构如下所示：

```
iptables(选项)(参数)
```

**选项：**

```
# 通用匹配：源地址目标地址的匹配
-p：指定要匹配的数据包协议类型；
-s, --source [!] address[/mask] ：把指定的一个／一组地址作为源地址，按此规则进行过滤。当后面没有 mask 时，address 是一个地址，比如：192.168.1.1；当 mask 指定时，可以表示一组范围内的地址，比如：192.168.1.0/255.255.255.0。
-d, --destination [!] address[/mask] ：地址格式同上，但这里是指定地址为目的地址，按此进行过滤。
-i, --in-interface [!] <网络接口name> ：指定数据包的来自来自网络接口，比如最常见的 eth0 。注意：它只对 INPUT，FORWARD，PREROUTING 这三个链起作用。如果没有指定此选项， 说明可以来自任何一个网络接口。同前面类似，"!" 表示取反。
-o, --out-interface [!] <网络接口name> ：指定数据包出去的网络接口。只对 OUTPUT，FORWARD，POSTROUTING 三个链起作用。

# 查看管理命令
-L, --list [chain] 列出链 chain 上面的所有规则，如果没有指定链，列出表上所有链的所有规则。

# 规则管理命令
-A, --append chain rule-specification 在指定链 chain 的末尾插入指定的规则，也就是说，这条规则会被放到最后，最后才会被执行。规则是由后面的匹配来指定。
-I, --insert chain [rulenum] rule-specification 在链 chain 中的指定位置插入一条或多条规则。如果指定的规则号是1，则在链的头部插入。这也是默认的情况，如果没有指定规则号。
-D, --delete chain rule-specification -D, --delete chain rulenum 在指定的链 chain 中删除一个或多个指定规则。
-R num：Replays替换/修改第几条规则

# 链管理命令（这都是立即生效的）
-P, --policy chain target ：为指定的链 chain 设置策略 target。注意，只有内置的链才允许有策略，用户自定义的是不允许的。
-F, --flush [chain] 清空指定链 chain 上面的所有规则。如果没有指定链，清空该表上所有链的所有规则。
-N, --new-chain chain 用指定的名字创建一个新的链。
-X, --delete-chain [chain] ：删除指定的链，这个链必须没有被其它任何规则引用，而且这条上必须没有任何规则。如果没有指定链名，则会删除该表中所有非内置的链。
-E, --rename-chain old-chain new-chain ：用指定的新名字去重命名指定的链。这并不会对链内部照成任何影响。
-Z, --zero [chain] ：把指定链，或者表中的所有链上的所有计数器清零。

-j, --jump target <指定目标> ：即满足某条件时该执行什么样的动作。target 可以是内置的目标，比如 ACCEPT，也可以是用户自定义的链。
-h：显示帮助信息；
```

**参数**

| 参数 | 作用 |
| ----- | ----- |
| -P | 设置默认策略:iptables -P INPUT (DROP) |
| -F | 清空规则链 |
| -L | 查看规则链 |
| -A | 在规则链的末尾加入新规则 |
| -I | num 在规则链的头部加入新规则 |
| -D | num 删除某一条规则 |
| -s | 匹配来源地址IP/MASK，加叹号"!"表示除这个IP外 |
| -d | 匹配目标地址 |
| -i | 网卡名称 匹配从这块网卡流入的数据 |
| -o | 网卡名称 匹配从这块网卡流出的数据 |
| -p | 匹配协议,如tcp,udp,icmp |
| --dport num | 匹配目标端口号 |
| --sport num | 匹配来源端口号 |

**命令选项输入顺序：**

```
iptables -t 表名 <-A/I/D/R> 规则链名 [规则号] <-i/o 网卡名> -p 协议名 <-s 源IP/源子网> --sport 源端口 <-d 目标IP/目标子网> --dport 目标端口 -j 动作
```

对于iptables命令的细节，请参考：https://wangchujiang.com/linux-command/c/iptables.html
