#                                               侯小猴的笔记本

腾讯内推邮箱: 348008515@qq.com  or  stephenkhou@tencent.com 简历请说明一下自己的求职意向哈

路漫漫其修远兮，吾将上下而求索～

![](https://github.com/stevenhoukai/myblog/blob/main/images/timg.jpg)


## **k8s**

- ### **k8s学习笔记**

  - #### **架构解析**

    - ######  *[k8s整体架构](https://github.com/stevenhoukai/myblog/blob/main/docs/k8s-jiagou.md)*

  - #### **网络三部曲**
    - ######  *[Pod网络](https://github.com/stevenhoukai/myblog/blob/main/docs/k8s-net-pod.md)*
    - ######  *[Service网络](https://github.com/stevenhoukai/myblog/blob/main/docs/k8s-net-service.md)*
    - ######  *[NodePort vs LoadBalancer vs Ingress](https://github.com/stevenhoukai/myblog/blob/main/docs/k8s-net-nodeport.md)*

## **MQ**

- ### **Pulsar**

  - #### **核心原理解析**

    - ######  *[一文看懂Apache-Pulsar（上）- 原理篇](https://stevenhoukai.github.io/2022/04/15/20220415-pulsarall/)*

    - ###### *[一文看懂Apache-Pulsar（下）- 实践篇 - 待更新](https://stevenhoukai.github.io/2022/04/15/20220415-pulsardown/)*

    - ###### *[Pulsar Topic 和分区](https://stevenhoukai.github.io/2022/02/14/20220214-pulsar-arch/)*

    - ###### *[消息存储原理与 ID 规则](https://stevenhoukai.github.io/2022/02/15/20220215-pulsar-storage/)*

    - ###### *[消息副本与存储机制](https://stevenhoukai.github.io/2022/02/16/20220216-pulsar-replicate/)*

  - #### **使用总结(golang版)**

    - ###### *[Pulsar的订阅模式](https://stevenhoukai.github.io/2022/02/19/20220219-pulsar-sub/)*

    - ###### *[如何使用定时和延迟消息-待更新]()*

    - ###### *[让消息高级一点--标签过滤-待更新]()*

    - ###### *[核心实践--RETRY&DLQ-待更新]()*
  
    - ###### *[Producer&Consumer最佳实践方式-待更新]()*
  
  - #### **Tips**


- ### **Kafka**

  - #### **硬核**

    - ###### *[kafka核心原理](https://stevenhoukai.github.io/2022/03/31/20220331-kafkaall/)*

  - #### **Tips**

## **Golang**

- ### **Golang核心**

  - ###### *[Golang的string](https://stevenhoukai.github.io/2022/03/24/20220324-gostring/)*

  - ###### *[slice](https://stevenhoukai.github.io/2022/03/26/20220326-goslice/)*


- ### **GoLang并发编程**

  - ###### *[初识goroutine](https://stevenhoukai.github.io/2020/12/04/20201204-goroutine/)*

  - ###### *[自定义实现简易goroutine池](https://stevenhoukai.github.io/2021/09/03/20210903-goroutinepool/)*

  - ###### *[引入简易goroutine池引发的一个bug](https://stevenhoukai.github.io/2021/09/14/20210914-goroutinepoolcpubug/)*

  - ###### *[golang的select使用总结](https://stevenhoukai.github.io/2021/09/14/20210914-goroutineselect/)*

- ### **Web开发与系统设计**

  - ###### *[如何设计一个高性能短链系统](https://stevenhoukai.github.io/2021/09/27/20210927-shorturl/)*

  - ###### *[如何设计一个cdk系统](https://stevenhoukai.github.io/2021/10/08/20211008-cdk/)*

  - ###### *[为什么我们要用protobuf](https://stevenhoukai.github.io/2022/03/17/20220313-protobuf/)*

  - ###### *[如何保护好自己的服务](https://stevenhoukai.github.io/2022/09/05/20220905-ratelimit/)*

- ### **工作踩坑记录**

  - ###### *[场景1:for循环遍历](https://stevenhoukai.github.io/2021/08/16/20210816-gapGolangFor/)*

  - ###### *[场景2:如何把Redis中同一前缀的所有key遍历出来](https://stevenhoukai.github.io/2021/08/19/20210819-redisScan/)*

  - ###### *[场景3:一个cpu被打满的一天](https://stevenhoukai.github.io/2021/09/14/20210914-goroutinepoolcpubug/)*

## **数据结构与算法**

- ###  **常用数据结构**

  - #### **二叉树**

    - ######  *[二叉树的遍历](https://stevenhoukai.github.io/2020/11/26/20201126-blbinarytree/)*
  
  - ####  **栈**
  
    - ######  *[用栈实现带括号的四则混合运算](https://stevenhoukai.github.io/2020/11/25/20201125-selfcomputer/)*

## **数据库**

- ### **Mysql**

  - #### **硬核**

    - ######  *暂无*

  - #### **Tips**


- ### **Redis**

  - #### **硬核**

    - ######  *[渐进式Rehash？](https://stevenhoukai.github.io/2021/08/24/20210824-redisRehash/)*

    - ###### *[Scan命令为啥会不漏值而且有可能会出现重复值](https://stevenhoukai.github.io/2021/08/25/20210825-redisScanExplain/)*

  - #### **Tips**


## **Linux与网络**

## **YonYou UAP**

- ###### *[UAP通讯模型解析](https://stevenhoukai.github.io/2019/07/19/20190719-2/)*

- ###### *[UAP事务如何处理？](https://stevenhoukai.github.io/2019/07/24/20190724/)*

- ###### *[简单模拟UAP客户端服务端通讯---RMI](https://stevenhoukai.github.io/2019/08/05/20190805/)*

## **JAVA**

- ### **Spring**

  - ###### *[手动简单实现基于xml的IOC](https://stevenhoukai.github.io/2020/01/28/20200127springioc/)*

  - ###### *[手动简单实现基于annotation的IOC](https://stevenhoukai.github.io/2020/01/29/springioc-anno/)*

## **番外篇**

- ###### *[hexo迁移（Windows -> Mac)](https://stevenhoukai.github.io/2020/12/18/20201218-HexoTransferWindowsToMac/)* 

## **爱好篇**

- ### 和弦分析****

  - ###### *[和弦的构成（分析）](https://stevenhoukai.github.io/2021/10/10/20211010-guitar/)*

