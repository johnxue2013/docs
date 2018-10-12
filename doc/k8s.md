# Kubernetes
## 核心概念
* Pod
若干相关容器的组合，Pod包含的容器运行在同一台宿主机上，这些容器使用相同的网络命名空间、IP地址和端口，相互之间能通过localhost来发现和通信。在k8s中创建、调度和管理最小单位是Pod,而不是容器。

* Replication Controller
Replication Controller用来控制管理Pod副本(Replica，或者称之为实例),Replication Controller确保任何时候K8s集群中有指定数量的Pod副本在运行。如果少于指定数量的Pod副本，Replication Controller会启动新的Pod副本，反之会杀死多余的副本以保证数量不变。另外Replication Controller 是弹性伸缩、滚动升级的实现核心。

* Service
Service是真实应用服务的抽象，定义了Pod的逻辑集合和访问这个Pod集合的策略。Service将代理Pod对外表现为一个单一访问接口，外部不需要了解后端Pod如何运行，这给扩展和维护带来了很多的好处，提供了一套简化的服务代理和发现机制。

* Node
K8s属于主从分布式集群架构，K8s Node(简称为Node，早起版本叫做Minion)运行并管理容器。Node作为K8s的操作单元，用来分配给Pod(或者说容器)进行绑定，Pod最终运行在Node上，Node可以认为是Pod的宿主机。


## Kubernetes 集群
一个Kubernetes集群包含两中资源类型：
- Master 协调集群
- Nodes 工作节点，运行应用

![image](https://github.com/johnxue2013/tools/blob/master/images/k8s-cluster.png)
