# 1.1 Kubernetes是什么
- 一个全新的基于容器技术的分布式架构领先方案
- 一个开放的开发平台
  - 不局限于任何一种语言
  - 没有限定任何编程接口
  - 对现有编程语言、框架、中间件没有任何侵入性
- 一个完备的分布式系统支撑平台
  - 多层次的安全防护和准入机制
  - 多租户应用支撑能力
  - 透明的服务注册和服务发现机制
  - 内建智能负载均衡器
  - 强大的故障发现和自我修复能力
  - 服务滚动升级和在线扩容能力
  - 可扩展的资源自动调度机制
  - 多粒度的资源配额管理能力
  - 完善的管理工具
## 集群管理
集群中的机器称为节点，k8s将这些节点分为Master节点和工作节点（Node）。

Master节点运行kube-apiserver,kube-controller-manager,kube-scheduler进程，负责整个集群的资源管理、Pod调度、弹性伸缩、安全控制、系统监控和纠错等功能

Node上运行kubelet，kube-proxy服务进程，负责Pod的创建、启动、监控、重启、销毁以及实现软件模式的负载均衡器
## Pod
Pod是k8s管理的最小运行单元。
一个Pod具有一下特征
- 运行在节点（Node，一个物理机或者虚拟机）环境中
- 运行着一个被称为Pause的容器，其他容器都是业务容器。业务容器共享Pause容器的网络栈和Volume挂载卷
## Service
Service（服务）是分布式集群架构的核心，一个Service对象拥有如下关键特征。
- 拥有一个唯一指定的名字（比如 mysql-server）
- 拥有一个虚拟IP（Cluster IP, Service IP或VIP）和端口号
- 能够提供某种远程服务能力
- 被映射到了提供这种服务能力的一组容器应用上

一个Service通常由多个相关的服务进程提供服务，每个服务进程都有一个独立的Endpoint（IP+Port）访问点。k8s能够让我们通过Service对应的虚拟Cluster IP+Service Port链接到指定的Service上。而k8s内建的透明负载均衡和故障恢复机制，能够保证不管Service对应多少服务进程，或者某个服务进程是否会有与发生故障而重新部署到其他机器，亦或者服务进程的IP地址或者端口号如何变化，都不会影响我们对Service的调用。
## Service与Pod映射
只有提供服务的一组Pod才映射成一个Service。

k8s为每个Pod设置一个标签（lable），例如name=mysql或者name=php,为相应的Service定义标签选择器（Lable Selector）,如MySQL Service的标签选择器的条件为name=mysql，意为该Service要作用于所有包含name=mysql标签的Pod上。
## Service扩容
为Service关联的Pod创建一个具有一下3个关键信息的RC（Replication Controller）
- 目标Pod的定义
- 目标Pod需要运行的副数量（Replicas）
- 要监控的目标Pod的标签（Label）

k8s通过RC中定义的Label筛选对应的Pod实例并实时监控状态和数量，数量少于定义的副本数量（Replicas）时，根据RC中定义的Pod模板创建新的Pod，并将此Pod调度到合适的Node上启动运行，直到Pod实例的数量达到预定目标。此过程完全自动化。

# 1.2 为什么要用Kubernetes 21页
