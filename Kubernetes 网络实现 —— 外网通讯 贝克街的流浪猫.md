# Kubernetes 网络实现 —— 外网通讯 | 贝克街的流浪猫
[](#引言 "引言")引言
--------------

本文介绍 Kubernetes 网络中外网通讯、LoadBalance 以及 Ingress 部分的实现方案，更多关于 Kubernetes 的介绍均收录于 [<Kubernetes 系列文章>](https://www.beikejiedeliulangmao.top/categories/Container/Kubernetes) 中。

### [](#与外网通讯 "与外网通讯")与外网通讯

到目前为止，我们已经清楚了 Kubernetes 集群内部的网络通讯原理，但是我们还没有讲清楚外部世界的网络怎么和我们的 Kubernetes service 交互。这包含两个部分，集群内的数据包怎么发送到外部网络，以及外部网络的数据包怎么流入 pod。

以我现在使用的内部云服务为例，我的 Kubernetes 集群实际上运行在一个虚拟机上，这些虚拟机都被指定了一个内部 ip，也就是 eth0 的 ip 地址，这个内部地址在我们的 Kubernetes 集群中能够直接访问。为了让我们的数据包能够触及外部世界的网络，云服务商会给我们的虚拟机挂载一个网关，这个网关有两个作用：其一是为我们的虚拟机提供一个路由表的 target，通过这个 target 虚拟机的数据包可以发送给网关继而由网关转发到外部网络。其二是做一个网络地址转换（NAT），给每一个数据包指定到一个公网 ip。由于这个网关的存在，我们的虚拟机才得以访问到公网。但是有一个小问题，因为我们的 pod 有自己的网络域，它是和虚拟机上的 eth0 网络域是独立的。而网关只能负责虚拟机 eth0 网络域的 ip，因为它根本不知道虚拟机上还有 pod subnet。那么 Kubernetes 是怎么解决这个问题的呢？  
![](https://bbm-hexo.oss-accelerate.aliyuncs.com/images/kubernetes/pod-to-internet.gif)
  
在上图中，数据包由 pod 的网络空间发出，经由 veth 对，连接到宿主机的网络空间 root。当数据包到达，root 网络空间后，会从网桥流向默认网络设备 eth0，因为数据包的目标 ip 没有命中宿主机的任意网络设备。在流到 eth0 之前，别忘了它还会经过 iptables 的处理。iptables 发现这个数据包由 pod 网络中发出，如果它保持这个源 ip 不变，网关 NAT 会无法识别这个网络域，数据包就发不出去，所以在这里 iptables 扮演着一个 NAT 的角色，它将源 ip 从 pod ip 改为 eth0 的 ip。这样这个数据包才能从虚拟机中经由网关传送到公网。在网关中，它还会再次进行一次 NAT，将虚拟机的内网地址，转换成公网地址。而从公网中返回的数据包也是经历了相同的路线，NAT 将目标地址外网 ip 转为虚拟机的内网 ip，然后虚拟机将内网 ip 转为 pod 的 ip。

还记得我们在试玩环节解决的外网访问 service 的实验么，当时我们使用了云服务中的 LoadBalancer，实际上在 Kubernetes 中有两种方式可以解决外网访问我们 service 的问题，一个就是前面说的 LoadBalancer，另一个 Kubernetes 的 Ingress Controller。

LoadBalancer，又称 Layer 4 （传输层）入口。当我们创建好 service 后，可以通过云服务商的 API 来创建 LB，这个 LB 对应了一个外网的 ip 地址，外网的用户可以通过这个 LB public ip 访问到我们的服务。前面的试玩中，因为我使用的云服务没有提供 Kubernetes 对接，所以我 Service type 选为 NodePort，并将 L4 LB 绑定到 service 的 node port 上，当 Service type 选为 node port 后，会由 kube-proxy 进程在宿主机上占用一个端口 Service Port，保证该端口不会被其他进程使用，然后修改 iptables 将发送到 Service Port 的包转发到随机 pod 中，这是 kube-proxy 的默认工作模式，该模式的优点是代理过程全部经由内核态完成，而且在第四层就能完成转发工作。

让我们看一看 LB 是怎么工作起来的。首先，我们部署了自己的服务 kubia，然后创建了 LB，外网的数据包到达 LB 后会分发到任意一个 Kubernetes 集群节点，经由该虚拟机节点的 iptables，它会再次进行一个负载均衡，分发到任意一个 service 的 pod 中，当该 pod 返回数据包时，必然是用的 pod 的 ip 地址，这怎么行呢，外网的服务访问者肯定希望返回的数据包 ip 是 LB 的 ip，这样才能对的上啊。所以，当数据包从 pod 中返回时，iptables 和 conntrack 齐心协力地重写了 ip 地址，虚拟机将 ip 改为自己的 eth0 ip 并返回给 LB，LB 再将虚拟机的 ip 改为自己的外网 ip。下面这张图就展示了，这里所说的流程。  
![](https://bbm-hexo.oss-accelerate.aliyuncs.com/images/kubernetes/internet-to-service.gif)
  
除了，LB 之外，另一种方案是 Layer 7（应用层）入口。一般来说 Layer 7 入口面向的是 HTTP/HTTPS 协议栈。我们需要将 Service 类型置为 NodePort，这样 Kubernetes 会选择一个端口作为服务的名义端口，然后每个节点上的 kube-proxy 都会占用该名义端口，防止其他进程也使用相同端口（会发生传输冲突），然后每个节点都会修改自己的 iptables，将流入该名义端口的包根据负载均衡规则转发到对应的 pod 中。

在 Kubernetes 中，我们称这种 Layer 7 的 HTTP LoadBalance 为 Ingress，它同样需要云服务商提供 L7 的 http LoadBalance，就像 Layer 4 LoadBalance 一样，它只能将请求转发到虚拟机上，然后虚拟机上的 iptables 再转发到对应的 pod 中。

下图展示的就是 Layer 7 应用级 LB 的工作方式，当我们通过 Kubernetes 的 api server 建立 Ingress 对象后，云服务提供商会感知到该事件，并建立自己的应用级 LB（ALB），该 ALB 会指向多个节点来保证可用性，同时会根据 Ingress 中定义的规则将请求转发给对应的节点 NodePort（NP）。  
![](https://bbm-hexo.oss-accelerate.aliyuncs.com/images/kubernetes/ingress-controller-design.png)
  
下图中描述了数据包的流转，公网中的数据包发给 Ingress LB 后，它将请求转发到虚拟机上的 NodePort，然后虚拟机根据 iptables 规则将请求转发到正确的 pod 中。就像前面所说的一样，这里也会使用 conntrack 来将返回包的源 ip 地址从 pod ip 改为 lb 的 ip。  
![](https://bbm-hexo.oss-accelerate.aliyuncs.com/images/kubernetes/ingress-to-service.gif)

### [](#LB-vs-Ingress "LB vs Ingress")LB vs Ingress

我们已经知道了 Kubernetes 中当 service 的类型为 LoadBalance 时，会使用 L4 LB，当使用 Ingress 暴露服务时，会使用 L7 LB，那么他们有什么不同呢？

所谓四层负载均衡，也就是主要通过报文中的目标地址和端口，再加上负载均衡设备设置的服务器选择方式，决定最终选择的内部服务器。以常见的 TCP 为例，负载均衡设备在接收到第一个来自客户端的 SYN 请求时，即通过上述方式选择一个最佳的服务器，并对报文中的目标 IP 地址进行修改 (改为后端服务器 IP），直接转发给该服务器。TCP 的连接建立，即三次握手是客户端和服务器直接建立的，负载均衡设备只是起到一个类似路由器的转发动作。在某些部署情况下，为保证服务器回包可以正确返回给负载均衡设备，在转发报文的同时可能还会对报文原来的源地址进行修改，当然大部分情况下可能不会这么做，那样的话服务器的回包可能就不会经过 LB 设备。  
![](https://www.beikejiedeliulangmao.top/images/loading-cat.gif)
  
所谓七层负载均衡，也称为 “内容交换”，也就是主要通过报文中的真正有意义的应用层内容，再加上负载均衡设备设置的服务器选择方式，决定最终选择的内部服务器。以常见的 TCP　为例，负载均衡设备如果要根据真正的应用层内容再选择服务器，只能先和客户端建立连接（TCP 三次握手）后，才可能接收到客户端发送的真正应用层内容的报文，然后再根据该报文中的特定字段，再加上负载均衡设备设置的服务器选择方式，决定最终选择的内部服务器。负载均衡设备在这种情况下，更类似于一个代理服务器。负载均衡和前端的客户端以及后端的服务器会分别建立 TCP 连接。所以从这个技术原理上来看，七层负载均衡明显地对负载均衡设备的要求更高，处理七层的能力也必然会低于四层模式的部署方式。

L7 LB 的优点是它更加灵活，举个例子：因为它运行在 HTTP 协议栈中，所以它能知道 URL 信息，所以可以通过 URL 来作为转发规则，同时我们还可以通过 HTTP 的 `X-Forwarded-For` 头来获取到 HTTP 请求的 client ip。另外一个优点是更加安全，网络中最常见的 SYN Flood 攻击，即黑客控制众多源客户端，使用虚假 IP 地址对同一目标发送 SYN 攻击，通常这种攻击会大量发送 SYN 报文，耗尽服务器上的相关资源，以达到 Denial of Service (DoS) 的目的。从技术原理上也可以看出，四层模式下这些 SYN 攻击都会被转发到后端的服务器上；而七层模式下这些 SYN 攻击自然在负载均衡设备上就截止，不会影响后台服务器的正常运营。

现在的 7 层负载均衡，主要还是着重于应用广泛的 HTTP 协议，所以其应用范围主要是众多的网站或者内部信息平台等基于 B/S 开发的系统。4 层负载均衡则对应其他 TCP 应用，例如基于 C/S 开发的系统。

[](#参考内容 "参考内容")参考内容
--------------------

\[1\] kubernetes GitHub 仓库  
\[2\] Kubernetes 官方主页  
\[3\] Kubernetes 官方 Demo  
\[4\] 《Kubernetes in Action》  
\[5\] 理解 Kubernetes 网络之 Flannel 网络  
\[6\] Kubernetes Handbook  
\[7\] iptables 概念介绍及相关操作  
\[8\] iptables 超全详解  
\[9\] 理解 Docker 容器网络之 Linux Network Namespace  
\[10\] A Guide to the Kubernetes Networking Model  
\[11\] Kubernetes with Flannel — Understanding the Networking  
\[12\] 四层、七层负载均衡的区别