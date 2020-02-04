---
title: "从docker容器到pod"
date: 2020-02-03
type: "notes"
draft: false
---

很多人应该像我一样，第一次接触docker的概念，都会听到过：docker技术比虚拟机技术更为轻便、快捷，docker容器本质上是进程，甚至我们或多或少接受了docker技术基于linux内核中两个非常重要的特性，namespace和cgroup。那么，namespace和cgroup到底是怎么来隔离docker容器进程的呢？今天我们就来一探究竟。

## docker容器

在了解docker容器进程怎么隔离之前，我们先来看看linux内核中的namespace和cgroup到底是什么。

### namespace

如果让我们自己实现一种类似于vm一样的虚拟技术，我们首先会想到的是怎么解决每个vm的进程与宿主机进程的隔离问题。2008年发布的linux内核v2.6.24带来的linux namespace特性使得我们可以对linux做各种各样的隔离。

熟悉`chroot`命令的想哦同学都应该大体能猜到linux namespace是如何发挥作用的，在linux系统中，系统默认的目录结构都是以根目录`/`开始的，`chroot`命令用来以指定的位置作为`/`来运行指令。与此类似，了解[linux启动流程](https://morven.life/notes/the_knowledge_of_linux/)的人都知道在linux启动的时候又一个Pid为1的`init`进程，其他所有的进程都由此衍生而来，`init`和其他衍生而来的进程形成以`init`进程为根节点树状结构，在同一个进程树里的进程只有有足够的权限就可以审查甚至终止其他进程，这样，显然会带来安全性问题。而PID namespace（linux namespace的一种）允许我们创建单独的进程树，新进程树里面有自己的PID为`1`的进程，该进程一般为创建新进程树的时候指定。PID namespace使得不同进程树里面的进程相互直接不能直接访问，提供了进程间的隔离，甚至可以创建嵌套的进程树：

![pid-namespace.jpg](https://i.loli.net/2020/02/04/34f2VPtHXoTKcjw.jpg)

在linux中，创建namespace很简单，只需要通过`unshare`命令并且指定创建的namespace的类型以及其他参数，比如下面的命令就会创建新的pid namespace并且在其中运行`zsh`：

```
# unshare --fork --pid --mount-proc zsh
```

因为`zsh`是交互式的shell，所以你会发现自己进入了一个新的世界(pid namespace)，我们可以在新的世界里再运行新的进程`sleep`：

```
# ps aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  8.4  0.0  42676  5684 pts/1    S    04:06   0:00 zsh
root        23  0.0  0.0  39084  3252 pts/1    R+   04:06   0:00 ps aux
# sleep 100000

```

启动`sleep`进程前只有两个个进程在运行，`zsh`和`ps`，而且`zsh`进程的pid为`1`，所有其他进程都是它的子进程。但是，如果你打开新的终端并且查看此`zsh`进程会发现它的pid并不是`1`：

```
# ps -ef | grep zsh
root      7180  7179  0 04:06 pts/1    00:00:00 zsh
...
# ps -ef | grep sleep
root      7851  7180  0 04:10 pts/1    00:00:00 sleep 100000
...
```

有此可见，通过父进程树是可以看到子进程树里面的进程的，但是子进程树里面看不到父进程树里面的进程。

事实上，除了pid namespace，linux内核有提供了很多不同类型的namespace：

- Mount(mnt): Mount namespace可以在不影响宿主机文件系统的基础上挂载或者取消挂载文件系统
- PID(Process ID): 在一个pid namespace中，第一个进程的pid为1，所有在此namespace中的其他进程都是此进程的子进程。操作系统级别的其他进程不需要在此进程中
- network(netns): network namespace会虚拟出新的内核协议栈，每个network namespace包括自己的网络设备，路由表等
- IPC(Interprocess Communication): 用来隔离处于不同IPC namespace的进程间的通信资源，如共享内存等
- UTS: UTS namespace用于隔离hostname与domainname

如果运行的linux环境提供[util-linux](https://en.wikipedia.org/wiki/Util-linux)，那么就可以通过它提供的`lsns`命令来查看当前的linux namespace：

```
# lsns -t pid -o NS,PID,PATH
        NS   PID PATH
4026531836     1 /proc/1/ns/pid
4026532445  7180 /proc/7180/ns/pid
```

想要进入到某个linux namespace，需要使用`nsenter`命令，是不是和`docker exec`有点儿类似？

```
# nsenter --pid=/proc/7180/ns/pid echo $SHELL
/bin/bash
```

不对啊，应该是`bash`才对啊，为什么会这样呢？仔细看了doc才明白原来是`nsenter`只是进入到目标namespace，但是`SHELL`环境变量依赖于上下文，也就是说`nsenter`只是切换了namespace，但并没有切换程序运行的上下文环境。与之对应的`docker exec`会完整的切换namespace和运行上下文，所以我们能够使用`docker exec`进行各种debug操作。

### cgroup

上面提到一个进程树里面的进程不能与其他进程树里的进程交互，其实这是不完全对的。举例来说，一个进程树里面的进程会占有宿主机的资源(CPU/Memory/NetworkIO/DisIO...)，这样有可能导致其他的进程得不到足够的资源。linux内核通过[cgroup](https://en.wikipedia.org/wiki/Cgroups)来限制进程能占用的资源数量，这些资源包括CPU，内存，网络带宽，磁盘IO等。需要说明的是namespace和sgroup是两个不同的特性，上面提到的各种namespace有可能只用到其中的一种或者两种，namespace和cgroup可以一起合作来完成对一组进程的管理。

让我们开始创建一个cgroup：

```
# useradd morven
# cgcreate -a morven -g memory:mycgrp
```

通过以上命令我们创建了名为`memory:mycgrp`的cgroup，owner设为`morven`用户，我们来看看对于`memory:mycgrp`默认的资源限制：

```
# ls -l /sys/fs/cgroup/memory/mycgrp/
total 0
-rw-r--r-- 1 morven root 0 Feb  4 05:02 cgroup.clone_children
--w--w--w- 1 morven root 0 Feb  4 05:02 cgroup.event_control
-rw-r--r-- 1 morven root 0 Feb  4 05:02 cgroup.procs
-rw-r--r-- 1 morven root 0 Feb  4 05:02 memory.failcnt
--w------- 1 morven root 0 Feb  4 05:02 memory.force_empty
-rw-r--r-- 1 morven root 0 Feb  4 05:02 memory.kmem.failcnt
-rw-r--r-- 1 morven root 0 Feb  4 05:02 memory.kmem.limit_in_bytes
-rw-r--r-- 1 morven root 0 Feb  4 05:02 memory.kmem.max_usage_in_bytes
-r--r--r-- 1 morven root 0 Feb  4 05:02 memory.kmem.slabinfo
-rw-r--r-- 1 morven root 0 Feb  4 05:02 memory.kmem.tcp.failcnt
-rw-r--r-- 1 morven root 0 Feb  4 05:02 memory.kmem.tcp.limit_in_bytes
-rw-r--r-- 1 morven root 0 Feb  4 05:02 memory.kmem.tcp.max_usage_in_bytes
-r--r--r-- 1 morven root 0 Feb  4 05:02 memory.kmem.tcp.usage_in_bytes
-r--r--r-- 1 morven root 0 Feb  4 05:02 memory.kmem.usage_in_bytes
-rw-r--r-- 1 morven root 0 Feb  4 05:02 memory.limit_in_bytes
-rw-r--r-- 1 morven root 0 Feb  4 05:02 memory.max_usage_in_bytes
-rw-r--r-- 1 morven root 0 Feb  4 05:02 memory.move_charge_at_immigrate
-r--r--r-- 1 morven root 0 Feb  4 05:02 memory.numa_stat
-rw-r--r-- 1 morven root 0 Feb  4 05:02 memory.oom_control
---------- 1 morven root 0 Feb  4 05:02 memory.pressure_level
-rw-r--r-- 1 morven root 0 Feb  4 05:02 memory.soft_limit_in_bytes
-r--r--r-- 1 morven root 0 Feb  4 05:02 memory.stat
-rw-r--r-- 1 morven root 0 Feb  4 05:02 memory.swappiness
-r--r--r-- 1 morven root 0 Feb  4 05:02 memory.usage_in_bytes
-rw-r--r-- 1 morven root 0 Feb  4 05:02 memory.use_hierarchy
-rw-r--r-- 1 morven root 0 Feb  4 05:02 notify_on_release
-rw-r--r-- 1 root   root 0 Feb  4 05:02 tasks
```

默认是以`bytes`为单位，我们来限制`memory:mycgrp`最多使用`10MB`的内存：

```
# echo 10000000 > /sys/fs/cgroup/memory/mycgrp/memory.limit_in_bytes
```

接下来，我们使用`memory:mycgrp`这个cgroup创建进程运行java程序：
```
cgexec -g memory:mycgrp java -jar test.jar
error: Could not execute process `java -jar test.jar` (never executed)

Caused by:
  Cannot allocate memory (os error 12)
```

可以看到，内存的限制导致我们不能在`memory:mycgrp`这个cgroup运行消耗内存比较大的java程序。

实际上，除了限制进程对宿主机各种资源的消耗大小，有时候我们还需要限制进程可以进行哪些系统调用，比如说，我们要限制特性的应用程序不能访问网络，那么就需要linux内核提供的另外一个功能[seccomp-bpf](https://en.wikipedia.org/wiki/Seccomp)，在此就不展开来说。

### 组合成docker容器

有了上面介绍的知识，我们很容易想到docker会为每一个容器创建namespace和cgroup的组合来实现隔离：

![docker-namespace-cgroup.jpg](https://i.loli.net/2020/02/04/j27FwnWPNZaU9fe.jpg)

除了需要挂载宿主机的文件系统或者与宿主机做端口映射之外，每个容器实际上都是独立的namespace和cgroup的组合。但是，通过一些命令行参数，我们完全可以让多个容器使用同一个namespace：

```
cat <<EOF >> nginx.conf
error_log stderr;
events { worker_connections  1024; }
http {
    access_log /dev/stdout combined;
    server {
        listen 80 default_server;
        server_name example.com www.example.com;
        location / {
            proxy_pass http://127.0.0.1:2368;
        }
    }
}
EOF
# docker run --ipc=shareable -d --name nginx -v `pwd`/nginx.conf:/etc/nginx/nginx.conf -p 8080:80 nginx
```

接下来，我们启动一个容器运行[ghost](https://github.com/TryGhost/Ghost)，但是我们添加了额外的参数保证`ghost`容器加入到`nginx`容器的namespace：

```
# docker run -d --name huge -v $(pwd):/src --net=container:nginx --ipc=container:nginx --pid=container:nginx ghost
```

现在就可以访问`http://localhost:8080/`就可以看到ghost的页面，实际声这时候`nginx`容器可以将本地请求代理给`ghost`容器，通过指定namespace参数我们可以让多个容器运行在相同的namespace中：

![docker-namespace-cgroup-share.jpg](https://i.loli.net/2020/02/04/yUQSt8mrnLZIw9a.jpg)

## pod

## 参考

- https://jvns.ca/blog/2016/10/10/what-even-is-a-container/
- https://www.toptal.com/linux/separation-anxiety-isolating-your-system-with-linux-namespaces
