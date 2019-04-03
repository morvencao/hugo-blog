---
title: "Linux的启动流程"
date: 2015-02-10
type: "notes"
draft: false
---

和Window等其他操作系统一样，linux的启动也分为两个阶段：**引导（boot）**和**启动（startup）**。**引导阶段**开始于打开电源开关，接下来板载程序[BIOS](https://en.wikipedia.org/wiki/BIOS)的开始POST上电自检主过程，结束于内核初始化完成。**启动阶段**接管剩余的工作，直到操作系统初始化完成进入可操作状态，并能够执行用户的功能性任务。

我们不花过多篇幅讲解引导阶段的硬件板载程序加载运行的过程。事实上，由于在板载程序上很多行为基本上固定的，程序员很难介入，所以接下来主要讲讲主板的引导程序如何加载内核程序以及控制权交给linux操作系统之后的各个服务的启动流程。

### GRUB引导加载程序

所谓引导程序，一个用于计算机寻找操作系统内核并加载其到内存的智能程序，通常位于硬盘的第一个扇区，并由BIOS载入内存执行。为什么需要引导程序，而不是直接由BIOS加载操作系统？原因是BOIS只会自动加载硬盘的第一个扇区的512字节的内容，而操作系统的大小远远大于这个值，所以才会先加载引导程序，然后通过引导程序加载程序加载操作系统到内存中。

目前，各个Linux发行版主流的引导程序是GRUB(GRand Unified BootLoader)。GRUB的作用有以下几个：
- 加载操作系统的内核
- 拥有一个可以让用户选择到底启动哪个系统的的菜单界面
- 可以调用其他的启动引导程序，来实现多系统引导

GRUB1现在已经逐步被弃用，在大多数现代发行版上它已经被GRUB2所替换。GRUB2通过`/boot/grub2/grub.cfg`进行配置，最终GRUB定位和加载linux内核程序到内存中，并转移控制权到内核程序。

### 内核程序

内核程序的相关文件位于`/boot`目录下，这些内核文件均带有前缀`vmlinuz`。内核文件都是以一种自解压的压缩格式存储以节省空间。在选定的内核加载到内存中并开始执行后，在其进行任何工作之前，内核文件首先必须从压缩格式解压自身。

```
# ll /boot/
total 152404
drwxr-xr-x  4 root root     4096 Nov 29 15:34 ./
drwxr-xr-x 22 root root      335 Jan 16 12:22 ../
-rw-r--r--  1 root root   190587 Aug 10  2018 config-3.2.0-3-amd64
-rw-r--r--  1 root root   190611 Oct  2 12:22 config-3.2.0-4-amd64
drwxr-xr-x  5 root root     4096 Nov 29 15:33 grub/
-rw-r--r--  1 root root 39413114 Nov 29 15:33 initrd.img-3.2.0-3-amd64
-rw-r--r--  1 root root 39423838 Nov 29 15:32 initrd.img-3.2.0-4-amd64
-rw-r--r--  1 root root      255 Oct  2 12:22 retpoline-3.2.0-3-amd64
-rw-r--r--  1 root root      255 Oct 24 08:31 retpoline-3.2.0-4-amd64
-rw-------  1 root root  3902569 Aug 10  2018 System.map-3.2.0-3-amd64
-rw-------  1 root root  3904838 Oct  2 12:22 System.map-3.2.0-4-amd64
-rw-------  1 root root  7159744 Aug 10  2018 vmlinuz-3.2.0-3-amd64
-rw-------  1 root root  7166688 Oct  2 12:22 vmlinuz-3.2.0-4-amd64
```

### Init进程

一旦内核文件自解压完成，linux操作系统开始启动运行第一个程序`/sbin/init`，它的作用是初始化系统环境。由于`init`是第一个运行的程序，它的PID为`1`。其他所有进程都从它衍生，是所有其他进程的祖先。事实上`init`以守护进程方式存在：

```
# ps -ef
UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  0 Mar29 ?        00:00:17 /sbin/init
root         2     0  0 Mar29 ?        00:00:00 [kthreadd]
root         3     2  0 Mar29 ?        00:00:22 [ksoftirqd/0]
root         5     2  0 Mar29 ?        00:00:00 [kworker/0:0H]
...
```

定义、管理和控制`init`进程的系统被称为**init系统**。我们知道仅仅将内核运行起来是没有什么实际意义，必须由`init`系统将整个linux操作系统代入可操作状态。`init`系统主要的任务就是去运行这些开机启动的程序。但是，不同的场合需要启动不同的程序来进入不同的预设运行模式。例如，启动`shell`进行人机交互，这样就可以让计算机执行一些预订程序完成有实际意义的任务，启动`X`图形系统以便提供友好的人机界面，更加高效的完成任务。这里，命令行界面的`shell`或者`X`图形界面都是一种预设的运行模式。

#### Init系统的历史

大多数linux发行版的`init`系统是和System V相兼容的，被称为`SysVInit`，这就是最初的`init`系统。在linux主要应用于服务器和PC机的时代，`SysVInit`运行概念简单清晰，运行稳定。但是它主要依赖于Shell脚本，导致启动速度太慢。这点在很少重启的服务器上并没有多少影响，而随着linux内核的不断演化发展linux操作系统越来越多地被应用到移动终端设备的时候，启动慢就严重影响用户体验，被越来越多人诟病。

为了解决启动慢的问题，人们开始思考改进`SysVInit`，先后出现了`UpStart`和`Systemd`这两个主要的新一代init系统。`UpStart`已经开发了8年多，在不少系统中已经替换`SysVInit`，而`Systemd`出现比较晚，但发展更快，大有取代`UpStart`统治各个linux发行版的趋势。事实上，Ubuntu和RHEL采用`UpStart`替代了传统的`SysVInit`。而Fedora发行版则从版本15开始使用`Systemd`。

#### SysVInit

对于使用`SysVInit`的老式linux发行版来说，使用“运行级别”（Run Level）来定义"预订的运行模式"。这样的linux发行版一般预置七种运行级别（0-6）。通常情况下，0代表关机，1代表单用户模式（也就是维护模式），6代表重启。运行级别2-5，各个发行版不太一样，但基本都是多用户模式（也就是正常模式）。

`init`进程首先读取检查运行级别的设置文件`/etc/inittab`是否含有`initdefault`项，它代表系统默认的运行级别；如果没有默认运行级别，那么就需要用户进入系统控制台手动决定进入何种运行模式。运行级别的设置文件`/etc/inittab`的第一行一般是这样的：

```
id:2:initdefault:
```

这代表系统启动时的默认运行级别为`2`，如果需要指定其他级别，需要手动修改这个值。

但是问题来了，在不同的运行级别中，系统需要初始化运行的服务进程和需要进行的初始化准备都是不同的，有些linux发行版的运行级别3不需要启动X图形界面系统。系统怎么知道每个级别应该加载哪些程序呢？...回答是每个运行级别在`/etc`目录下面，都有一个对应的子目录，指定要加载的服务程序。这样，用户只需要指定需要进入哪种运行级别，`SysVInit`将负责执行所有该级别下必须的初始化工作。

```
# ll /etc | grep rc
drwxr-xr-x   2 root root    4096 Jan 16 00:48 rc0.d/
drwxr-xr-x   2 root root    4096 Jan 16 00:48 rc1.d/
drwxr-xr-x   2 root root    4096 Jan 16 00:48 rc2.d/
drwxr-xr-x   2 root root    4096 Jan 16 00:48 rc3.d/
drwxr-xr-x   2 root root    4096 Jan 16 00:48 rc4.d/
drwxr-xr-x   2 root root    4096 Jan 16 00:48 rc5.d/
drwxr-xr-x   2 root root    4096 Jan 16 00:48 rc6.d/
-rwxr-xr-x   1 root root     306 Apr 20  2016 rc.local*
drwxr-xr-x   2 root root    4096 Aug 20  2018 rcS.d/
```

上面目录名中的`rc`，表示run command（运行程序），最后的`d`表示directory（目录）。下面让我们看看`/etc/rc2.d`目录中到底指定了哪些程序：

```
# ll  /etc/rc2.d
total 20
lrwxrwxrwx   1 root root   16 Aug 20  2018 K01autofs -> ../init.d/autofs*
-rw-r--r--   1 root root  677 Feb  5  2016 README
lrwxrwxrwx   1 root root   16 Aug 20  2018 S01apport -> ../init.d/apport*
lrwxrwxrwx   1 root root   17 Aug 20  2018 S01rsyslog -> ../init.d/rsyslog*
lrwxrwxrwx   1 root root   15 Aug 20  2018 S01uuidd -> ../init.d/uuidd*
lrwxrwxrwx   1 root root   17 Aug 20  2018 S02postfix -> ../init.d/postfix*
lrwxrwxrwx   1 root root   15 Aug 20  2018 S02rsync -> ../init.d/rsync*
lrwxrwxrwx   1 root root   13 Aug 20  2018 S02ssh -> ../init.d/ssh*
...
```

我们看到，除了`README`以外，其他文件名都是"字母S+两位数字+程序名"的形式。字母`S`表示Start，代表在当前级别下运行启动对应程序；字母`K`，就代表Kill，即从其他运行级别切换过来时需要关闭的程序；后面的两位数字表示处理顺序，数字越小越早处理，所以第一个启动的程序是apport，然后是rsyslog、uuidd、postfix...数字相同时，则按照程序名的字母顺序启动，所以rsyslog会先于uuidd启动。

同时，我们也注意到，`/etc/rcN.d`目录里列出的程序其实都设为符号链接，指向另外一个目录`/etc/init.d`下真正的程序启动脚本，init进程逐一加载开机启动程序，其实就是运行这个目录里的启动脚本。这样做的好处是就是如果需要手动关闭或重启某个进程，直接到目录`/etc/init.d`中寻找对应的启动脚本即可。比如，要重启ssh服务，只需要运行下面的命令：

```
/etc/init.d/ssh restart
```

![](https://i.loli.net/2019/04/02/5ca31c74cc695.jpg)

#### Upstart

当linux内核进入2.6之后，内核的很多功能完全适用于桌面系统甚至嵌入式设备，于是开发人员就开始尝试将linux移植的工作。最具有代表性的是Ubuntu的开发人员[Scott James Remnant](https://en.wikipedia.org/wiki/Scott_James_Remnant)，他曾试图将linux安装在笔记本电脑上，但发现老式的`SysVInit`并不适合桌面系统或便携式设备，原因是：

- 桌面系统或便携式设备的特点是经常重启，而且要频繁地使用硬件热插拔技术，而`SysVInit`没有办法处理这类按需启动外设的需求
- 网络共享盘的挂载问题，`SysVInit`在网络初始化之前分析`/etc/fstab`来发现响应的挂载文件以及挂载位置，但如果网络没有启动，类似于NFS或者iSCSI的网络共享盘都不可访问，当然也无法进行挂载操作。

针对以上`SysVInit`的不足，Ubuntu开发人员决定重新设计和开发一个全新的init系统，也就是`UpStart`。

`UpStart`基于事件机制，例如U盘插入USB接口后，`udev`得到内核通知，发现新设备接入，这就是一个新的事件。`UpStart`在感知到该事件之后触发相应的等待任务，比如处理`/etc/fstab`中存在的挂载点。采用这种事件驱动的模式，`UpStart`完美地解决了即插即用设备带来的新问题，同时通过并行启动程序显著减少系统启动时间。

![](https://i.loli.net/2019/04/02/5ca33dfc347f8.jpg)

在上面的例子中，有7个不同的程序需要启动：Job A、Job B...Job F。在`SysVInit`中，每一个启动项目都由一个独立的脚本负责，它们由`SysVInit`按照顺序串行调用。因此总的启动时间为`T1+T2+T3+T4+T5+T6+T7`。其中一些任务有依赖关系，比如A,B,C,D，而E和F却和A,B,C,D无关。这种情况下，`UpStart`能够并发地运行任务`{(A,B,C),(D,E),F}`，使得总的启动时间减少为`T1+T2+T3`。这无疑增加了系统启动的并行性，从而提高了系统启动速度。但是在`UpStart`中，有依赖关系的服务还是必须先后启动串行执行，比如任务`{(A,B,C)}`。

在`UpStart`中主要的概念是`job`和`event`。所谓`job`其实就是完成特定任务的工作单元，例如启动一个后台服务，或者运行一个配置命令。每个`job`都等待响应一个或多个`event`，一旦获取到响应的`event`，`UpStart`就触发该对应的`job`完成相应的工作。使用`UpStart`的linux发行版系统初始化的过程是在`job`和`event`的相互协作下完成的，可以大致描述如下：内核初始化完成之后，init进程开始运行，init进程自身会发出不同的`event`，这些最初的`event`会触发一些`job`运行；而每个`job`运行过程中会释放新的不同的`event`，这些`event`又将触发新的`job`运行，如此反复，直到整个系统正常运行起来。

那么，哪些`event`会触发某个`job`的运行？这是由"job配置文件"定义的。任何一个`job`都是由一个job配置的文本文件定义的。这个文件包含多个部分，每个部分是一个完整的定义模块，定义了`job`的一个方面，比如`author`部分定义了工作的作者。工作配置文件存放在`/etc/init`目录下面，以`.conf`作为文件后缀的文件。下面，我们就来看看linux下面`cron`的job配置文件定义：

```
# cat /etc/init/cron.conf
# cron - regular background program processing daemon
#
# cron is a standard UNIX program that runs user-specified programs at
# periodic scheduled times

description	"regular background program processing daemon"

start on runlevel [2345]
stop on runlevel [!2345]

expect fork
respawn

exec cron
```

从上面的输出可以看到：

- `description`是关于job的一段简短的描述信息
- `start on`用来定义触发job的所有event
- `stop on`和`start on`语义相反，定义job在哪些event触发时需要停止
- `expect`分为两种，`expect fork`表示进程只会fork一次；`expect daemonize`表示进程会fork两次来将自己变成后台进程
- `exec`配置job需要运行的命令
- `script`定义job需要运行的脚本

关于job的配置文件的具体详细定义，请参考http://upstart.ubuntu.com/cookbook/

> Note: 需要注意的是，而传统的linux发行版系统初始化是基于运行级别的，即`SysVInit`，所以现在很多linux的软件还是采用传统的`SysVInit`脚本启动方式，并没有为`UpStart`开发新的启动脚本，因此即便在Debian和Ubuntu发行版系统上，还必须模拟老式`SysVInit`的运行级别模式，以便和多数现有软件兼容。

#### Systemd

`Systemd`是多数linux发行版系统中最新的init系统，它主要的设计目标是克服`SysVInit`固有的缺陷，提高系统的启动速度。`Systemd`和Ubuntu的`UpStart`是竞争对手，有逐渐取代`UpStart`统治各个linux发行版的版本的趋势。

> Note: 和`UpStart`类似，`Systemd`引入了新的配置方式，对应用程序的开发也有一些新的要求。但是由于历史的原因linux上的很多应用程序并没有来得及为它做相应的改变。如果`Systemd`想替代目前正在运行的init系统，就必须和现有程序兼容。事实上，`Systemd`确实提供了和`SysVInit`兼容的特性，系统中已经存在的服务和进程无需修改即可由`Systemd`管理，这降低了系统向`Systemd`迁移的成本，使得`Systemd`替换现有init系统成为可能。

1) 更快的启动速度

`Systemd`提供了比`UpStart`更激进的并行启动能力，采用了`socket/D-Bus activation`等技术启动服务，最终的目标是：

- 尽可能启动更少的进程
- 尽可能将更多进程并行启动

![](https://i.loli.net/2019/04/02/5ca348201d1e5.jpg)

如上图所示，`Systemd`在`UpStart`的基础上更进一步提高多个启动程序的并发性，即便对于那些相互依赖而必须串行启动的程序也可以并发启动。由于所有的任务都同时并发执行，总的启动时间被进一步降低为`T1`。

2) 按需启动能力

传统的`SysVInit`系统的初始化会将所有用到的后台服务进程全部启动运行，并且系统必须等待所有的程序都启动就绪之后，才算是操作系统初始化完成。这样不仅造成系统启动时间过长，而且也会带来系统资源的浪费。事实上，某些服务很可能在很长一段时间内，甚至整个服务器运行期间都没有被使用过，花费在启动这些服务上的时间是不必要的，同时还会造成不必要的资源浪费。

`Systemd`可以提供按需启动的能力，只有在某个服务程序被真正请求的时候才启动它。当该服务程序结束之后，`Systemd`可以关闭它，等待下次需要时再次启动它。

3) 使用`CGroup`跟踪进程

Init系统的一个重要职责就是负责跟踪和管理服务进程的生命周期，它不仅要可以启动一个服务，也必须也能够停止服务。实际上，停止服务程序比想象中的复杂很多，原因是服务进程一般都会作为后台进程运行，为此服务程序有时候会fork两次。前面在`UpStart`的job配置文件中介绍`expect`关键字时我们说过，需要在配置文件中正确地配置`expect`，这样`UpStart`才能通过对fork系统调用进行计数，从而获知真正的后台进程的PID。但是找到正确的PID在实际操作起来很困难，于是`UpStart`使用`strace`这个笨重的方案来跟踪fork、exit等系统调用。而`Systemd`则采用`CGroup`特性完成服务进程跟踪的任务。当要停止服务进程的时候，通过查询`CGroup`，`Systemd`可以确保找到所有的相关进程，从而干净地停止服务进程。

> Note: CGroup主要用来实现系统资源配额管理，它提供了类似文件系统的接口，当进程创建子进程时，子进程会继承父进程的CGroup。因此无论服务如何启动新的子进程，所有的这些相关进程都会属于同一个CGroup，`Systemd`只需要简单地遍历指定的CGroup即可正确地找到所有的相关进程。

4) 日志服务

`Systemd`自带日志服务`Journal`，用来克服linux系统现有的`syslog`服务的缺点。比如：

- `syslog`不安全，消息的内容无法验证。每一个本地进程都可以声称自己是`Apache PID 4711`，但是`syslog`并没有任何机制去验证而直接将日志并保存到磁盘上。
- 数据没有严格的格式。自动化的日志分析器需要分析人类自然语言字符串来识别消息。`syslog`无严格确定格式导致分析困难低效，另外日志格式的变化会导致分析代码需要更新甚至重写。

`Systemd`自带日志服务`Journal`采用二进制格式保存所有日志信息，用户使用`journalctl`命令来查看日志信息。无需自己编写复杂脆弱的字符串分析处理程序。

5) 配置单元

