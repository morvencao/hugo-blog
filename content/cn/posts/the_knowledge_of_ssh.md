---
title: "SSH协议以及端口转发"
date: 2015-03-19
categories: ['note', 'tech']
draft: false
---

[SSH](https://en.wikipedia.org/wiki/Secure_Shell)估计是每台Linux机器的标配了。日程工作中程序员在本机写完Code之后，很少在本机上直接部署测试，经常需要通过SSH协议登录到实验室的Linux主机上验证。实际上SSH具备多种功能，不仅仅是远程登录这么简单，今天我们就详细探讨一下SSH协议以及它的高级功能-端口转发。

## SSH的原理

SSH是一种网络协议，用于计算机之间的加密登录，也就是说这种登陆是安全的。SSH协议之所以安全，是因为它基于非对称加密。基本的过程可以描述为：

1. 客户端通过`SSH user@remote-host`发起登录远程主机的请求
2. 远程主机收到请求之后，将自己的公钥发给客户端
3. 客户端使用这个公钥加密登录密码之后发给远程主机
4. 远程主机使用自己的私钥，解密登陆请求，获得登录密码，如果正确，则建立SSH通道，之后所有客户端和远程主机的数据传输都要加密发送

看似很完美是吗？其实有个问题，如果有人中途拦截了登录请求，将自己伪造的公钥发送给客户端，客户端其实并不能识别这个公钥的可靠性，因为SSH协议并不像HTTPS协议那样，公钥是包含在证书中心CA来颁发的证书之中。所以有攻击者（中间人攻击-Man-in-the-middle-attack）拦截客户端到远程主机的登陆请求，伪造公钥来获取远程主机的登录密码，SSH协议的安全机制就荡然无存了。

说实话，SSH协议本身确实无法阻止这种攻击形式，最终还是依靠终端用户自身来识别并规避风险。

比如，我们在第一次登录某一台远程主机的时候，会得到如下提示：

```
$ ssh user@remote-host
The authenticity of host '10.30.110.230 (10.30.110.230)' can't be established.
ECDSA key fingerprint is SHA256:RIXlybA1rgf4mbnWvLuOMAxGRQQFgfnM2+YbYLT7aQA.
Are you sure you want to continue connecting (yes/no)?
```

这个提示的意思是说无法确定远程主机的真实性，只能得到它的指纹(fingerprint)，需要你确认是否信任这个返回的公钥。这里所说的"指纹"是RSA公钥的MD5哈希结果。我们知道为了保证RSA私钥的安全性，一般RSA公钥设置基本都不小于1024位，很难直接让终端用户去确认完整的RSA公钥，于是通过MD5哈希函数将之转化为128位的指纹，就很容易识别了。实际上，有很多网络应用程序都是用RSA公钥指纹来让终端用户识别公钥的可靠性。

具体怎么决定是依赖于终端用户了，所以推荐的做法是将远程主机的公钥保存在合适的地方方便核对。这是，如果用户决定接受这个返回的公钥，系统会继续提示：

```
Warning: Permanently added '10.30.110.230' (RSA) to the list of known hosts.
```

紧接着就是用户输入密码来登录远程主机。同时远程主机的公钥会被保存在`$HOME/.ssh/known_hosts`这个文件中，下次登录的时候就不需要用户再次核对指纹了。每个SSH用户都有自己的`known_hosts`文件，此外系统也有一个这样的文件，通常是`/etc/ssh/ssh_known_hosts`，保存一些对所有用户都可信赖的远程主机的公钥。

除了使用密码登录，SSH协议还支持公钥登录。这里的公钥不再是远程主机的公钥，而是客户端的RSA公钥。原理很简单：

1. 用户将客户端的RSA公钥保存在远程主机上
2. 客户端向远程主机发起登录请求
3. 远程主机会向客户端发送一段随机字符串
4. 客户端用自己的RSA私钥加密后再发送给远程主机
5. 远程主机使用保存的的RSA公钥进行解密，如果解密成功则代表客户端是可信的，不需要密码就能确认身份

在客户端生成RSA公私钥对，一般使用`ssh-keygen`。举例来说，我们要生成`2048`位RSA密钥对：

```
$ ssh-keygen -b 2048 -t rsa -f foo_rsa
Generating public/private rsa key pair.
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in foo_rsa.
Your public key has been saved in foo_rsa.pub.
The key fingerprint is:
b8:c4:5f:2a:94:fd:b9:56:9d:5b:fd:96:02:5a:7e:b7 user@oasis
The key's randomart image is:
+--[ RSA 2048]----+
|                 |
|                 |
|                 |
|     . +         |
|      * S .  . ..|
|     o o + +. o o|
|      o o *..  oo|
|       . ..o o.oo|
|         .. . oE.|
+-----------------+
```

- `-b`指定密钥位数
- `-t`指定密钥类型：`rsa`，`dsa`, `ecdsa`

> Note: 如果密钥对用于`ssh`，那么习惯上私钥命名为`id_rsa`，对应的公钥就是`id_rsa.pub`，然后可以使用`ssh-copy-id`把密钥追加到远程主机的`.ssh/authorized_key`文件里：

```
$ ssh-copy-id user@remote-host
```

Note: 有时候执行完上述命令之后，客户端还是不能免密登陆远程主机，很有可能是远程主机禁用了RSA公钥登录，请检查远程主机的`/etc/ssh/sshd_config`文件，确认下面几行前面"#"注释是否取掉：

```
RSAAuthentication yes
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys
```

保存之后需要重启SSH服务。

## 端口转发

通过前面的介绍，我们知道SSH会自动加密和解密客户端和远程主机之间的网络通信数据，这个功能有时候非常有用，其他TCP网络应用程序可以通过SSH来建立一条加密通道来传输网录数据，这就是SSH端口转发的作用。事实上，譬如Telnet，SMTP，LDAP之类的TCP应用均能够从中获益。不只是网络数据的加密传输，如果你工作的环境中的防火墙限制了一些网络端口的使用，但是允许SSH的连接，那么也是能够通过将TCP端口转发来使用SSH进行通信。

<img src="https://i.loli.net/2019/03/24/5c97970c975a7.jpg" style="width:60%;"/>


如上图所示，使用了端口转发之后，TCP端口A与B之间不是直接通信，而是转发到了SSH客户端及服务端来通信，从而自动实现了数据加密并同时绕过了防火墙的限制。

总的来说，端口转发可以分为本地转发，远程转发以及动态转发等

### 本地转发

SSH本地转发命令的基本格式是这样的：

```
ssh -L <local port>:<remote host>:<remote port> <SSH hostname>
```

举个例子，在实验室里有一台LDAP服务器（LdapServerHost），但是限制了只有本机上部署的应用才能直接连接此LDAP服务器。如果我们由于调试或者测试的需要想临时从远程机器（LdapClientHost）直接连接到这个LDAP服务器，有什么方法能够实现呢？

使用本地转发的作用，我们可以远程登录主机上执行如下命令：

```
$ ssh -L 7001:localhost:389 LdapServerHost
```

<img src="https://i.loli.net/2019/03/24/5c97992b19abf.jpg" style="width:60%;"/>

> Note: 本例中我们选择了`7001`端口作为本地的监听端口，在选择端口号时要注意非管理员帐号是无权绑定`1-1023`端口的，所以最好选择一个`1024-65535`之间的未占用的端口。

然后我们可以将远程机器（LdapClientHost）上的应用直接配置到本机的`7001`端口上（而不是LDAP服务器的`389`端口上）。之后的数据流将会是下面这个样子：

1. 在LdapClientHost上的应用将数据发送到本机的`7001`端口
2. 本机的SSH Client将`7001`端口收到的数据加密并转发到LdapServertHost的SSH Server
3. SSH Server会解密收到的数据并将之转发到监听的LDAP `389`端口
4. 最后再将从LDAP返回的数据原路返回以完成整个流程

可以看到，这整个流程应用并没有直接连接LDAP服务器，而是连接到了本地的一个监听端口，SSH端口转发完成了剩下的所有事情，加密，转发，解密...

这里有几个地方需要注意：

- SSH端口转发需要保持SSH连接不断开，一旦关闭了此连接，相应的端口转发也会随之关闭。保持SSH连接可以在执行SSH命令的之后加`-f`来保持SSH连接并转入后台
- 上述命令中的`<remote host>`为什么是`localhost`，它指向的是哪台机器的回环接口呢？在本例中，它指向LdapServertHost。为什么用`localhost`而不是IP地址或者主机名呢？其实这个取决于LDAP服务器的设置了，如果LDAP服务器限制只有本机`lookback`接口访问的话，那么自然就只有`localhost或者IP为`127.0.0.1`才能访问LDAP服务器，而不能用真实IP或者主机名
- 命令中的`<remote host>`和`<SSH hostname>`必须是同一台机器么？其实是不一定的，它们可以是两台不同的机器。后面的例子里会详细阐述这点。
- 执行上述命令后会在LdapClientHost建立端口转发，那么这个端口转发可以被其他机器使用么？比如能否新增加一台LdapClientHost2来直接连接LdapClientHost的`7001`端口？答案是不行的，在主流SSH实现中，本地端口转发绑定的是`lookback`接口，这意味着只有`localhost`或者`127.0.0.1`才能使用本机的端口转发，其他机器发起的连接只会得到"connection refused"。好在SSH同时提供了`GatewayPorts关键字，可以通过指定它与其他机器共享这个本地端口转发：

```
$ ssh -g -L <local port>:<remote host>:<remote port> <SSH hostname>
```

### 远程转发

SSH远程转发命令的基本格式是这样的：

```
ssh -R <local port>:<remote host>:<remote port> <SSH hostname>
```

举例来说，假设由于网络或防火墙的原因我们不能用SSH直接从LdapClientHost连接到LDAP服务器（LdapServertHost），但是反向连接却是被允许的，那此时就可以通过远程端口转发来实现对LADP服务器的访问。在LDAP服务器（LdapServertHost）主机上执行如下命令：

```
$ ssh -R 7001:localhost:389 LdapClientHost
```

<img src="https://i.loli.net/2019/03/24/5c979dce7810d.jpg" style="width:60%;"/>

可以看到，和本地端口转发相比，这次只是SSH Server和SSH Client的位置对调了一下，但是数据流传输路径还是一模一样：LdapClientHost上的应用将数据发送到本机的`7001`端口上，而本机的SSH Server会将`7001`端口收到的数据加密并通过SSH隧道转发到LdapServertHost的SSH Client，SSH Client会解密收到的数据并将之转发到监听的LDAP `389`端口，最后再将从LDAP返回的数据原路返回以完成整个流程。

弄清楚了本地转发和远程转发，我们现在来看看前面遗留的一个问题：本地转发命令中的`<remote host>`和`<SSH hostname>`可以是不同的机器么?

```
ssh -L <local port>:<remote host>:<remote port> <SSH hostname>
```

答案是可以的！举例来说，我们有四台机器(HostA, HostB, HostC, HostD)，其中HostA想访问HostHostD上面的LDAP服务，但是由于网络限制，HostA不能直接访问HostD，但是HostA可以访问HostB，HostB也能SSH到HostC，HostC能直连HostD，如何通过SSH端口转发来让HostA访问HostD的LDAP服务呢？我们只需要在HostB上执行本地端口转发：

```
$ ssh -g -L 7001:<D>:389 <C>
```

<img src="https://i.loli.net/2019/03/24/5c97a29669643.jpg" style="width:60%;"/>

这样，应用客户端HostA要访问HostD的LDAP服务，只需要访问HostB的`7001`端口即可。注意我们在命令中指定了`-g`参数以保证HostA能够使用HostB建立的本地端口转发。

### 动态转发

经过前面的介绍，我们发现本地转发和远程转发前提都是要求有一个固定的应用服务端的端口号，例如前面例子中的LDAP服务的`389`端口。相对于本地转发和远程转发的单一端口转发模式而言，动态转发有点更加强大的端口转发功能，无需指定被访问目标主机的端口。这个端口号需要在本地通过协议指定，该协议就是简单、安全、实用的SOCKS协议。

动态转发的命令格式：

```
ssh -D <local port> <SSH Server>
```

例如，当我们本机上执行下面的命令之后，SSH会帮我们创建一个SOCKS代理服务器，所有发送到本机`8888`端口的数据都会转发到远程主机。接下来，我们可以直接在浏览器中设置SOCKS代理：`localhost:8888`，这样，在本机上无法端无法访问的网站现在也就可以通过远程主机代理来正常访问。

```
$ ssh -D 8888 user@remote-host
```

<img src="https://i.loli.net/2019/03/24/5c97a65adcb61.jpg" style="width:60%;"/>


SSH还有一些别的参数，也值得了解一下。

- `-N`参数，表示只连接远程主机，不打开远程shell
- `-T`参数，表示不为这个连接分配TTY

这个两个参数可以放在一起用，代表这个SSH连接只用来传数据，不执行远程操作。

```
$ ssh -NT -D 8888 user@remote-host
```

- `-f`参数，表示SSH连接成功后，转入后台运行。这样就可以在不中断SSH连接的情况下，在本地shell中执行其他操作，要关闭这个后台连接，就只有用kill命令去杀掉进程。

另外，SSH还支持其他形式的端口转发，比如X协议的图形界面应用端口转发，这里就不详细说明了。
