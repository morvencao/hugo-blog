---
title: "一份好吃的Istio入门餐"
date: 2019-06-23
type: "posts"
draft: false
---

前两天Istio正式发布1.2版本，至此，Istio社区再次回归每3个月一个大版本+每月一个小版本更新的传统。从Istio1.0发布以来，社区在对改进Istio的一些非功能性的需求（例如性能与扩展性）方面的努力是大家有目共睹的。我觉得是时候写一篇通俗易懂的Istio入门文章，让更多的人去体验一下Istio在云时代所带来的各种益处。

### 为什么需要Istio

说到Istio，就不得不提到另外一个名词：Service Mesh，中文名为**服务网格**。

相信很多人对于传统的单体应用以及它所面临的问题很熟悉，一种显而易见的方案是将其分解为多个微服务。这确实能简化微服务的开发，但是不得不说，也带来非常多的挑战，比如对于成百上千的微服务的通信、监控以及安全性的管理并不是一件简单的事。目前为止，针对这些问题确实有一些解决方案（比如[Spring Cloud](https://spring.io/projects/spring-cloud)），但基本都是通过类似于类库或者定制化脚本的方式将为服务串联在一起，附带很高的维护升级成本。

![](https://i.loli.net/2019/06/24/5d1033052a66711533.jpg)

Service Mesh的出现正是为了解决这一问题。它是位于底层网络与上层服务之间的一层基础设施，提供了一种透明的、与编程语言无关的方式，使网络配置、安全配置以及遥测等操作能够灵活而简便地实现自动化。从本质上说，它解耦了服务的开发与运维工作。如果你是一名开发者，那么部署升级服务的时候不需要担心这些操作会对整个分布式系统带来哪些运维层面的影响；与之对应，如果你是运维人员，那么也可以放心大胆的服务之间的运维结构进行变更，而不需要修改服务的源代码。

而Istio真正地将Service Mesh的概念发扬光大。它创新性地将控制平面（Control Plane）与数据平面（Data Plane）解耦，通过独立的控制平面可以对Mesh获得全局性的可观测性（Observability）和可控制性（Controllability），从而让Service Mesh有机会上升到平台的高度，这对于对于希望提供面向微服务架构基础设施的厂商，或是企业内部需要赋能业务团队的平台部门都具有相当大的吸引力。

为什么会需要这样的设计呢？

我们先来看一个单独的一个微服务，正常情况下不管单个微服务部署在一个物理机上，亦或一个容器里，甚至一个k8s的pod里面，它的基本数据流向不外乎所有的Inbound流量作为实际业务逻辑的输入，经过微服务处理的输出作为Outbound流量：

![](https://i.loli.net/2019/06/24/5d108aa2db1e428844.jpg)

随着应用的衍进与扩展，微服务的数量快速增长，整个分布式系统变得“失控”，没有一种通用可行的方案来监控、跟踪这些独立、自治、松耦合的微服务组件的通信；对于服务发现和负载均衡，虽说像k8s这样的平台提供了通过kube-proxy下发iptables规则来实现的基本的服务发现和转发能力，不能满足高并发应用下的高级的服务特性；至于更加复杂的熔断、限流、降级等需求，看起来不通过应用侵入式编程几乎不可能实现。

![](https://i.loli.net/2019/06/24/5d1057e6e164054939.jpg)

但是，真的是这样吗？我们来换一个思路，想象一下，如果我们在部署微服务的同时部署一个sidecar，这个sidecar充当代理的功能，它会拦截所有流向微服务业务逻辑的流量，做一些预处理（解包然后抽取、添加或者修改特定字段再封包）之后在将流量转发给微服务，微服务处理的输出也要经过sidecar拦截做一些后处理（解包然后抽取、删除或者修改特定字段再封包），最后将流量分发给外部：

![](https://i.loli.net/2019/06/24/5d108af27291e99200.jpg)

随着微服务数量的增长，整个分布式应用看起来类似于这样，可以看到所有的sidecar接管了微服务之间通信的流量：

![](https://i.loli.net/2019/06/24/5d105d956892f16330.png)

这样的架构也有问题，数量少的情况下，sidecar的配置简单，我们可以简单应对，但是sidecar的数量会随着微服务的数量不断增长，sidecar需要又一个能集中管理控制模块根据整个分布式系统的架构变动动态下发sidecar配置，这里我们把随着分布式应用一起部署的sidecar成为数据平面（Data Plane），而集中是的管理模块成为控制平面（Control Plane）。现在整个架构看起来会像是这样：

![](https://i.loli.net/2019/06/24/5d105f047283710431.jpg)

### Istio架构设计

让我们再次简化一下上面的架构设计，这次我们将关注点放在数据平面里两个部署sidecar的微服务中，它们之间通信的所有流量都要被sidecar拦截，经过预处理之后转发给对应的应用逻辑：

![](https://i.loli.net/2019/06/24/5d1063f1532fc96659.jpg)

这基本就是Istio的设计原型，只不过Istio默认使用[Envoy](https://www.envoyproxy.io/)作为sidecar代理（Istio利用了Envoy内建的大量特性，例如服务发现与负载均衡、流量拆分、故障注入、熔断器以及分阶段发布等功能），而在控制层面Istio也分为四个主要模块：

1. Pilot: 为Envoy sidecar提供服务发现功能，为智能路由（例如A/B测试、金丝雀部署等）和弹性（超时、重试、熔断器等）提供流量管理功能；它将控制流量行为的高级路由规则转换为特定于Envoy的配置，并在运行时将它们传播到sidecar；

2. Mixer: 负责在服务网格上执行访问控制和使用策略，并从Envoy代理和其他服务收集遥测数据；代理提取请求级属性，发送到Mixer进行评估；

3. Citadel: 通过内置身份和凭证管理赋能强大的服务间和最终用户身份验证；可用于升级服务网格中未加密的流量，并为运维人员提供基于服务标识而不是网络控制的强制执行策略的能力；

4. Galley: 用来验证用户编写的Istio API配置；未来的版本中Galley将接管Istio获取配置、 处理和分配组件的顶级责任，从而将其他的Istio组件与从底层平台（例如Kubernetes）获取用户配置的细节中隔离开来；

最终，Istio的架构设计如下图所示：

![](https://istio.io/docs/concepts/what-is-istio/arch.svg)

### Istio快速上手

**部署Istio控制平面**

> Note: Istio可以用来运行在多种不同的环境里面，包括Kubernetes、Docker、Consul甚至BareMental（物理机上），文本以Kubernetes环境为例来演示Istio的多种特性。

在部署使用Istio之前，我们需要准备一个Kubernetes的环境，如果只是试用Istio的话，推荐试用[Minikube](https://github.com/kubernetes/minikube)或者[Kind](https://github.com/kubernetes-sigs/kind)来快速搭建Kubernetes环境。

有了Kubernetes环境并且配置好kubectl之后，我们就可以正式开始了。

1. 下载并解压最新的Istio发布版本（截至发布本文时的最新Istio版为为1.2.0）

```

root@mcdev1:~# curl -L https://git.io/getLatestIstio | ISTIO_VERSION=1.2.0 sh -
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
100  2140  100  2140    0     0   1787      0  0:00:01  0:00:01 --:--:--  1787
Downloading istio-1.2.0 from https://github.com/istio/istio/releases/download/1.2.0/istio-1.2.0-linux.tar.gz ...  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   614    0   614    0     0   1130      0 --:--:-- --:--:-- --:--:--  1132
100 20.2M  100 20.2M    0     0  3766k      0  0:00:05  0:00:05 --:--:-- 4632k
Istio 1.2.0 Download Complete!

Istio has been successfully downloaded into the istio-1.2.0 folder on your system.

Next Steps:
See https://istio.io/docs/setup/kubernetes/install/ to add Istio to your Kubernetes cluster.

To configure the istioctl client tool for your workstation,
add the /root/istio-1.2.0/bin directory to your environment path variable with:
	 export PATH="$PATH:/root/istio-1.2.0/bin"

Begin the Istio pre-installation verification check by running:
	 istioctl verify-install

Need more information? Visit https://istio.io/docs/setup/kubernetes/install/
```

2. 将`istioctl`命令行工具加入`PATH`环境变量，并且验证k8s集群环境满足Istio部署条件：

```
root@mcdev1:~# export PATH="$PATH:/root/istio-1.2.0/bin"
root@mcdev1:~# istioctl verify-install

Checking the cluster to make sure it is ready for Istio installation...

Kubernetes-api
-----------------------
Can initialize the Kubernetes client.
Can query the Kubernetes API Server.

Kubernetes-version
-----------------------
Istio is compatible with Kubernetes: v1.14.1.


Istio-existence
-----------------------
Istio will be installed in the istio-system namespace.

Kubernetes-setup
-----------------------
Can create necessary Kubernetes configurations: Namespace,ClusterRole,ClusterRoleBinding,CustomResourceDefinition,Role,ServiceAccount,Service,Deployments,ConfigMap.

SideCar-Injector
-----------------------
This Kubernetes cluster supports automatic sidecar injection. To enable automatic sidecar injection see https://istio.io/docs/setup/kubernetes/additional-setup/sidecar-injection/#deploying-an-app

-----------------------
Install Pre-Check passed! The cluster is ready for Istio installation.
```

3. 安装部署Istio控制平面：

> Note: 本文为了演示，使用最基本的Istio安装配置；如果需要定制Istio的安装配置或者在生产环境中使用Istio，请使用[helm](https://github.com/helm/helm)来定制化安装配置，详情请移步：https://istio.io/docs/setup/kubernetes/install/helm/

```
root@mcdev1:~# kubectl apply -f istio-1.2.0/install/kubernetes/istio-demo.yaml
```

4. 验证Istio控制平面安装成功：

> Note: 本文使用的最基本的Istio安装配置除了会部署Istio控制层面的核心组件Pilot、Mixer(Policy+Telemetry)、Citadel和Galley之外，还会部署Sidecar-injector（用于给应用自动注入envoy sidecar）、ingressgateway、egressgateway以及一些addons，如Prometheus、Grafana、Kiali、Jaeger.

```
root@mcdev1:~# kubectl -n istio-system get pod
NAME                                      READY   STATUS      RESTARTS   AGE
grafana-6fb9f8c5c7-bg8jd                  1/1     Running     0          8m2s
istio-citadel-7695fc84d9-jhr6s            1/1     Running     0          8m2s
istio-cleanup-secrets-1.2.0-cbjjs         0/1     Completed   0          8m3s
istio-egressgateway-687f8559cb-tn24g      1/1     Running     0          8m2s
istio-galley-75466f5dc7-ms8mm             1/1     Running     0          8m2s
istio-grafana-post-install-1.2.0-jls4k    0/1     Completed   0          8m3s
istio-ingressgateway-5d8d989c76-trpkn     1/1     Running     0          8m2s
istio-pilot-5465879d79-swpz2              2/2     Running     0          8m2s
istio-policy-6448f68f75-9b5lk             2/2     Running     5          8m2s
istio-security-post-install-1.2.0-xtxjv   0/1     Completed   0          8m3s
istio-sidecar-injector-6f4c67c6cd-fgj6c   1/1     Running     0          8m1s
istio-telemetry-56b9b87478-sbk5p          2/2     Running     6          8m2s
istio-tracing-5d8f57c8ff-nx72r            1/1     Running     0          8m1s
kiali-7d749f9dcb-wlmrt                    1/1     Running     0          8m2s
prometheus-776fdf7479-g2dgg               1/1     Running     0          8m2s
```

**部署Istio数据平面**

前面提到过Istio数据平面其实就是由一些与微服务一起部署的sidecar组成，因此，在这里我们以Istio官方提供的样例程序[bookinfo](https://istio.io/docs/examples/bookinfo/)为例来介绍Istio数据平面的部署。

Bookinfo样例程序模仿在线书店的一个分类，显示一本书的信息，页面上会显示一本书的描述，书籍的细节（ISBN、页数等），以及关于这本书的一些评论。它由四个单独的微服务构成，可谓“麻雀虽小，五脏俱全”，bookinfo例子用来演示多种Istio特性。

- `productpage`: `productpage`微服务会调用`details`和`reviews`两个微服务，用来生成页面
- `details`: 这个微服务包含了书籍的信息
- `reviews`: 这个微服务包含了书籍相关的评论。它还会调用`ratings`微服务
- `ratings`: `ratings`微服务中包含了由书籍评价组成的评级信息

`reviews`微服务有3个版本：

`v1`版本不会调用`ratings`服务；
`v2`版本会调用`ratings`服务，并使用1到5个黑色星形图标来显示评分信息；
`v3`版本会调用`ratings`服务，并使用1到5个红色星形图标来显示评分信息；

下图展示了这个应用的端到端架构：

![](https://istio.io/docs/examples/bookinfo/noistio.svg)

可以看到bookinfo是一个异构应用，几个微服务是由不同的语言编写的，而且这些服务对Istio并无依赖。

要使用Istio管理bookinfo，无需对应用做出任何改变，只需要把envoy sidecar注入到每个微服务之中。最终部署结构如下图所示：

![](https://istio.io/docs/examples/bookinfo/withistio.svg)

1. 在Kubernetes环境里，有两种方式注入sidecar：手动注入和自动注入

- 手动注入使用istio命令行工具`istioctl`

```
root@mcdev1:~# kubectl apply -f <(istioctl kube-inject -f istio-1.2.0/samples/bookinfo/platform/kube/bookinfo.yaml)
```

- 自动注入依赖于`istio-sidecar-injector`（使用[MutatingWebhook准入控制器](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#what-are-admission-webhooks)来实现），需要部署应用前给对应的namespace打上label(`istio-injection=enabled`)

```
root@mcdev1:~# kubectl label ns default istio-injection=enabled
root@mcdev1:~# kubectl apply -f istio-1.2.0/samples/bookinfo/platform/kube/bookinfo.yaml
```

2. 确认bookinfo所有的服务和Pod都已经正确定义和启动并且sidecar也被注入：

```
root@mcdev1:~# kubectl get svc
NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
details       ClusterIP   10.108.10.87    <none>        9080/TCP   2m42s
kubernetes    ClusterIP   10.96.0.1       <none>        443/TCP    51m
productpage   ClusterIP   10.103.98.1     <none>        9080/TCP   2m42s
ratings       ClusterIP   10.104.90.24    <none>        9080/TCP   2m42s
reviews       ClusterIP   10.100.151.78   <none>        9080/TCP   2m42s
root@mcdev1:~# kubectl get pod
NAME                              READY   STATUS    RESTARTS   AGE
details-v1-5544dc4896-swbgq       2/2     Running   0          2m44s
productpage-v1-7868c48878-l9zpd   2/2     Running   0          2m45s
ratings-v1-858fb7569b-f2hqh       2/2     Running   0          2m45s
reviews-v1-796d4c54d7-8r8b4       2/2     Running   0          2m45s
reviews-v2-5d5d57db85-46bmc       2/2     Running   0          2m45s
reviews-v3-77c6b4bdff-h4tgq       2/2     Running   0          2m45s
root@mcdev1:~# kubectl get pod -o jsonpath="{.items[*].spec.containers[*].name}"
details istio-proxy productpage istio-proxy ratings istio-proxy reviews istio-proxy reviews istio-proxy reviews istio-proxy
```

3. 既然bookinfo应用已经启动并在运行中，我们需要使应用程序可以从外部访问（例如使用浏览器）；这时候需要部署一个[Istio Gateway](https://istio.io/docs/reference/config/networking/v1alpha3/gateway/)来充当外部网关来作为访问应用的流量入口：

```
root@mcdev1:~# kubectl create -f istio-1.2.0/samples/bookinfo/networking/bookinfo-gateway.yaml
gateway.networking.istio.io/bookinfo-gateway created
virtualservice.networking.istio.io/bookinfo created
```

4. 确认网关的地址以及路径来访问bookinfo应用：

> Note: 如果Istio部署在使用Kind创建的kubernetes集群中，需要kubectl做端口转发；这里我们使用`任意一个Node的IP地址`+`ingressgateway的NodePort`来访问bookinfo应用。

```
root@mcdev1:~# export INGRESS_HOST=<node_ip>
root@mcdev1:~# export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
root@mcdev1:~# export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
root@mcdev1:~# curl -s http://${GATEWAY_URL}/productpage | grep -o "<title>.*</title>"
<title>Simple Bookstore App</title>
```

> Note: 这时候也可以在浏览器中访问`http://${GATEWAY_URL}/productpage`，如果一切正常，多次刷新浏览器页面，我们会发现review部分会随机的出现无星、黑星和红星，这是因为我们目前没有创建任何流量规则，所以使用默认的随机(round robin)模式在不同的reviews版本之间切换。

5. 创建默认的destination rules；在使用流量规则控制访问的review版本之前，我们需要创建一系列的destination rules用来定义每个版本对应的subset：

```
root@mcdev1:~# kubectl apply -f istio-1.2.0/samples/bookinfo/networking/destination-rule-all.yaml
destinationrule.networking.istio.io/productpage created
destinationrule.networking.istio.io/reviews created
destinationrule.networking.istio.io/ratings created
destinationrule.networking.istio.io/details created
```

6. 现在我们开始创建第一条流量规则-让所有访问reviews服务的流量只访问v1版本（无星）：

```
root@mcdev1:~# grep -B 3 -A 1 reviews istio-1.2.0/samples/bookinfo/networking/virtual-service-all-v1.yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
root@mcdev1:~# kubectl apply -f istio-1.2.0/samples/bookinfo/networking/virtual-service-all-v1.yaml
virtualservice.networking.istio.io/productpage created
virtualservice.networking.istio.io/reviews created
virtualservice.networking.istio.io/ratings created
virtualservice.networking.istio.io/details created
```

> Note: 上面创建的这条流量规则的意思是我们让所有对review服务的访问都指向v1版本，这时候在浏览器中访问`http://${GATEWAY_URL}/productpage`，如果一切正常，多次刷新浏览器页面，我们会发现review部分总是出现无星版本，这说明我们的流量规则生效了。

7. 接下来，我们来创建一条流量规则来实现review微服务的金丝雀发布：

```
root@mcdev1:~# cat istio-1.2.0/samples/bookinfo/networking/virtual-service-reviews-80-20.yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 80
    - destination:
        host: reviews
        subset: v2
      weight: 20
root@mcdev1:~# kubectl apply -f istio-1.2.0/samples/bookinfo/networking/virtual-service-reviews-80-20.yaml
virtualservice.networking.istio.io/reviews configured
```

> Note: 上面创建的这条流量规则的意思是我们让所有对review服务的访问流量中80%指向v1，20%指向v2，这时候在浏览器中访问`http://${GATEWAY_URL}/productpage`，如果一切正常，多次刷新浏览器页面，我们会发现页面中review部分大约有20%的几率会看到页面中出带黑星的评价内容，刷新次数越多，结果越准确。

> Note: Istio官方文档包括更复杂的例子来说名Istio的各种功能，如果想进一步了解，请移步：https://istio.io/docs/tasks/traffic-management/

8. 使用Jaeger来对bookinfo应用做分布式跟踪：

```
root@mcdev1:~# kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=jaeger -o jsonpath='{.items[0].metadata.name}') 15032:16686 &
```

> Note: 为了演示方便，我们这里使用kubectl命令行工具对jaeger做端口转发，这样就可以通过`http://localhost:15032`来访问jaeger的UI

> Note: Istio默认使用样本跟踪率为`1%`，为了确保jaeger收集到分布式跟踪的数据，请使用一下脚本实现多次访问bookinfo应用：


```
root@mcdev1:~# for i in `seq 1 100`; do curl -s -o /dev/null http://$GATEWAY_URL/productpage; done
```

> 然后在jaeger仪表盘的左边区域的服务列表里面选择`productpage.default`，然后点击`Find Traces`，一切正常的话，jaeger的UI会显示最新的20条分布式跟踪的记录数据。

![](https://istio.io/docs/tasks/telemetry/distributed-tracing/jaeger/istio-tracing-list.png)

> 随机选择一条分布式跟踪的记录打开，我们会发现每一条jaeger的分布式跟踪链（称为一个trace）是有多个不同的单独请求（称为一个span）组成的树状结构；同时我们也可以打开每个单独的请求来查看请求的状态以及其他的元数据。

![](https://istio.io/docs/tasks/telemetry/distributed-tracing/jaeger/istio-tracing-details.png)

9. 使用Grafana来可视化指标度量：

```
root@mcdev1:~# kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=grafana -o jsonpath='{.items[0].metadata.name}') 3000:3000 &
```
> Note: 为了演示方便，这里使用kubectl命令行工具对Grafana做端口转发，这样就可以通过`http://localhost:3000/dashboard/db/istio-mesh-dashboard`访问Grafana的UI来查看网格的全局视图以及网格中的服务和工作负载。

![](https://istio.io/docs/tasks/telemetry/metrics/using-istio-dashboard/dashboard-with-traffic.png)

> 除了查看网格的全局视图以及网格中的服务和工作负载外，Grafana还可以查看Istio控制层面主要组件Pilot，Galley等运行状况。

10. 使用Kiali来做网格整体可视化：

```
root@mcdev1:~# kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=kiali -o jsonpath='{.items[0].metadata.name}') 20001:20001 &
```

> Note: 为了演示方便，我们这里使用kubectl命令行工具对Kiali做端口转发，现在我们可以通过`http://localhost:2001/kiali/`访问Kiali的UI，默认用户名和密码都是`admin`。

> 登录Kiali的UI之后会显示Overview页面，这里可以浏览整个服务网格的概况：

![](https://istio.io/docs/tasks/telemetry/kiali/kiali-overview.png)

> 要查看指定bookinfo的整体服务图，可以点击`default`命名空间卡片，会显示类似于下面的页面，甚至可以动态显示流量走向：

![](https://istio.io/docs/tasks/telemetry/kiali/kiali-graph.png)

### 小结

至此，我们这篇文章中介绍了Istio设计的初衷以及整体架构，也在一个kubernetes环境中部署了Istio的控制层面，并以一个简单明了的样例程序bookinfo展示了Istio数据平面的组成以及创建简单的流量规则来实现控制访问的微服务版本，最后，还展示了使用Istio的一些addon来做微服务的分布式调用链跟踪和可视化整个服务网格的整体状况。这篇文章只是帮助我们快速上手Istio，关于Istio各方面的细节，后续文章我们一一解读。