作为新一代的init系统，`Systemd`将系统初始化过程中每一个启动服务或者配置的任务抽象为一个配置单元，即`Unit`。可以认为一个服务是一个配置单元；一个挂载点是一个配置单元；一个交换分区的配置是一个配置单元。`Systemd`将配置单元归纳为以下一些不同的类型：

> Note: 随着`Systemd`正在快速发展演进，新功能不断增加的同时，配置单元类型也将来继续增加。

- `service`: 最常用的一类配置单元，代表一个后台服务进程，比如`mysqld`
- `socket`: 此类配置单元封装网络通信中的一个套接字。目前`Systemd`支持流式、数据报和连续包的AF_INET、AF_INET6、AF_UNIX socket。每一个套接字配置单元都有一个相应的服务配置单元。相应的服务在第一个"连接"进入套接字时就会启动(例如：`nscd.socket`在有新连接后便启动`nscd.service`)
- `device`: 此类配置单元封装一个存在于linux设备树中的设备。每一个使用`udev`规则标记的设备都将会在`Systemd`中作为一个设备配置单元出现
- `mount`: 此类配置单元封装文件系统结构层次中的一个挂载点。`Systemd`将对这个挂载点进行监控和管理。比如可以在启动时自动将其挂载；可以在某些条件下自动卸载。`Systemd`会将`/etc/fstab`中的条目都转换为挂载点，并在开机时处理
- `automount`: 此类配置单元封装系统结构层次中的一个自挂载点。每一个自挂载配置单元对应一个挂载配置单元，当该自动挂载点被访问时，`Systemd`执行挂载点中定义的挂载行为
- `swap`: 和挂载配置单元类似，交换配置单元用来管理交换分区。用户可以用交换配置单元来定义系统中的交换分区，可以让这些交换分区在启动时被激活
- `target`: 此类配置单元为其他配置单元进行逻辑分组。它们本身实际上并不做什么，只是引用其他配置单元而已。这样便可以对配置单元做一个统一的控制，从而实现`SysVinit`系统中"运行级别"概念。比如想让系统进入图形化模式，需要运行许多服务和配置命令，这些操作都由一个个的配置单元表示，将所有这些配置单元组合为一个目标(target)，就表示需要将这些配置单元全部执行一遍以便进入目标所代表的系统运行状态。(例如：`multi-user.target`相当于在传统使用`SysVInit`的系统中运行级别5)
- `timer`: 定时器配置单元用来定时触发用户定义的操作，这类配置单元取代了`atd`、`crond`等传统的定时服务
- `snapshot`: 与`target`配置单元相似，快照是一组配置单元，保存了系统当前的运行状态

