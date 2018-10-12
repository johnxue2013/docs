# Kubernetes
## Kubernetes 集群
一个Kubernetes集群包含两中资源类型：
- Master 协调集群
- Nodes 工作节点，运行应用  

![image](https://github.com/johnxue2013/tools/blob/master/images/k8s-cluster.png)

Master 负责管理集群，master协调所有集群中的活动，比如(scheduling applications)调度应用程序，维护应用程序的所需状态，扩展应用程序并推出新的更新

Node是一个虚拟机或者一个物理机，用作Kubernetes集群中的工作计算机。每个node都有一个Kubelet，它是一个管理节点并与Kubernetes Master通信的代理。该节点还应具有用于处理容器操作的工具，例如Docker或rkt。处理生产流量的Kubernetes集群应至少具有三个节点。  

在Kubernetes上部署应用程序时，您可以告诉master 启动应用程序容器。master 调度容器以在集群的节点上运行。node使用master公开的Kubernetes API与master进行通信。终端户还可以直接使用Kubernetes API与群集进行交互。  

Kubernetes集群可以部署在物理机或虚拟机上。要开始使用Kubernetes开发，您可以使用Minikube。Minikube是一种轻量级Kubernetes实现，可在本地计算机上创建VM并部署仅包含一个节点的简单集群。Minikube适用于Linux，macOS和Windows系统。 Minikube CLI提供了与群集配合使用的基本引导操作，包括启动，停止，状态和删除。

## Kubernetes Pods
当我们创建一个Deployment，Kubernetes创建一个**Pod**来托管你的应用程序实例。Pod是一个Kubernetes抽象，表示一组一个或多个应用程序容器（如Docker或rkt），以及这些容器的一些共享资源。这些资源包括：
- Shared storage, as Volumes
- Networking, as a unique cluster IP address
- Information about how to run each container, such as the container image version or specific ports to use

Pod模拟特定于应用程序的“逻辑主机”，并且可以包含相对紧密耦合的不同应用程序容器。 例如，Pod可能既包含带有Node.js应用程序的容器，也包含一个不同的容器，用于提供Node.js网络服务器要发布的数据。 Pod中的容器共享IP地址和端口空间，始终位于同一位置并共同调度，并在同一节点上的共享上下文中运行。

Pod是Kubernetes平台上的原子单元。 当我们在Kubernetes上创建Deployment时，该Deployment会在其中创建包含容器的Pod（而不是直接创建容器）。 每个Pod都与调度它的节点绑定，并保持在那里直到终止（根据重启策略）或删除。 如果节点发生故障，则会在群集中的其他可用节点上安排相同的Pod。

> 总结： A Pod is a group of one or more application containers (such as Docker or rkt) and includes shared storage (volumes), IP address and information about how to run them.

### Pods 概览

![image](https://github.com/johnxue2013/tools/blob/master/images/pods-overview.png)

### Nodes
Pod使用运行在一个Node中，Node是Kubernetes中的工作机器，可以是虚拟机器，也可以是物理机器，具体取决于集群。每个Node由Master管理，一个Node可以拥有多个Pod，Kubernetes master会在集群中的多个Node上自动处理，调度Pod。Master的自动调度考虑了每个节点上的可用资源。

每一个Kubernetes节点至少运行:
- Kubelet, 负责Kubernetes Master和Node之间通信的过程;它管理Pod和在机器上运行的容器。
- 容器运行时（如Docker，rkt）负责从注册表中提取容器映像，解压缩容器以及运行应用程序。

> 总结: 容器只应在一个Pod中一起安排，如果它们紧密耦合并且需要共享磁盘等资源

### Node overview
![image](https://github.com/johnxue2013/tools/blob/master/images/node-overview.png)