前言
理解K8s的架构是运用好K8s的基础，本文波波帮助大家梳理一下K8s的架构。我们先会对K8s的架构进行一个概览，然后分别剖析Master和Worker节点的组件构成，然后把这些组件再集成起来，通过一个发布样例展示这些组件是如何配合工作的，最后展示K8s集群的总体架构。

架构概览


上图是K8s架构的概览。K8s集群中主要有两类角色，一类是Master节点，另外一类是Worker节点，简单讲，Master节点主要用来管理和调度集群资源的，而Worker节点则是提供资源的。在一个高可用的K8s集群中，Master和Worker一般都有多个节点构成，这些节点可以是物理机，也可以是虚拟机。

Worker节点提供的资源单位称为Pod，简单理解，Pod就是K8s云平台提供的虚拟机。Pod里头住的是应用容器，比如Docker容器，容器是CPU/Mem资源隔离单位。大部分场景下，一个Pod只住一个应用容器，但是也有一些场景，一个Pod里头可以住多个容器，其中一个是主容器，其它则是辅助容器。一个Pod里头的容器共享Pod的网络栈和存储资源。

K8s主要解决集群资源调度的问题。简单讲，就是当有应用发布请求过来的时候，K8s需要根据集群资源空闲现状，将这个应用的Pods合理的分配到空闲的Worker节点上去。同时，K8s需要时刻监控集群，如果有节点或者Pods挂了，它要能够重新协调和启动Pods，保证应用高可用，这个术语叫自愈。还有，K8s需要管理集群网络，保证Pod/服务之间可以互通互联。

Master节点组件


Master节点是K8s集群大脑，它由如下组件构成：

Etcd： 它是K8s的集中状态存储，所有的集群状态数据，例如节点，Pods，发布，配置等等，最终都存储在Etcd中。Etcd是一个分布式KV数据库，采用Raft分布式一致性算法。Etcd高可用部署一般需要至少三个节点。Etcd集群可以独立部署，也可以和Master节点住在一起。
API server： 它是K8s集群的接口和通讯总线。用户通过kubectl，dashboard或者sdk等方式操作K8s，背后都通过API server和集群进行交互。集群内的其它组件，例如Kubelet/Kube-Proxy/Scheduler/Controller-Manager等，都通过API server和集群进行交互。API server可以认为是Etcd的一个代理Proxy，它是唯一能够访问操作Etcd数据库的组件，其它组件，都必须通过API server间接操作Etcd。API server不仅接受其它组件的API请求，它还是集群的事件总线，其它组件可以订阅在API server上，当有新事件发生时候，API server会将相关事件通知到感兴趣的组件。
Scheduler： 它是K8s集群负责调度决策的组件。Scheduler掌握当前的集群资源使用情况，当有新的应用发布请求被提交到K8s集群，它负责决策相应的Pods应该分布到哪些空闲节点上去。K8s中的调度决策算法是可以扩展的。
Controller Manager： 它是保证集群状态最终一致的组件。它通过API server监控集群状态，确保实际状态和预期状态最终一致，如果一个应用要求发布十个Pods，Controller Manager保证这个应用最终启动十个Pods，如果中间有Pods挂了，Controller Manager会负责协调重启Pods，如果Pods启多了，Controller Manager会负责协调关闭多余Pods。也即是说，K8s采用最终一致调度策略，它是集群自愈的背后实现机制。
Worker节点组件


Worker节点是K8s集群资源的提供者，它由如下组件构成：

Kubelet: 它是Worker节点资源的管理者，相当于一个Agent角色。它监听API server的事件，根据Master节点的指示启动或者关闭Pod等资源，也将本节点状态数据汇报给Master节点。如果说Master节点是K8s集群的大脑，那么Kubelet就是Worker节点的小脑。
Container Runtime: 它是节点容器资源的管理者，如果采用Docker容器，那么它就是Docker Engine。Kubelet并不直接管理节点的容器资源，它委托Container Runtime进行管理，比如启动或者关闭容器，收集容器状态等。Container Runtime在启动容器时，如果本地没有镜像缓存，则需要到Docker Registry(或Docker Hub)去拉取相应镜像，然后缓存本地。
Kube-Proxy: 它是管理K8s中的服务(Service)网络的组件。Pod在K8s中是ephemeral的概念，也就是不固定的，PodIP可能会变(包括预期和非预期的)。为了屏蔽PodIP的可能的变化，K8s中引入了Servie概念，它可以屏蔽应用的PodIP，并且在调用时进行负载均衡。Kube-Proxy是实现K8s服务(Service)网络的背后机制。另外，当需要把K8s中的服务(Service)暴露给外网时，也需要通过Kube-Proxy进行代理转发。
流程样例


如果我们把Master节点和Worker节点集成起来，就构成上图所示的K8s集群。下面我们通过一个发布流程，展示上面介绍的这些组件是如何配合工作的。

假设管理员要发布一个新应用，他通过Kubectl命令行工具将发布请求提交到API server，API server将请求存储到Etcd数据库中。
Scheduler通过API server监听到有新的应用发布请求，它通过调度算法决策，选择若干可发布的空闲节点，并将发布决策更新到API server。
被选中的Worker节点上的Kubelet通过API server监听到有给自己的新发布任务，它根据任务指示在本地启动相应的Pods(间接通过Container Runtime启动容器)，并将任务执行成功情况报告给API server。
所有Worker节点上Kube-Proxy通过API server监听到有新的发布，它获取应用的PodIP/ClusterIP/端口等相关数据，更新本地的iptables表规则，让本地的Pods可以通过iptables转发方式，访问到新发布应用的Pods。
Controller Manager通过API server，时刻监控新发应用的健康状况，保证实际状态和预期状态最终一致。
总体架构


上图是一个K8s集群的总体架构。实际K8s集群中还有一个覆盖(Overlay)网络，集群中的Pods通过覆盖网络可以实现IP寻址和通讯。实现覆盖网络的技术有很多，例如Flannel/VxLan/Calico/Weave-Net等等。外网流量如果要访问K8s集群内部的服务，一般要走负载均衡器(Load Balancer)，背后流量会通过Kube-Proxy间接转发到服务Pods上。

除了上述组件，K8s外围一般还有存储，监控，日志和分析等配套支持服务。

总结和课程推荐
下表我把本文讲到的一些K8s关键组件的作用做一个梳理总结，方便大家理解记忆。
————————————————

                            版权声明：本文为博主原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接和本声明。
                        
原文链接：https://blog.csdn.net/yang75108/article/details/100215486