每个配置单元都有一个对应的配置文件，系统管理员的任务就是编写和维护这些不同的配置文件，比如MySQL服务对应名为`mysql.service`的配置文件。这种配置文件的语法非常简单，用户不需要再编写和维护复杂的系统脚本了。

6) 依赖关系

虽然`Systemd`将大量的启动工作解除了依赖，使得它们可以并发启动。但还是存在有些任务，它们之间存在"天生的依赖"，比如：挂载必须等待挂载点在文件系统中被创建；挂载也必须等待相应的物理设备就绪。为了解决这类依赖问题，`Systemd`的配置单元之间可以彼此定义依赖关系。

`Systemd`用配置单元定义文件中的关键字来描述配置单元之间的依赖关系。比如：`unit A`依赖`unit B`，可以在`unit B`的定义中用"require A"来表示。这样`Systemd`就会保证先启动`unit A`再启动`unit B`。

7) Target和运行级别

`Systemd`用目标（target）替代了传统`SysVinit`系统中"运行级别"的概念，提供了更大的灵活性，我们可以继承一个已有的目标，并添加其它的服务程序，来创建自己的新目标。下表列举了`Systemd`下的目标和常见运行级别的对应关系：

| SysVInit运行级别 | Systemd目标 | 描述 |
| --------------- | ---------- | ---- |
| 0 | runlevel0.target, poweroff.target | 关闭系统 |
| 1, s, single | runlevel1.target, rescue.target	  | 单用户模式 |
| 2, 4 | runlevel2.target, runlevel4.target, multi-user.target |	用户定义/域特定运行级别，默认等同于3 |
| 3 | runlevel3.target, multi-user.target | 多用户非图形化，用户可以通过多个控制台或网络登录 |
| 5 | runlevel5.target, graphical.target | 多用户图形化，通常为所有运行级别3的服务外加图形化登录 |
| 6 | runlevel6.target, reboot.target | 重启 |
| emergency | emergency.target | 紧急Shell |

