---
title: "使用 Telepresence 在本地调试 Kubernetes 微服务"
date: 2019-07-24T23:10:21+08:00
lastmod: 2019-11-11T23:07:23+08:00
keywords: ["Telepresence", "Kubernetes", "K8S", "microservices", "microservices development"]
---

微服务作为一种全新的软件架构现在正变得越来越火。基本原因我觉得有两点：一方面软件系统越做越复杂，通过拆分将一个大系统解耦成一个个独立的子系统，我们就降低了整个系统的复杂性。另一方面，[Kubernetes](https://zh.wikipedia.org/wiki/Kubernetes) 的出现使得编排这么多子系统变得简单，可以说 Kubernetes 是目前为止微服务最好的载体。

Kubernetes 解决了微服务运行时的环境问题，但对开发环境就不那么友好了。比方说如果我们要在本地开发调试一个服务A，但服务A可能依赖服务B、C，而服务B又有一层依赖D，我们就需要在本地把服务B、C、D都搭建起来才能调试服务A。这显然是一个很痛苦的过程。

![Microservices Dependency Hell](/image/telepresence/dependency-hell.svg)

业界有朋友用 `docker-compose` 来模拟集群中的场景。这个方案的不足之处在于它需要把 Kubernetes 的那一套逻辑用 `docker-compose.yml` 文件重写一遍，这给我们带来了维护成本。另一方面，本地机器很可能不具备某些微服务所依赖的资源。

```yaml
ratesvc:
  image: kubeapps/ratesvc:latest
  environment:
    - JWT_KEY=secret  # <------------------------ 手工维护
  command:
    - /ratesvc
    - --mongo-url=mongodb://root@mongodb  # <---- 手工维护
    - --mongo-database=ratesvc

mongodb:
  image: bitnami/mongodb:3
  environment:
    - MONGODB_ROOT_PASSWORD=password123

auth:
  image: kubeapps/oauth2-bitnami:latest
  volumes:
    - ./config.yaml:/config/monocular.yaml  # <-- 手工维护
  ...

volumnes:  # <----------------------------------- 手工维护
  monocular-data:
```

另一种解决方案就是我这里要介绍的 Telepresence 了，它能够在不修改程序代码的情况下，让本地应用程序无感的接入到 Kubernetes 集群中，这样我们就可以直接在本地开发调试微服务了。

## 简介

[Telepresence](https://www.telepresence.io/) 是一个 CNCF [^CNCF] 基金会下的项目。它的工作原理是在本地和 Kubernetes 集群中搭建一个透明的双向代理，这使得我们可以在本地用熟悉的 IDE 和调试工具来运行一个微服务，同时该服务还可以无缝的与 Kubernetes 集群中的其他服务进行交互，好像它就运行在这个集群中一样。

这是一个 Telepresence 工作原理图，它将集群中的数据卷、环境变量、网络都代理到了本地（除了数据卷外，其他两个对应用程序来说都是透明的）：

![Telepresence Proxies](/image/telepresence/telepresence-proxies.svg)

有了这些代理之后：

1. 本地的服务就可以直接使用域名访问到远程集群中的其他服务
2. 本地的服务直接访问到 Kubernetes 里的各种资源，包括环境变量、secrets、config map等
3. 甚至集群中的服务还能直接访问到本地暴露出来的接口

## 安装

macOS:

```bash
brew cask install osxfuse  # required by sshfs to mount the pod's filesystem
brew install datawire/blackbird/telepresence
```

其他平台请参考：[https://www.telepresence.io/reference/install](https://www.telepresence.io/reference/install)

如果官方的安装包没有覆盖到你的平台，其实也可以从源代码安装，因为它本身就是用 Python3 写的，熟悉 Python 的朋友安装这个程序应该不难，我自己就在 CentOS 7 上安装成功了。

## 使用场景

假设我们有两个服务 A 和 B，服务 A 是依赖于服务 B 的。下面分两个场景来看看如何用 Telepresence  分别调试 A 和 B。

![Service A&B](/image/telepresence/service-a-b.svg)

### 调试服务 A - 本地与远端服务联调

服务 A 在本地运行，服务 B 运行在远端集群中。借助 Telepresence 搭建的代理，A 就能直接访问到 B。比方说我们的服务 B 是这样一个程序，它监听在8000端口上。每当有人访问时它就返回`Hello, world!`。

```bash
$ kubectl run service-b --image=datawire/hello-world --port=8000 --expose
$ kubectl get service service-b
NAME        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
service-b   10.0.0.12    <none>        8000/TCP   1m
```

现在在本地用默认参数启动 Telepresence ，等它连接好集群：

```bash
$ telepresence
T: Starting proxy with method 'vpn-tcp', which has the following limitations: All processes are affected, only one telepresence can run per machine, and you
T: can't use other VPNs. You may need to add cloud hosts and headless services with --also-proxy. For a full list of method limitations see
T: https://telepresence.io/reference/methods.html
T: Volumes are rooted at $TELEPRESENCE_ROOT. See https://telepresence.io/howto/volumes.html for details.
T: Starting network proxy to cluster using new Deployment telepresence-1566230249-7112632-14485

T: No traffic is being forwarded from the remote Deployment to your local machine. You can use the --expose option to specify which ports you want to
T: forward.

T: Setup complete. Launching your command.
@test_cluster|bash-4.2#
```

这时候就可以开始调试服务 A 了，因为服务 B 暴露出来的接口本地已经可以直接访问到：

```bash
$ curl http://service-b:8000/
Hello, world!
```

这里要说明一下这背后发生的事情：

1. 当运行 Telepresence 命令的时候，它创建了一个`Deployment`，这个`Deployment`的 Spec 是一个转发流量的代理容器，我们可以这样查看到它 `kubectl get pod -l telepresence`。
2. 同时它还在本地创建了一个全局的 VPN，使得本地的所有程序都可以访问到集群中的服务。 Telepresence 其实还支持其他的网络代理模式（使用`--method`切换），`vpn-tcp`是默认的方式，其他的好像用处不大，`inject-tcp`甚至要在后续的版本中[取消掉](https://www.youtube.com/watch?v=k9lh4ZuKsiQ)。
3. 当本地的`curl`访问`http://service-b:8000/`时，对应的 DNS 查询和 HTTP 请求都被 VPN 路由到集群中刚刚创建的容器中处理。如果域名解析不了 (Could not resolve host)，可以试试加上 search 后缀：`service-b.<NAMESPACE>.svc.cluster.local`。

新的拓扑结构为：

![Local Service A](/image/telepresence/local-service-a.svg)

除此之外 Telepresence 还将远端的文件系统通过`sshfs`挂载到本地`$TELEPRESENCE_ROOT`下面（也支持通过参数`--mount <MOUNT_PATH>`指定挂载的路径）。这样，我们的应用程序就可以在本地访问到远端的文件系统：

```bash
$ ls $TELEPRESENCE_ROOT/var/run/secrets/kubernetes.io/serviceaccount
ca.crt  namespace  token
```

如果我们退出 Telepresence 对应的 Shell，它也会做一些清理工作，比如取消本地 VPN、删除刚刚创建的`Deployment`等。

### 调试服务 B - 集群内服务与本地联调

服务 B 与刚才的不同之处在于，它是被别人访问的，要调试它，首先得要有真实的访问流量。我们如何才能做到将别人对它的访问路由到本地来，从而实现在本地捕捉到集群中的流量呢？

Telepresence 提供这样一个参数，`--swap-deployment <DEPLOYMENT_NAME[:CONTAINER]>`，用来将集群中的一个`Deployment`替换为本地的服务。对于上面的`service-b`，我们可以这样替换：

```bash
$ telepresence --swap-deployment service-b --expose 8000:8000
```

这个时候集群中的服务 A 再想访问服务 B 的8000端口时，Telepresence 就会将这个请求转发到本地的8000端口。即新的拓扑结构变成：

![Local Service B](/image/telepresence/local-service-b.svg)

它的工作原理概述如下：

1. 在集群中创建一个代理`Deployment`，并复制`service-b`的所有`Label`。
2. 建立一个路由通道，将代理容器的所有流量转发到本地 8000 端口。
3. 将`service-b`的 replicas 数设为0，这样 K8S `Service` 的 selector 就只能匹配到刚刚创建的代理容器上。

通过这样的方法，我们就有机会将集群中的请求转发到本地，然后在本地查看到具体的请求数据，调试逻辑，以及生成新的回复。

## 总结

这篇文章里我先提出了微服务开发中一个常见的问题，然后介绍了 Telepresence 项目，并且举例说明了怎样用它来调试两种常见的微服务场景。当然，Telepresence 还在不断的演进，本文中使用的是[v0.103](https://github.com/telepresenceio/telepresence/tree/0.103)版本，后续版本很可能有些不一样的地方，也欢迎大家不断指正。



[^CNCF]: Cloud Native Computing Foundation，致力于推广云原生应用，旗下的代表项目有 Kubernetes，etcd等。