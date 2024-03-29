# k8s网络 - Pod网络

## 前言
K8s是一个强大的平台，但它的网络比较复杂，涉及很多概念，例如Pod网络，Service网络，Cluster IPs，NodePort，LoadBalancer和Ingress等等，只有深入理解K8s网络，才能为理解和用好K8s打下坚实基础。为了帮助大家理解，参照TCP/IP协议栈，这里把K8s的网络分解为四个抽象层，从0到3，除了第0层，每一层都是构建于前一层之上，如下图所示：

![](https://github.com/stevenhoukai/myblog/blob/main/images/net-pod-1.jpg)


第0层Node节点网络比较好理解，也就是保证K8s节点(物理或虚拟机)之间能够正常IP寻址和互通的网络，这个一般由底层(公有云或数据中心)网络基础设施支持。第0层我们假定已经存在，所以不展开。第1到3层网络，用三篇文章来分别进行解析，本文是第一篇<Pod网络>。


## Pod网络概念模型
Pod相当于是K8s云平台中的虚拟机，它是K8s的基本调度单位。所谓Pod网络，就是能够保证K8s集群中的所有Pods(包括同一节点上的，也包括不同节点上的Pods)，逻辑上看起来都在同一个平面网络内，能够相互做IP寻址和通信的网络，下图是Pod网络的简化概念模型：


![](https://github.com/stevenhoukai/myblog/blob/main/images/net-pod-2.jpg)


Pod网络构建于Node节点网络之上，它又是上层Service网络的基础。主要分为节点上的Pod之间的网络，以及不同节点上的Pod之间网络，下面分别进行剖析。

同一节点上的Pod网络
前面提到，Pod相当于是K8s云平台中的虚拟机，实际一个Pod中可以住一个或者多个(大多数场景住一个)应用容器，这些容器共享Pod的网络栈和其它资源如Volume。那么什么是共享网络栈？同一节点上的Pod之间如何寻址和互通？我以下图样例来解释：



上图节点上展示了Pod网络所依赖的3个网络设备，eth0是节点主机上的网卡，这个是支持该节点流量出入的设备，也是支持集群节点间IP寻址和互通的设备。docker0是一个虚拟网桥，可以简单理解为一个虚拟交换机，它是支持该节点上的Pod之间进行IP寻址和互通的设备。veth0则是Pod1的虚拟网卡，是支持该Pod内容器互通和对外访问的虚拟设备。docker0网桥和veth0网卡，都是linux支持和创建的虚拟网络设备。

![](https://github.com/stevenhoukai/myblog/blob/main/images/net-pod-3.jpg)

上图Pod1内部住了3个容器，它们都共享一个虚拟网卡veth0。内部的这些容器可以通过localhost相互访问，但是它们不能在同一端口上同时开启服务，否则会有端口冲突，这就是共享网络栈的意思。Pod1中还有一个比较特殊的叫pause的容器，这个容器运行的唯一目的是为Pod建立共享的veth0网络接口。如果你SSH到K8s集群中一个有Pod运行的节点上去，然后运行docker ps，可以看到通过pause命令运行的容器。

Pod的IP是由docker0网桥分配的，例如上图docker0网桥的IP是172.17.0.1，它给第一个Pod1分配IP为172.17.0.2。如果该节点上再启一个Pod2，那么相应的分配IP为172.17.0.3，如果再启动Pod可依次类推。因为这些Pods都连在同一个网桥上，在同一个网段内，它们可以进行IP寻址和互通，如下图所示：

![](https://github.com/stevenhoukai/myblog/blob/main/images/net-pod-4.jpg)


从上图我们可以看到，节点内Pod网络在172.17.0.0/24这个地址空间内，而节点主机在10.100.0.0/24这个地址空间内，也就是说Pod网络和节点网络不在同一个网络内，那么不同节点间的Pod该如何IP寻址和互通呢？下一节我们来分析这个问题。

不同节点间的Pod网络
现在假设我们有两个节点主机，host1(10.100.0.2)和host2(10.100.0.3)，它们在10.100.0.0/24这个地址空间内。host1上有一个PodX(172.17.0.2)，host2上有一个PodY(172.17.1.3)，Pod网络在172.17.0.0/16这个地址空间内。注意，Pod网络的地址，是由K8s统一管理和分配的，保证集群内Pod的IP地址唯一。我们发现节点网络和Pod网络不在同一个网络地址空间内，那么host1上的PodX该如何与host2上的PodY进行互通？
![](https://github.com/stevenhoukai/myblog/blob/main/images/net-pod-5.jpg)



实际上不同节点间的Pod网络互通，有很多技术实现方案，底层的技术细节也很复杂。为了简化描述，我把这些方案大体分为两类，一类是路由方案，另外一类是覆盖(Overlay)网络方案。

如果底层的网络是你可以控制的，比如说企业内部自建的数据中心，并且你和运维团队的关系比较好，可以采用路由方案，如下图所示：

![](https://github.com/stevenhoukai/myblog/blob/main/images/net-pod-6.jpg)


这个方案简单理解，就是通过路由设备为K8s集群的Pod网络单独划分网段，并配置路由器支持Pod网络的转发。例如上图中，对于目标为172.17.1.0/24这个范围内的包，转发到10.100.0.3这个主机上，同样，对于目标为172.17.0.0/24这个范围内的包，转发到10.100.0.2这个主机上。当主机的eth0接口接收到来自Pod网络的包，就会向内部网桥转发，这样不同节点间的Pod就可以相互IP寻址和通信。这种方案依赖于底层的网络设备，但是不引入额外性能开销。

如果底层的网络是你无法控制的，比如说公有云网络，或者企业的运维团队不支持路由方案，可以采用覆盖(Overlay)网络方案，如下图所示：

![](https://github.com/stevenhoukai/myblog/blob/main/images/net-pod-7.jpg)


所谓覆盖网络，就是在现有网络之上再建立一个虚拟网络，实现技术有很多，例如flannel/weavenet等等，这些方案大都采用隧道封包技术。简单理解，Pod网络的数据包，在出节点之前，会先被封装成节点网络的数据包，当数据包到达目标节点，包内的Pod网络数据包会被解封出来，再转发给节点内部的Pod网络。这种方案对底层网络没有特别依赖，但是封包解包会引入额外性能开销。

## CNI简介
考虑到Pod网络实现技术众多，为了简化集成，K8s支持CNI(Container Network Interface)标准，不同的Pod网络技术可以通过CNI插件形式和K8s进行集成。节点上的Kubelet通过CNI标准接口操作Pod网路，例如添加或删除网络接口等，它不需要关心Pod网络的具体实现细节。


## 总结
K8s的网络可以抽象成四层网络，第0层节点网络，第1层Pod网络，第2层Service网络，第3层外部接入网络。除了第0层，每一层都构建于上一层之上。
一个节点内的Pod网络依赖于虚拟网桥和虚拟网卡等linux虚拟设备，保证同一节点上的Pod之间可以正常IP寻址和互通。一个Pod内容器共享该Pod的网络栈，这个网络栈由pause容器创建。
不同节点间的Pod网络，可以采用路由方案实现，也可以采用覆盖网络方案。路由方案依赖于底层网络设备，但没有额外性能开销，覆盖网络方案不依赖于底层网络，但有额外封包解包开销。
CNI是一个Pod网络集成标准，简化K8s和不同Pod网络实现技术的集成。
有了Pod网络，K8s集群内的所有Pods在逻辑上都可以看作在一个平面网络内，可以正常IP寻址和互通。但是Pod仅仅是K8s云平台中的虚拟机抽象，最终，我们需要在K8s集群中运行的是应用或者说服务(Service)，而一个Service背后一般由多个Pods组成集群，这时候就引入了服务发现(Service Discovery)和负载均衡(Load Balancing)等问题;