8) `Systemd`的使用

**unit配置单元文件的编写**

类似`UpStart`的Job配置文件，开发人员需要在`unit`配置单元文件中定义服务程序启动的命令行语法，以及和其他服务的依赖关系等。举例来说，下面展示了SSH服务程序的配置单元文件，服务配置单元文件以`.service`为文件名后缀：

```
#cat /etc/system/system/sshd.service
[Unit]
Description=OpenSSH server daemon
[Service]
EnvironmentFile=/etc/sysconfig/sshd
ExecStartPre=/usr/sbin/sshd-keygen
ExecStart=/usrsbin/sshd –D $OPTIONS
ExecReload=/bin/kill –HUP $MAINPID
KillMode=process
Restart=on-failure
RestartSec=42s
[Install]
WantedBy=multi-user.target
```

可以看到，SSH服务配置单元文件分为三个部分：第一个是`[Unit]`部分，这里仅仅包含一个简短的描述信息；第二部分是`[Service]`定义，其中`ExecStartPre`定义启动服务之前应该运行的命令，`ExecStart`定义启动服务的具体命令行语法；第三部分是`[Install]`，`WangtedBy`表明这个服务是在多用户模式下所需要的。

既然说到多用户模式，那就来看看`multi-user.target`的配置单元，它属于`target`类型的配置单元：

