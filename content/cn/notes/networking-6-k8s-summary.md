---
title: "Kubernetes网络(一)"
date: 2020-02-03
type: "notes"
draft: false
---

通过前面的一些列笔记，我们对于容器网络的各种模型的实现原理已经有了基本的认识，而Kubernetes作为一种容器编排平台，通过整合规模庞大的容器实例形成集群才真正将容器技术发扬光大。这些容器可能运行在异构的底层网络环境中，如何保证这些容器间的互通是实际生产环境中首要考虑的问题。

## Kubernetes网络基本要求

Kubernetes对容器技术做了更多的抽象，其中最重要的一点是提出pod的概念，pod是Kubernetes资源调度的基本单元，我们可以简单地认为pod是容器的一种延伸扩展，同一个pod内的所有容器共享同一个netns网络命名空间，并且基于此提出Kubernetes网络设计的原则：

> 每一个Pod都有一个独特的IP地址，所有pod都在一个可以直接连通的、扁平的网络空间中。

由此我们可知：

1. 同一个pod内的所有容器之间共享端口，可直接通过`localhost`+对应的端口来访问
2. 每个pod有单独的IP，所以不需要考虑容器端口与主机端口映射以及端口冲突问题

同时，Kubernetes进一步确定了对一个合格集群网络的基本要求：
1. 任意两个pod之间其实是可以直接通信的，无需显式地使用NAT进行地址的转换；
2. 任意集群节点node与任意pod之间是可以直接通信的，无需使用明显的地址转换，反之亦然；
3. 任意pod看到自己的IP跟别人看见它所用的IP是一样的，中间不能经过地址转换；

也就是说，必须同时满足以上三点的网络模型才能适用于kubernetes，事实上，在早期的Kubernetes中，并没有什么网络标准，只是提出了以上基本要求，只有满足这些要求的网络才可以部署Kubernetes，基于这样的底层网络假设，Kubernetes设计了`pod-deployment-service`的经典三层服务访问机制。直到1.1发布，Kubernetes才开始采用全新的[CNI(Container Network Interface)](https://github.com/containernetworking/cni)网络标准。

## CNI

其实，我们在前面介绍容器网络的时候，就提到了CNI网络规范，CNI相对于[CNM(Container Network Model)](https://github.com/docker/libnetwork/blob/master/docs/design.md)对开发者的约束更少，更开放，不依赖于Docker。事实上，CNI规范确实非常简单，详见：https://github.com/containernetworking/cni/blob/master/SPEC.md

![networking-cni.jpg](https://i.loli.net/2020/02/04/Iz3AwFR6lPdbcmp.jpg)

实现一个CNI网络插件只需要**一个配置文件**和**一个可执行的文件**：
- 配置文件描述插件的版本、名称、描述等基本信息
- 可执行文件会被上层的容器管理平台调用，一个CNI可执行文件自需要实现将容器加入到网络的ADD操作以及将容器从网络中删除的DEL操作（以及一个可选的VERSION查看版本操作）

调用CNI网络插件的数据通过两种方式传递：
- 环境变量
- 标准输入

kubernetes使用了CNI网络插件的工作流程：
1. kubelet先创建pause容器生成对应的netns网络命名空间
2. 网络driver根据配置调用具体的CNI插件，可以配置成CNI插件链来进行链式调用
3. 当一个CNI插件被调用时，它根据环境变量以及1个命令行参数来获得需要执行的操作、目标网络netns、容器的网络设备必要信息，然后执行ADD操作
3. CNI插件给pause容器配置正确的网络，pod中其他的容器都是用pause容器的网络

那么，什么是pause容器，它在pod中处于什么样的位置呢？这就涉及到单个pod的基本网络模型。

## pod的网络模型

pod作为容器的一种扩展技术
