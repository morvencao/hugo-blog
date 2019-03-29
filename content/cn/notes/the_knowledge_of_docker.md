---
title: "容易被忽视的docker知识"
date: 2018-03-14
type: "notes"
draft: false
---

[Docker](https://www.docker.com/)是一个划时代的产品，它彻底地释放了计算机虚拟化的威力，极大的提高了应用的部署、测试和分发。虽然我们几乎每天都使用docker，但还是有一些特别容易被忽略却很重要的docker知识，今天，我们就集中起来聊一聊。

## Docker与传统虚拟机的区别

经常有人说“docker是一种性能非常好的虚拟机”，这种说法是错误的。Docker相比于传统虚拟机的技术来说更为轻便，具体表现在docker不是在宿主机上虚拟出一套硬件后再运行一个完整的操作系统，然后再在其上运行所需的应用进程，而是docker容器里面的进程直接运行在宿主的内核（Docker会做文件系统、网络互联到进程隔离等等），容器内没有自己的内核，而且也没有进行硬件虚拟。这样一来docker会相对于传统虚拟机来说“体积更轻、跑的更快、同宿主机下可创建的个数更多”。

Docker不是虚拟机，容器中的应用都应该以前台执行，而不是像虚拟机、物理机里面那样，用`systemd`去启动后台服务，容器内没有后台服务的概念。举个例子，常有人在dockerfile里面这样写：

```
CMD service nginx start
```

然后发现容器执行后就立即退出了，甚至在容器内去使用`systemctl`命令结果却发现根本执行不了。没有区分docker容器和虚拟机的差异，依旧在以传统虚拟机的角度去理解容器。对于docker容器而言，其启动程序就是容器应用进程，容器就是为了主进程而存在的，主进程退出，容器就失去了存在的意义，从而退出，其它辅助进程不是它需要关心的东西。而使用CMD指令`service nginx start`，则是以后台守护进程形式启动nginx服务。`CMD service nginx start`最终会被docker引擎转化为`CMD [ "sh", "-c", "service nginx start"]`，因此主进程实际上是`sh`。那么当`service nginx start`命令结束后，`sh`也就结束了，`sh`作为主进程退出了，自然就会令容器退出。

正确的做法是直接执行`nginx`可执行文件，并且要求以前台形式运行：

```
CMD ["nginx", "-g", "daemon off;"]
```

## 慎用docker commit

知道使用`docker commit`可以在基础镜像层的基础上定制新的镜像。我们知道镜像是多层存储，每一层是在前一层的基础上进行的修改；而容器同样也是多层存储，是在以镜像为基础层，在其基础上加一层作为容器运行时的存储层。

举个例子，我们使用`docker commit`的定制一个nginx镜像：

```
$ docker run --name mynginx -d -p 80:80 nginx
```

上面的命令帮我们启动一个nginx的容器，接着我们就可以使用`http://localhost`来访问这个容器提供的web服务了，如果没啥意外的话，我们会看到
接着我们访问如下输出：

```
$ curl http://localhost
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

现在我们想定制这个镜像的输出，可以交互式终端方式进入`mynginx`容器，并执行了`bash`命令：

```
$ docker exec -it mynginx bash
$ echo '<h1>Hello, Docker!</h1>' > /usr/share/nginx/html/index.html
$ exit
exit
```

我们修改了容器的文件，也就是改动了容器的存储层，可以通过`docker diff`命令看到具体的改动：

```
$ docker diff mynginx
C /var
C /var/cache
C /var/cache/nginx
A /var/cache/nginx/scgi_temp
A /var/cache/nginx/uwsgi_temp
A /var/cache/nginx/client_temp
A /var/cache/nginx/fastcgi_temp
A /var/cache/nginx/proxy_temp
C /root
A /root/.bash_history
C /run
A /run/nginx.pid
C /usr
C /usr/share
C /usr/share/nginx
C /usr/share/nginx/html
C /usr/share/nginx/html/index.html
```

紧接着就能像`git`一样提交保存我们的定制：
```
$ docker commit --message "update index.html" mynginx nginx:v2
sha256:f186f20e1afc40dc16cd93bd9843e15aea7e9d0db67057850f15ff7fed10d2f2
```

可以使用`docker history`查看新镜像的更改历史：

```
$ docker history nginx:v2
IMAGE               CREATED              CREATED BY                                      SIZE                COMMENT
f186f20e1afc        About a minute ago   nginx -g daemon off;                            97B                 update index.html
2bcb04bdb83f        13 hours ago         /bin/sh -c #(nop)  CMD ["nginx" "-g" "daemon…   0B
<missing>           13 hours ago         /bin/sh -c #(nop)  STOPSIGNAL SIGTERM           0B
<missing>           13 hours ago         /bin/sh -c #(nop)  EXPOSE 80                    0B
<missing>           13 hours ago         /bin/sh -c ln -sf /dev/stdout /var/log/nginx…   22B
<missing>           13 hours ago         /bin/sh -c set -x  && apt-get update  && apt…   54MB
<missing>           13 hours ago         /bin/sh -c #(nop)  ENV NJS_VERSION=1.15.10.0…   0B
<missing>           13 hours ago         /bin/sh -c #(nop)  ENV NGINX_VERSION=1.15.10…   0B
<missing>           13 hours ago         /bin/sh -c #(nop)  LABEL maintainer=NGINX Do…   0B
<missing>           14 hours ago         /bin/sh -c #(nop)  CMD ["bash"]                 0B
<missing>           14 hours ago         /bin/sh -c #(nop) ADD file:4fc310c0cb879c876…   55.3MB
```

测试我们的新镜像：

```
$ docker run -d -p 81:80 nginx:v2
14951d3027633d59b86458211d7fd234949a04c878c508fc904a96e5d79c42bb
$ curl http://localhost:81
<h1>Hello, Docker!</h1>
```

确实，`docker commit`可以帮助我们保存容器运行现场，实验docker镜像的分层存储，但是仔细观察之前`docker diff mynginx`的结果，可以看到除了我们自己的修改之外，还有很多文件也被改动了，仅仅是简单的一个文件就如此，当遇到安装更新软件包等复杂操作的时候，会有大量我们无法控制的改动，可能会是镜像变得无比臃肿。所以要慎用`docker commit`，尤其是生产环境。

此外，使用`docker commit`意味着所有对镜像的操作都是黑箱操作，生成的镜像也被称为**黑箱镜像**，这就是说就是除了制作镜像人知道新镜像是怎么生成的，别人根本无从得知。虽然`docker diff`或许可以告诉得到一些线索，但是远远不到可以确保生成一致镜像的地步。这种黑箱镜像的维护工作是非常痛苦的。

所以要定制镜像，推荐使用`dockerfile`，它可以很好地解决制作镜像无法重复的问题、镜像构建透明性的问题等。

## Docker指令RUN的最佳实践

Docker镜像的分层存储特性决定了dockerfile中每一个指令都会建立一层，`RUN`也不例外。每一个`RUN`指令新建一层，在其上执行这些命令，执行结束后，commit 这一层的修改，构成新的镜像。所以我们在使用RUN指令的时候一定要注意不要将多个无意义且有关联的命令写到不同的RUN指令中，因为这不但会使镜像层数增加，而且有些中间层完全没有必要保存。另外，`Union FS`是有最大层数限制的，之前的docker版本规定是最大不得超过42层，现在是不得超过127层。推荐的做法是使用一个RUN指令，并使用`&&`将各个关联的命令串联起来，来建立新的一层，同时最好在命令的最后有相关的清理工作，例如：

```
FROM debian:stretch

RUN buildDeps='gcc libc6-dev make wget' \
    && apt-get update \
    && apt-get install -y $buildDeps \
    && wget -O redis.tar.gz "http://download.redis.io/releases/redis-5.0.3.tar.gz" \
    && mkdir -p /usr/src/redis \
    && tar -xzf redis.tar.gz -C /usr/src/redis --strip-components=1 \
    && make -C /usr/src/redis \
    && make -C /usr/src/redis install \
    && rm -rf /var/lib/apt/lists/* \
    && rm redis.tar.gz \
    && rm -r /usr/src/redis \
    && apt-get purge -y --auto-remove $buildDeps
```

> Note: 相关的清理工作一定要在执行命令的同一层，因为镜像是多层存储，每一层的东西并不会在下一层被删除。


## Docker镜像构建上下文(Context)

我们使用dockerfile构建镜像的常用操作是：

```
$ docker build -t <image>:<tag> .
```

看到在命令的最后有一个`.`。`.`表示当前目录，而`dockerfile`就在当前目录，因此很多人认为这个`.`是指dockerfile所在路径。这么理解其实是不准确的，其实这在指定镜像构建的**上下文路径**。何为上下文呢？

要了解镜像构建的上下文，先要知道docker的架构。Docker是典型的CS(Client/Server)架构的软件，以后台服务运行的docker引擎作为服务器端，而实际和终端用户进行交互的是docker客户端。docker引擎暴露一组[Rest API](https://docs.docker.com/develop/sdk/)供客户端调用，从而完成各种实际功能。虽然表面上我们好像是在本机执行各种 docker 功能，但实际上，一切都是使用的远程调用形式在服务端（Docker 引擎）完成。

当我们进行镜像构建的时候，并非所有定制都会通过`RUN`指令完成，经常会需要将一些本地文件复制进镜像，比如通过`COPY`指令、`ADD`指令等。而`docker build`命令构建镜像，其实并非在本地构建，而是在服务端，也就是docker引擎中构建的。那么在这种客户端/服务端的架构中，如何才能让服务端获得本地文件呢？

这就引入了**上下文**的概念。当构建的时候，用户需要指定构建镜像上下文的路径，`docker build`命令得知这个路径后，会将路径下的所有内容打包，然后上传给docker引擎。这样docker引擎收到这个上下文包后，展开就会获得构建镜像所需的一切文件。

比如，我们在有着一个dockerfile：

```
FROM node:slim
COPY ./package.json /app/
```


上面的COPY指令并不是复制dockerfile所在目录下的`package.json`，而是复制**上下文**目录下的`package.json`到新镜像中。像COPY这类指令中的源文件的路径都是相对路径。这也是为什么`COPY ../package.json /app`或者`COPY /opt/xxxx /app`无法工作的原因，因为这些路径已经超出了**上下文**的范围，Docker引擎无法获得这些位置的文件，如果真的需要那些文件，应该将它们复制到上下文目录中去。

现在仔细观察一下构建的过程，我们会发现有这个发送上下文的过程：

```
$ docker build -t myapp:v1 .
Sending build context to Docker daemon   2.56kB
Step 1/2 : FROM node:slim
 ---> 6a8b33e0406d
Step 2/2 : COPY ./package.json /app/
 ---> 5bc4ede5a3aa
Successfully built 5bc4ede5a3aa
Successfully tagged myapp:v1
```

了解**上下文**对于镜像构建是很重要的，避免犯一些不应该的错误。比如有人常常把dockerfile放到硬盘某个目录下面去构建镜像，结果发现`docker build`执行后，在发送一个几十GB的东西，极为缓慢而且很容易构建失败。那是因为这种做法是在让`docker build`打包整个硬盘，这显然是使用错误。

推荐的做法是，应该会将`dockerfile`置于一个空目录下，或者项目根目录下。如果该目录下没有所需文件，那么应该把所需文件复制一份过来。如果目录下有些东西确实不希望构建时传给docker引擎，那么可以用`.gitignore`一样的语法写一个`.dockerignore`，该文件是用于剔除构建上下文中不需要传送给docker引擎的文件或目录。

另外，需要关注的是`docker build`还支持从URL(比如git仓库)、tar压缩包构建，甚至还可以从标准输入读取dockerfile来构建镜像。

## COPY和ADD的区别

COPY指令的作用是复制文件到新镜像中，有两种格式：

```
COPY [--chown=<user>:<group>] <source1>, ... <destionation>
COPY [--chown=<user>:<group>] ["<source1>",... "<destionation>"]
```

使用COPY指令，源文件的各种元数据都会保留。比如读、写、执行权限、文件变更时间等。这个特性对于镜像定制很有用。特别是构建相关文件都在使用Git进行管理的时候。

ADD指令和COPY的格式和性质基本一样，却是更高级的复制。

1）比如`<source>`可以是URL。这种情况下，docker引擎会试图去下载这个URL的文件放到`<destination>`，下载后的文件权限自动设置为`600`，如果这并不是想要的权限，那么还需要增加额外的一层RUN进行权限调整，另外，如果下载的是个压缩包，需要解压缩，也一样还需要额外的一层RUN指令进行解压缩。

所以不如直接使用 RUN 指令，然后使用`wget`或者`curl`工具下载，处理权限、解压缩、然后清理无用文件更合理。因此，这个功能其实并不实用，而且不推荐使用。

2）如果`<source>`为一个tar压缩文件的话，压缩格式为`gzip`,`bzip2`以及`xz`的情况下，ADD指令将会自动解压缩这个压缩文件到`<destination>`。

此外，需要注意的是ADD指令会令镜像构建缓存失效，从而可能会令镜像构建变得比较缓慢。

所以在COPY和ADD指令中选择最佳实践是：**所有的文件复制均使用COPY指令，仅在需要自动解压缩的场合使用ADD**。

## CMD和ENTRYPOINT的区别

CMD指令和ENTRYPOINT指令都是用来指定默认的容器主进程的启动命令的。

CMD指令指定的默认启动命令及参数，在运行时可以指定新的命令来替代镜像设置中的这个默认命令，比如，`ubuntu`镜像默认的`CMD`是`/bin/bash`，如果我们直接`docker run -it ubuntu`的话，会直接进入`bash`。我们也可以在运行时指定运行别的命令，如`docker run -it ubuntu cat /etc/os-release`。这就是用`cat /etc/os-release`命令替换了默认的`/bin/bash`命令了，输出了系统版本信息。

ENTRYPOINT指令指定的默认命令在运行时也可以替代，不过比CMD要略显繁琐，需要通过`docker run`的参数`--entrypoint`来指定新的命令。同时，当指定了ENTRYPOINT指令后，CMD指令的含义就发生了改变，不再是直接的运行其命令，而是将CMD的内容作为参数传给ENTRYPOINT指令，换句话说实际执行时，将变为：

```
<ENTRYPOINT> "<CMD>"
```

ENTRYPOINT指令还有一个作用是做程序运行前的一些准备工作。

例如`mysql`之类的数据库，可能需要一些数据库配置、初始化的工作，这些工作要在最终的`mysql`服务器运行之前解决。此外，可能希望避免使用`root`用户去启动服务，从而提高安全性，而在启动服务前还需要以`root`身份执行一些必要的准备工作，最后切换到服务用户身份启动服务。或者除了服务外，其它命令依旧可以使用`root`身份执行，方便调试等。

这些准备工作是和容器CMD指令的内容无关，这种情况下，可以写一个脚本，然后放入ENTRYPOINT指令中去执行，而这个脚本会将接到的参数（也就是`<CMD>`）作为命令，在脚本最后执行。比如官方镜像redis中就是这么做的：

```
FROM alpine:3.6
...
RUN addgroup -S redis && adduser -S -G redis redis
...
ENTRYPOINT ["docker-entrypoint.sh"]

EXPOSE 6379
CMD [ "redis-server" ]
```

可以看到其中为了redis服务创建了`redis`用户以用户组，并在最后指定了ENTRYPOINT为`docker-entrypoint.sh`脚本：

```
#!/bin/sh
...
# allow the container to be started with `--user`
if [ "$1" = 'redis-server' -a "$(id -u)" = '0' ]; then
    chown -R redis .
    exec su-exec redis "$0" "$@"
fi

exec "$@"
```

该脚本的内容就是根据CMD的内容来判断，如果是`redis-server`的话，则切换到`redis`用户身份启动服务器，否则依旧使用`root`身份执行。比如：

```
$ docker run -it redis id
uid=0(root) gid=0(root) groups=0(root)
```