```
#cat multi-user.target
[Unit]
Description=Multi-User System
Documentation=man.systemd.special(7)
Requires=basic.target
Conflicts=rescue.service rescure.target
After=basic.target rescue.service rescue.target
AllowIsolate=yes
[Install]
Alias=default.target
```

第一部分`[Unit]`中的`Requires`字段定义表明`multi-user.target`启动的时候`basic.target`也必须被启动；另外`After`字段表明`basic.target`停止的时候，`multi-user.target`也必须停止；如果查看`basic.target`配置单元文件，会发现它又在`Requires`字段中指定了`sysinit.target`等其他的单元；同样`sysinit.target`也会包含其他的单元，正是采用这样的嵌套链式的结构，最终所有需要支持多用户模式的组件服务都会被初始化完成。

在`[Install]`中有`Alias`定义，即定义本单元的别名，这样在运行`systemctl`的时候就可以使用这个别名来引用本单元，这里的别名是`default.target`。

此外在`/etc/systemd/system`目录下还可以看到诸如`*.wants`的目录，放在该目录下的配置单元文件等同于在`[Unit]`中的`wants`关键字，即本单元启动时，还需要启动这些服务单元。我们可以把自己写的`foo.service`文件放入`multi-user.target.wants`目录下，这样每次都会被默认启动了。

