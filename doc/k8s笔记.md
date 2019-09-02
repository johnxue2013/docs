# Kubernetes的几个重要概念
## Cluster
Cluster是计算、存储和网络资源的集合，Kubernetes利用这些资源运行各种基于容器的应用。

## Master Master是Cluster的大脑，它主要职责是调度，即决定将应用放在哪里运行。Master运行Linux操作系统，可以是物理机或者虚拟机。为了实现高可用，可以运行多个Master

## Node
Node的职责是运行容器应用。Node由Master管理，Node负责监听并汇报容器的状态，同时根据Master的要求管理容器的生命周期。Node运行在Linux操作系统上，可以是物理机或者是虚拟机。

## Pod
Pod是容器的集合，通常会将紧密相关的一组容器放到一个Pod中（也就是说每个Pod可以包含一个或多个容器），同一个Pod中的所有容器共享IP地址和Port空间，也就是说他们在同一个network namespace中。

Kubernetes以Pod为最小单位进行调度、扩展、共享资源、管理生命周期。

Pod中所有容器使用同一个网络namespace，即相同的IP地址和Port空间，他们可以直接用localhost通信。同样的，这些容器可以共享存储，当K8s挂在volume到Pod，本质上是将volume挂在到Pod中的每一个容器。

Pods有两种使用方式：
1. 运行单一容器。
one-contaniner-per-Pod是K8s最常见的模型，这种情况下，只将单个容器简单封装成Pod。即便只有一个容器，K8s管理的也是Pod而不是直接管理容器。
2. 运行多个容器

## Controller
创建Pod

## Service
每个Pod都有自己的IP，外接如何访问这些副本呢？要知道Pod很可能被频繁的销毁和重启，他们的IP会发生变化，用IP来访问不太现实。答案是Service。

Kubernetes Service定义了外界访问一组特定Pod的方式。Service有自己的IP和端口，Service为Pod提供了负载均衡。

K8s运行容器（Pod）与访问容器（Pod）这两项任务分别由Controller和Service执行。

## Namespace
如果有多个用户或项目组使用同一个KubernetesCluster，如何将他们创建的Controller、Pod等资源分开呢？

答案就是Namespace。

Namespace可以将一个物理的Cluster逻辑上划分多个虚拟Cluster，每个Cluster就是一个Namespace。不同Namespace里资源是完全隔离的。

Kubernetes默认创建了两个Namespace，
1. default: 创建资源时如果不指定，将被放到这个Namespace中。
2. kube-system: Kubernetes自己创建的系统资源将放到这些Namespace中


### 第五章 运行应用
运行一个Deployment
```
kubectl run nginx-deployment --image=nginx:1.7.9 --replicas=2
```
上面的命令将部署包含两个副本的Deployment nginx-deployment,容器的image为nginx:1.7.9

运行kebuctl get deployment <deployment name>可以查看应用的状态

```
kebuctl get deployment nginx-deployment
```

还可以使用kubectl describe deployment <deployment name>了解更多的详细信息

kubernetes支持两种创建资源的方式：
（1）用kubectl命令直接创建，比如"kubectl run nginx-deployment --image=nginx:1.7.9 --relicas=2"，在命令中通过参数指定

（2）通过配置文件和kubectl apply <file name> 命令创建。要完成前面相同的工作，可执行命令 `kubectl apply -f ngnix.yml`, nginx.yml的内容如下:
```xml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
 name: nginx-deployment
spec:
 relicas: 2
 template:
  metadata:
   labels:
    app: web_server
   spec:
    containers:
    - name: nginx
      image:nginx:1.7.9
```

两种方式比较：
(1) 基于命令的方式：
- 简单、直观、快捷、上手快。
- 适合临时测试或实验

（2）基于配置文件的方式：
- 配置文件描述了What，即应用最终要达到的状态
- 配置文件提供了创建资源的模板，能够重复部署。
- 可以像代码管理一样管理部署
- 适合正式的、跨环境的、规模化部署
- 这种方式要求熟悉配置文件语法，有一定难度

### 用label控制Pod的位置
默认配置下Scheduler会将Pod调度到所有可用的Node。不过有些情况我们希望将Pod部署到指定的Node，比如将有大量磁盘I/O的Pod部署到配置了SSD的Node；或者Pod需要GPU，需要运行在配置了GPU的节点上。

K8s是通过label来实现这个功能的。

label是key-value对，各种资源都可以设置label，灵活添加各种自定义属性。比如执行如下命令标注k8s-node1是配置了SSD的节点。
```
kubectl label node K8s-node1 disktype=ssd
```

通过
```
kubectl get node --show-labels
```
查看界节点的label

有了disktype这个自定义label，接下来就可以指定将Pod部署到k8s-node1。配置文件如下

```xml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
 name: nginx-deployment
spec:
 relicas: 2
 template:
  metadata:
   labels:
    app: web_server
   spec:
    containers:
    - name: nginx
      image:nginx:1.7.9
    nodeSelector:
     disktype: ssd
```
在Pod模板的spec里通过nodeSelector指定将此Pod部署到具有label distype=ssd的Node上。

要删除Node上的label的话，可以使用
```
kubectl lablel node k8s-node1 distype-
```
不过此时已经部署在k8s-node1上的Pod并不会重启，除非在nginx.yml中删除nodeSelector配置，然后通过kubectl apply 重新部署

### 通过service访问Pod
Kubernetes Service从逻辑上代表了一组Pod，具体是哪些Pod则是由label来挑选的。Service有自己的IP，而且这个IP是不变的（创建好的Pod如果重新部署或者重启IP将会改变）。客户端只要访问Service的IP，Kubernetes负责建立和维护Service与Pod的映射关系。无论后端Pod如何变化，对客户端不会有任何影响，因为Service没有变。

举个例子，创建下面的这个Deployment
```xml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: httpd
spec:
  replicas: 3
  template:
    metadata:
      labels:
        run: httpd
    spec:
      containers:
        - name: httpd
          image: httpd
          ports:
            - containerPort: 80
```

Service利用cloud provider特有的load balanceer对外提供服务，cloud provider负责将load balancer的流量导向Service。目前支持的cloud provider有GCP、AWS、Azur等。
