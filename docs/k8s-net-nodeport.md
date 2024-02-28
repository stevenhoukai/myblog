# k8s网络 - NodePort vs LoadBalancer vs Ingress

## 前言
在上一篇中，我们学习了K8s的4层网络栈中的第2层Service网路。有了Service网络，K8s集群内的应用可以通过服务名/ClusterIP进行统一寻址和访问，而不需要关心应用集群中到底有多少个Pods，Pod的IP是什么，会不会变化，以及如何以负载均衡方式去访问等问题。但是，K8s的Service网络只是一个集群内部网络，集群外部是无法直接访问的。而我们发布的应用，有些是需要暴露出去，要让外网甚至公网能够访问的，这样才能对外输出业务价值。K8s如何将内部服务暴露出去？这是本文要重点讲解的问题。

![](https://github.com/stevenhoukai/myblog/blob/main/images/net-pod-1.jpg)


在讲到K8s如何接入外部流量的时候，会经常会听到NodePort，LoadBalancer和Ingress等概念，这些概念都是和K8s外部流量接入相关的，它们既是不同概念，同时又有关联性。下面我们分别解释这些概念和它们之间的

## NodePort
先提前强调一下，NodePort是K8s将内部服务对外暴露的基础，后面的LoadBalancer底层也是依赖于NodePort的。

之前我们学习了K8s网络的4层抽象，Service网络在第2层，节点网络在第0层。实际上，只有节点网络是可以直接对外暴露的，具体暴露方式要看数据中心或公有云的底层网络部署，但不管采用何种部署，节点网络对外暴露是完全没有问题的。那么现在的问题是，第2层的Service网络如何通过第0层的节点网络暴露出去？我们可以回看一下K8s服务发现的原理图，如下图所示，然后不妨思考一下，K8s集群中有哪一个角色，即掌握Service网络的所有信息，可以和Service网络以及Pod网络互通互联，同时又可以和节点网络打通？

![](https://github.com/stevenhoukai/myblog/blob/main/images/service-4.jpg)


答案是Kube-Proxy。上一篇我们提到Kube-Proxy是K8s内部服务发现的一个关键组件，事实上，它还是K8s将内部服务暴露出去的关键组件。Kube-Proxy在K8s集群中所有Worker节点上都部署有一个，它掌握Service网络的所有信息，知道怎么和Service网络以及Pod网络互通互联。如果要将Kube-Proxy和节点网络打通(从而将某个服务通过Kube-Proxy暴露出去)，只需要让Kube-Proxy在节点上暴露一个监听端口即可。这种通过Kube-Proxy在节点上暴露一个监听端口，将K8s内部服务通过Kube-Proxy暴露出去的方式，术语就叫NodePort(顾名思义，端口暴露在节点上)。下图是通过NodePort暴露服务的简化概念模型。

![](https://github.com/stevenhoukai/myblog/blob/main/images/nport-1.jpg)

如果我们要将K8s内部的一个服务通过NodePort方式暴露出去，可以将服务发布(kind: Service)的type设定为NodePort，同时指定一个30000~32767范围内的端口。服务发布以后，K8s在每个Worker节点上都会开启这个监听端口。这个端口的背后是Kube-Proxy，当K8s外部有Client要访问K8s集群内的某个服务，它通过这个服务的NodePort端口发起调用，这个调用通过Kube-Proxy转发到内部的Servcie抽象层，然后再转发到目标Pod上。Kube-Proxy转发以及之后的环节，可以和上一篇的内容对接起来。注意，为了直观形象，上图的Service在K8s集群中被画成一个独立组件，实际是没有独立Service这样一个组件的，只是一个抽象概念，如果要理解这个抽象的底层实现细节，可以回看一下K8s服务发现原理。

## LoadBalancer
上面我们提到，将K8s内部的服务通过NodePort方式暴露出去，K8s会在每个Worker节点上都开启对应的NodePort端口。逻辑上看，K8s集群中的所有节点都会暴露这个服务，或者说这个服务是以集群方式暴露的(实际支持这个服务的Pod可能就分布在其中有限几个节点上，但是因为所有节点上都有Kube-Proxy，所以所有节点都知道该如何转发)。既然是集群，就会涉及负载均衡问题，谁负责对这个服务的负载均衡访问？答案是需要引入负载均衡器(Load Balancer)。下图是通过LoadBalancer，将服务对外暴露的概念模型。

![](https://github.com/stevenhoukai/myblog/blob/main/images/nport-2.jpg)

假设我们有一套阿里云K8s环境，要将K8s内部的一个服务通过LoadBalancer方式暴露出去，可以将服务发布(Kind: Service)的type设定为LoadBalancer。服务发布后，阿里云K8s不仅会自动创建服务的NodePort端口转发，同时会自动帮我们申请一个SLB，有独立公网IP，并且阿里云K8s会帮我们自动把SLB映射到后台K8s集群的对应NodePort上。这样，通过SLB的公网IP，我们就可以访问到K8s内部服务，阿里云SLB负载均衡器会在背后做负载均衡。

值得一提的是，如果是在本地开发测试环境里头搭建的K8s，一般不支持Load Balancer也没必要，因为通过NodePort做测试访问就够了。但是在生产环境或者公有云上的K8s，例如GCP或者阿里云K8s，基本都支持自动创建Load Balancer。

## Ingress
有了前面的NodePort + LoadBalancer，将K8s内部服务暴露到外网甚至公网的需求就已经实现了，那么为啥还要引入Ingress这样一个概念呢？它起什么作用？

我们知道在公有云(阿里云/AWS/GCP)上，公网LB+IP是需要花钱买的。我们回看上图的通过LoadBalancer(简称LB)暴露服务的方式，发现要暴露一个服务就需要购买一个独立的LB+IP，如果要暴露十个服务就需要购买十个LB+IP，显然，从成本考虑这是不划算也不可扩展的。那么，有没有办法只需购买一个(或者少量)的LB+IP，但是却可以按需暴露更多服务出去呢？答案其实不复杂，就是想办法在K8内部部署一个独立的反向代理服务，让它做代理转发。谷歌给这个内部独立部署的反向代理服务起了一个奇怪的名字，就叫Ingress，它的简化概念模型如下图所示：
![](https://github.com/stevenhoukai/myblog/blob/main/images/nport-3.jpg)



Ingress就是一个特殊的Service，通过节点的**HostPort(80/443)**暴露出去，前置一般也有LB做负载均衡。Ingress转发到内部的其它服务，是通过集群内的Service抽象层/ClusterIP进行转发，最终转发到目标服务Pod上。Ingress的转发可以基于Path转发，也可以基于域名转发等方式，基本上你只需给它设置好转发路由表即可，功能和Nginx无本质差别。

注意，上图的Ingress概念模型是一种更抽象的画法，隐去了K8s集群中的节点，实际HostPort是暴露在节点上的。

所以，Ingress并不是什么神奇的东西，首先，它本质上就是K8s集群中的一个比较特殊的Service(发布Kind: Ingress)。其次，这个Service提供的功能主要就是7层反向代理(也可以提供安全认证，监控，限流和SSL证书等高级功能)，功能类似Nginx。第三，这个Service对外暴露出去是通过HostPort(80/443)，可以和上面LoadBalancer对接起来。有了这个Ingress Service，我们可以做到只需购买一个LB+IP，就可以通过Ingress将内部多个(甚至全部)服务暴露出去，Ingress会帮忙做代理转发。

那么哪些软件可以做这个Ingress？传统的Nginx/Haproxy可以，现代的微服务网关Zuul/SpringCloudGateway/Kong/Envoy/Traefik等等都可以。当然，谷歌别出心裁给这个东东起名叫Ingress，它还是做了一些包装，以简化对Ingress的操作。如果你理解了原理，那么完全可以用Zuul或者SpringCloudGateway，或者自己定制开发一个反向代理，来替代这个Ingress。部署的时候以普通Service部署，将type设定为LoadBalancer即可，如下图所示：

![](https://github.com/stevenhoukai/myblog/blob/main/images/nport-4.jpg)

注意，Ingress是一个7层反向代理，如果你要暴露的是4层服务，还是需要走独立LB+IP方式。

## Kubectl Proxy & Port Forward
上面提到的服务暴露方案，包括NodePort/LoadBalancer/Ingress，主要针对正式生产环境。如果在本地开发测试环境，需要对本地部署的K8s环境中的服务或者Pod进行快速调试或测试，还有几种简易办法，这边一并简单介绍下

### 办法一
通过kubectl proxy命令，在本机上开启一个代理服务，通过这个代理服务，可以访问K8s集群内的任意服务。背后，这个Kubectl代理服务通过Master上的API Server间接访问K8s集群内服务，因为Master知道集群内所有服务信息。这种方式只限于7层HTTP转发。
### 办法二
通过kubectl port-forward命令，它可以在本机上开启一个转发端口，间接转发到K8s内部的某个Pod的端口上。这样我们通过本机端口就可以访问K8s集群内的某个Pod。这种方式是TCP转发，不限于HTTP。
### 办法三
通过kubectl exec命令直接连到Pod上去执行linux命令，功能类似docker exec。

## 总结
NodePort是K8s内部服务对外暴露的基础，LoadBalancer底层有赖于NodePort。NodePort背后是Kube-Proxy，Kube-Proxy是沟通Service网络、Pod网络和节点网络的桥梁。
将K8s服务通过NodePort对外暴露是以集群方式暴露的，每个节点上都会暴露相应的NodePort，通过LoadBalancer可以实现负载均衡访问。公有云(如阿里云/AWS/GCP)提供的K8s，都支持自动部署LB，且提供公网可访问IP，LB背后对接NodePort。
Ingress扮演的角色是对K8s内部服务进行集中反向代理，通过Ingress，我们可以同时对外暴露K8s内部的多个服务，但是只需要购买1个(或者少量)LB。Ingress本质也是一种K8s的特殊Service，它也通过HostPort(80/443)对外暴露。
通过Kubectl Proxy或者Port Forward，可以在本地环境快速调试访问K8s中的服务或Pod。
K8s的Service发布主要有3种type，type=ClusterIP，表示仅内部可访问服务，type=NodePort，表示通过NodePort对外暴露服务，type=LoadBalancer，表示通过LoadBalancer对外暴露服务(底层对接NodePort，一般公有云才支持)。
至此，Kubernetes网络三部曲全部讲解完成，希望这三篇文章对你理解和用好K8s有帮助。下表是对三部曲的浓缩总结：
下表是核心组件以及其对应作用
|      | 作用 | 实现     |
| :---        |    :----:   |          :---: |
| 结点网络      | Master/Worker节点之间网络互通       | 交换机、路由器、网卡   |
| Pod网络   | pod虚拟机之间互通        | 虚拟网卡，虚拟网桥，网卡，路由器or覆盖网络 |
| Service网络   | 服务发现+负载均衡        | Kube-proxy，Kubelet，Master，Kube-DNS |
| NodePort   | 将Service暴露在节点网络上        | Kube-proxy |
| LoadBalancer   | 将Service暴露在公网+负载均衡        | 公有云LB + NodePort |
| Ingress   | 反向路由，安全，日志监控        | Nginx/Encoy/Traefik/Zuul/SpringCloudGateway |