**`Systemd`命令行工具`systemctl`**

类似于其他init系统的管理工具，比如`service`、`chkconfig`以及`telinit`，`systemctl`也可以完成复杂的服务进程管理任务，只是语法略有不同。例如：

```
systemctl restart mysql.service
```

用来重启MySQL服务程序

```
systemctl list-unit-files --type=service
```

用来列出可以启动或停止的服务程序列表

```
systemctl enable mysql.service
```

在下次启动时或满足其他触发条件时设置服务程序自动启用

有关具体详细的`systemctl`的使用，请参考:https://www.digitalocean.com/community/tutorials/how-to-use-systemctl-to-manage-systemd-services-and-units

### 用户登录

**登录方式**

等到所有的服务进程初始化完毕，linux操作系统可以让用户登录了，一般的linux发行版会提供三种方式让用户登录：

1. 命令行登录
2. SSH登录
3. 图形界面登录

其中每一种用户登录方式都有对应的认证方式：

1. 命令行登录：init进程调用getty（get teletype）程序，获取输入的用户名和密码然后调用login程序，核对密码。如果用户名密码组合正确无误，就从文件`/etc/passwd`读取该用户指定的shell，然后启动这个shell
2. SSH登录：类似于命令行登陆，只是init进程系统调用SSHD程序，取代getty和login，然后启动相应的shell
3. 图形界面登录：init进程调用显示管理器，Gnome图形界面对应的显示管理器为GDM（GNOME Display Manager），获取用户输入的用户名和密码。如果用户名密码组合正确无误，就读取`/etc/gdm3/Xsession`，启动用户会话

