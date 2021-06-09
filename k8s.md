# kubernetes
## 组件：

![](https://data-repository-01.oss-cn-shanghai.aliyuncs.com/img/20210605140917.png)

- Api Server：所有服务访问统一入口
- Controller Manager： 维持副本期望数、有状态无状态应用部署、确保所有的node运行同一个pod，一次性任务和定时任务
- Scheduler： 负责接收任务，选择合适的节点进行分配任务
- ETCD： 键值对数据库，存储K8S集群所有重要信息（持久化）
- Kubelet： 直接跟容器引擎交互，实现容器生命周期管理
- Kube-proxy: 负责写入规则至Ip tables、IPVS实现服务(端口)映射访问
- Core DNS: 可以为集群中的SVC创建一个域名-IP的对应关系解析
- Dash Board: 给K8S提供一个B/S的结构访问体系
- Ingress Controller： 官方只能实现四层代理，Ingress可以实现七层代理
- Federation: 提供一个可以扩集群中心多K8S统一管理功能
- Prometheus： 提供K8S集群的监控能力
- ELK：提供K8S集群日志统一分析接入平台

## Pod
一个Pod是可以启动(包含)多个容器的，多个容器之间共享这个Pod的端口，都使用Pod的IP地址和存储卷

### Pod类型
#### ReplicationController
简称RC，用来确保容器应用的副本数始终保持在用户定义的副本数，即如果有容器异常退出，会自动创建新的Pod代理；而如果异常多出来的容器也会自动回收。在新版的k8s中建议使用ReplicaSet来取代ReplicationController。
#### ReplicaSet
简称RS，和RC没有本质不同，只是名字不一样，并且RS支持集群式的selector
#### Deployment
虽然RS可以独立使用，但一般还是建议使用Deployment来自动管理RS，这样就无需担心和其他机制不兼容的问题（比如RS不支持rolling-update，但Deployment支持）

> 也就是先创建Deploment，Deployment再创建RS，RS再创建Pod

#### HPA(Horizontal Pod AutoScale)
仅适用于Deployment和RS，在V1版本中仅支持根据Pod的CPU利用率扩容，在V1alpha版本中支持根据内存和用户自定义的metric扩容
#### StatefulSet
为了解决有状态服务的问题（对应的Deployments和RelicaSets是为了无状态服务而设计），其应用场景包括：
- 稳定的持久化存储，即Pod重新调度后还是能访问到相同的持久化数据(基于PVC实现)
- 稳定的网络标识,即Pod重新调度后其PodName和HostName不变，基于Headless Service(即没有Cluster IP的Service)来实现
- 有序部署，有序扩展。（即从0到N-1，在下一个Pod运行之前所有的Pod必须都是Running和Ready状态），基于init containers来实现
- 有序收缩，有序删除，即从N-1到0
#### DaemonSet
确保全部(或者一些)Node上运行一个Pod副本。当有Node加入集群时，也会为他们新增一个Pod
#### Job, ConJob

## k8s集群的命令行工具
### kubectl
kubectl(kubecontroller的缩写)是k8s集群的命令行工具，通过kubectl能够对集群本身进行管理，并能够在集群上进行容器化应用的安装

#### 语法
```shell
$ kubectl [commond] [TYPE] [NAME] [flags]
```
commond: 指定要对资源执行的操作，例如 create、get、describe和delete
TYPE: 指定资源类型，资源类型是大小写敏感的，开发者能够以单数、复数和缩略的形式。 
NAME: 指定资源名称，名称也是大小写敏感的。如果省略名称，则查询所有的资源

flags: 指定可选的参数。例如，可用-s或者-server参数指定k8s api server的地址和端口

例如
```shell
$ kubectl get pod pod1
$ kubectl get pods pod1
```


