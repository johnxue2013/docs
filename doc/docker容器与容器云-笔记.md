# Docker容器与容器云

![image](https://github.com/johnxue2013/tools/blob/master/images/docker-2-1.png)


## 第一部分
### 第二章

### Docker参数解读
容器的生命周期涉及容器启动、停止等功能。
- `docker run 命令`

  命令格式
  ````Bash
  docker run [OPTIONS] IMAGE [COMMAND] [ARG...]
  ```

  如
  ```Bash
  docker run -it --name mytest ubuntu /bin/bash
  ```

  docker -d 参数可以让容器在后台运行，并在控制台打印出容器的ID 如：
  ```Bash
  ➜  ~ docker run -d --name my-redis redis
  823f91c2c4d0a35d40d2e76c1e9d05eabd124f2c2e8b213836727cf63ac36d28
  ```

- `docker start/stop/restart命令`

  docker run命令可以新建一个容器(container)来运行，而对于已经存在的容器，可以通过docker start/stop/restart命令来启动、停止和重启。使用docker run命令时，Docker将自动为每个新容器分配唯一ID作为标识。docker start等命令一般利用容器ID标识确定具体容器。

- docker pull命令

  语法
  ```Bash
  docker pull [OPTIONS] NAME[:TAG|@DIGEST]
  ```

  ```Bash
  #从官方hub拉取ubuntu:lastest镜像
  docker pull ubuntu

  #从官方拉取指明"ubuntu 12.04"tag的镜像
  docker pull ubuntu:ubuntu12.04

  # 从特定的仓库拉取ubuntu镜像
  docker pull SEL/ubuntu

  #从其他服务器拉取镜像
  # docker pull 10.10.103.215:5000/sshd
  ```

- docker images命令

  docker images 列出主机上所有镜像

- docker rmi和docker rm

  一个用于删除镜像，一个用于删除容器，可以同事删除多个镜像或者容器，也可按条件来删除

- docker attach命令

  可以连接到正在运行的容器，观察该容器的运行情况，或与容器的主进程进行交互