**进入Shell**

Shell，简单来说是命令行界面，让终端用户可以直接与操作系统交互。其中用户登录时打开的Shell，就叫做`login shell`。实际上，对于不同的用户登录方式，Shell初始化的操作是不同的：

1. 命令行登录：首先读入`/etc/profile`，这是对所有用户都有效的配置；然后依次寻找下面三个关于当前用户的配置文件：

```
~/.bash_profile
~/.bash_login
~/.profile
```

> Note: 上面三个文件只要有一个存在，就不再继续读入后面的文件。

2. SSH登录：与命令行登录情况完全相同
3. 图形界面登录：只加载`/etc/profile`和`~/.profile`，`~/.bash_profile`不管存在与否，都不会运行

**打开non-login shell**

事实上，在用户进入操作系统以后，往往会开启另外一个特殊的Shell叫作`non-login shell`，它是不同于登录时出现的Shell，也不读取`/etc/profile`和`.profile`等配置文件。`non-login shell`读入用户的bash配置文件`~/.bashrc`，而且大多数时候，对于bash的定制，都是写在这个文件里面的。即使不进入`non-login shell`，`.bashrc`也会正常运行，我们可以在`~/.profile`文件中可以看到类似于下面的代码片段：

```
if [ -n "$BASH_VERSION" ]; then
    if [ -f "$HOME/.bashrc" ]; then
        . "$HOME/.bashrc"
    fi
fi
```

总之，不管是哪种情况，`.bashrc`在用户登录的时候都会执行，用户的设置可以放心地都写入这个文件了。

之所以有这么繁多复杂的登录设置文件是由于历史原因造成的。在linux出现早期的时候，计算机运行速度很慢，载入配置文件需要很长时间，多数Shell的作者只好把配置文件分成了几个部分，阶段性载入。系统的通用设置放在`/etc/profile`，用户个人的、需要被所有子进程继承的设置放在`.profile`，不需要被继承的设置放在`.bashrc`。
