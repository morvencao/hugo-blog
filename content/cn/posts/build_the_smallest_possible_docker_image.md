---
title: "创建最小 Docker 镜像"
date: 2019-01-20
categories: ['note', 'tech']
draft: false
---

如果你熟悉 [docker](https://www.docker.com/)，你可能知道 docker 镜像存储使用 [Union FS](https://en.wikipedia.org/wiki/Union_mount) 的分层存储技术。在构建一个 docker 镜像时，会一层一层构建，前一层是后一层的基础，每一层构建完成之后就不会再改变。正是因为这一点，我们在构建 docker 镜像的时候，要特别小心，每一层尽量只包含需要的东西，构建应用额外的东西尽量在构建结束的时候删除。举例来说，比如你在构建一个 Go 语言编写的简单应用程序的时候，原则上只需要一个 Go 编译出来的二进制文件，没有必要保留构建的工具以及环境。

docker 官方提供了一个特殊的空镜像 scratch，使用这个镜像意味着我们不需要任何的已有镜像为基础，直接将我们自定义的指令作为镜像的第一层。

```dockerfile
FROM scratch
...
```

实际上，我们可以创建自己的 scratch 镜像：

```bash
$ tar cv --files-from /dev/null | docker import - scratch
```

那么，问题来了，我们可以使用 scratch 镜像为基础制作哪些镜像呢？答案是所有不需要任何依赖库的可执行文件都可以以 scratch 为基础镜像来制作。具体来说，对于 linux 下静态编译的程序来说，并不需要操作系统提供的运行时支持，所有需要的一切都已经在可执行文件中包含了，比如使用 Go 语言开发的很多应用会使用直接 `FROM scratch` 的方式制作镜像，这样最终的镜像体积非常小。

下面是一个简单的 Go 语言开发的 web 程序代码：

```golang
package main

import (
	"fmt"
	"net/http"
)

func main() {
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, "Hello, you've requested: %s\n", r.URL.Path)
	})

	http.ListenAndServe(":80", nil)
}
```

我们可以使用 `go build` 来编译此程序，并以 scratch 为基础制作 docker 镜像，dockerfile 如下：

```dockerfile
FROM scratch
ADD helloworld /
CMD ["/helloworld"]
```

接下来开始编译并构建 docker 镜像：

```bash
mc@mcmbp:~/gocode/src/hello# go build -o helloworld
mc@mcmbp:~/gocode/src/hello# docker build -t helloworld .
Sending build context to Docker daemon  7.376MB
Step 1/3 : FROM scratch
 --->
Step 2/3 : ADD helloworld /
 ---> 000f150706c7
Step 3/3 : CMD ["/helloworld"]
 ---> Running in f9c2c6932a34
Removing intermediate container f9c2c6932a34
 ---> 496f865c05e4
Successfully built 496f865c05e4
Successfully tagged helloworld:latest
```

这样镜像就构建成功了，我们来看一下大小

```bash
mc@mcmbp:~/gocode/src/hello# docker image ls
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
helloworld          latest              496f865c05e4        8 seconds ago       7.37MB
```

但是运行这个镜像，会发现容器无法创建：

```bash 
mc@mcmbp:~/gocode/src/hello# docker run -ti -p 80:80 helloworld
standard_init_linux.go:207: exec user process caused "no such file or directory"
```

原因是我们的 helloworld 可执行文件运行的时候依赖的一些库如 libc 还是动态链接的，而 scratch 镜像完全是空的，所以构建 helloworld 可执行文件的时候指定静态链接标志 `-static` 和其他参数，使生成的 helloowrld 二进制文件静态链接所有的库：

```bash
$ CGO_ENABLED=0 go build -a -ldflags '-extldflags "-static"' -o helloworld .
```

然后重新创建 docker 镜像：

```bash
mc@mcmbp:~/gocode/src/hello# CGO_ENABLED=0 go build -a -ldflags '-extldflags "-static"' -o helloworld .
mc@mcmbp:~/gocode/src/hello# docker build -t helloworld .
Sending build context to Docker daemon  7.316MB
Step 1/3 : FROM scratch
 --->
Step 2/3 : ADD helloworld /
 ---> 3fec774cb2a4
Step 3/3 : CMD ["/helloworld"]
 ---> Running in cbe7dc97d6ad
Removing intermediate container cbe7dc97d6ad
 ---> d15a1e6d759a
Successfully built d15a1e6d759a
Successfully tagged helloworld:latest
mc@mcmbp:~/gocode/src/hello# docker image ls -a
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
helloworld          latest              d15a1e6d759a        3 seconds ago       7.31MB
<none>              <none>              3fec774cb2a4        3 seconds ago       7.31MB
```

运行 docker 镜像：

```bash
mc@mcmbp:~/gocode/src/hello# docker run -ti -d -p 80:80 helloworld
3c77ae750352245369c4d142e4e57fd3c0f1e11d67ef857235417ec475ef6286
mc@mcmbp:~/gocode/src/hello# curl -v localhost
* Rebuilt URL to: localhost/
*   Trying 127.0.0.1...
* Connected to localhost (127.0.0.1) port 80 (#0)
> GET / HTTP/1.1
> Host: localhost
> User-Agent: curl/7.47.0
> Accept: */*
>
< HTTP/1.1 200 OK
< Date: Tue, 19 Mar 2019 12:54:28 GMT
< Content-Length: 27
< Content-Type: text/plain; charset=utf-8
<
Hello, you've requested: /
* Connection #0 to host localhost left intact
```

但是，问题来了，如果我们在 MacOS 上面编译 helloowrld 二进制文件并制作镜像，可以运行 docker 容器吗？答案是不行！

我们需要在的时候指定 `GOOS=linux`, 也就是完整的编译命令应该是这样的：

```bash
$ CGO_ENABLED=0 GOOS=linux go build -a -ldflags '-extldflags "-static"' -o helloworld .
```

对于有些有强迫症的程序员来说，他们想更进一步，为什么不在容器里面编译可执行文件然后再构建镜像呢？这样的好处在于可以控制 Go 语言的编译环境，保证可重复编译，而且对于某些与持续集成工具集成的项目来说十分友好。

Docker-CE 17.5 引入了一个从 scratch 构建镜像的新特性，叫做“Multi-Stage Builds”。有了这个新特性之后，我们可以这样去写我们的 dockerfile:

```dockerfile
FROM golang as compiler
RUN CGO_ENABLED=0 go get -a -ldflags '-s' \
github.com/morvencao/helloworld
FROM scratch
COPY --from=compiler /go/bin/helloworld .
EXPOSE 80
CMD ["./helloworld"]
```

是的，你没有看错，确实是一个 dockerfile 里面包含两个 FROM 指令，需要说明的是：

1. `FROM golang as compiler` 是给第一阶段的构建起一个名字叫 `compiler`
2. `COPY --from=compiler /go/bin/helloworld .` 是引用第一阶段的构建产出，以此构建第二阶段

如果你没有给第一阶段起名，可以使用构建阶段的序号(以0开始)来指定，像这样：`--from=0`，但是出于可读性的考虑，起名感觉更好一点儿。

构建完成后我们来看镜像的大小：

```bash
mc@mcmbp:~/gocode/src/hello# docker image ls
REPOSITORY        TAG      IMAGE ID       CREATED          SIZE
helloworld        latest   071ca07e23f5   1 minutes ago    7.31MB
<none>            <none>   2471fd63f0e7   1 minutes ago    720MB
golang            latest   6d0bfafa0452   2 weeks ago      703MB
```

运行 docker 镜像：

```bash
mc@mcmbp:~/gocode/src/hello# docker run -ti -d -p 80:80 helloworld
3c77ae750352245369c4d142e4e57fd3c0f1e11d67ef857235417ec475ef6286
mc@mcmbp:~/gocode/src/hello# curl -v localhost
* Rebuilt URL to: localhost/
*   Trying 127.0.0.1...
* Connected to localhost (127.0.0.1) port 80 (#0)
> GET / HTTP/1.1
> Host: localhost
> User-Agent: curl/7.47.0
> Accept: */*
>
< HTTP/1.1 200 OK
< Date: Tue, 19 Mar 2019 12:54:28 GMT
< Content-Length: 27
< Content-Type: text/plain; charset=utf-8
<
Hello, you've requested: /
* Connection #0 to host localhost left intact
```

这样，我们就将编译并创建最终镜像集成到一个 dockerfile 里面了，而且构建出来的镜像体积也非常小。

## Refer:

https://medium.com/@kelseyhightower/optimizing-docker-images-for-static-binaries-b5696e26eb07
https://medium.com/@adriaandejonge/simplify-the-smallest-possible-docker-image-62c0e0d342ef
