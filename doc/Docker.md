# 第一本docker书 笔记
- 可以使用一下命令创建长期运行的容器。
```Bash
docker run --name daemon_dave -d ubuntu /bin/sh -c "while true; do echo hello world; sleep 1; done"
```

- 使用docker logs 查看容器日志
```bin/Bash
docker logs {<container name> | <container ID>}
```
这条命令会输出最后几条日志并返回。可以使用-f参数来监控Docker日志，这与tail -f命令相似
或者加上-t参数 显示打印日志的时间
```bin/Bash
docker logs  -ft {<container name> | <container ID>}
```

- 查看容器内的进程
```bin/Bash
docker top {<container name> | <container ID>}
```


- 在容器内部运行进程
可以在容器内启动的进程有两种类型：后台任务和交互式任务

```bin/Bash
docker exec [OPTIONS] CONTAINER COMMAND [ARG...]
```

如在daemon_dave容器中新建一个new_config_file文件
```bin/Bash
docker exec -d daemon_dave touch /etc/new_config_file
```

也可以在daemon_dave容器中启动搞一个诸如打开shell的交互式任务
```bin/Bash
docker exec -it daemon_dave /bin/bash
```

- 停止守护式容器
```bin/Bash
doker stop {<container name> | <container ID>}
```
> stop命令会向Docker容器进程发送SIGTERM信号。如果想要快速终止容器，可以使用docker kill命令来向容器进程发送SIGKILL信号

- 自动重启容器
如果由于某种错误导致容器停止运行，可以通过--restart标志让Docker自动重新启动该容器。--restart会检查容器退出代码，并据此来决定是否要重启容器
```bin/Bash
docker run --restart=always --name daemon_dave -d ubuntu /bin/bash -c "while true;do echo hello world;sleep 1;done"
```
本例中，--restart标志被设置为always。无论容器退出代码是什么，Docker都会自动重启该容器。出了always，还可以设置为on-failure,这样
只有当容器的退出代码为非0的时候才会自动重启。此外on-failure还可以接受一个可选的重启次数参数，如
```bin/Bash
--restart=on-failure:5
```


- 深入容器
除了使用docker ps命令获取容器的信息，还可以使用docker inspect 来回去更多的容器信息

- 删除容器
可以使用docker rm命令来删除他们

> 可以使用 docker rm `docker ps -a -q`删除所有容器。

- TAG
为了区分同一个镜像的不同版本，docker提供了tag。如可以通过指定tag运行指定的镜像
```bin/Bash
docker run -it --name new_container ubuntu:12.04 /bin/bash
```

- docker 仓库
Docker Hub中有两种类型的仓库：用户仓库(user repository)和顶层仓库(top-level repository)。

用户仓库的命名由用户名和仓库名两部分组成如 jamtur01/pupet
用户名:jamtur01
仓库名: pupet

而顶层仓库只包含仓库名称如ubuntu

- 拉取镜像
用docker run命令从镜像启动一个容器时，如果该镜像在本地不存在，Docker会先从Docker Hub下载该镜像。如果没有指定具体的镜像标签，
那么Docker会下载latest标签的镜像。
也可以通过docker pull命令预先拉取镜像

> docker images查看所有镜像，或者docker images <image name> 查看特定的镜像

- 构建镜像
有两种方法

1. 使用`docker commit`命令
2. 使用`docker build`命名和Dockerfile文件

使用docker commit命令
```bin/Bash
$ docker run -it ubuntu /bin/bash
root@4aab3ce3cb76
$ apt-get -yqq update
$ apt -get -y install apache2
```
我们启动了一个容器，并在里面安装了Apache。我们会将整个容器作为web服务器运行起来，所以需要保存当前安装的东西。为了完成此项工作，需要
先使用exit命令从容器中退出，之后再使用docker commit命名

```bin/Bash
$ docker commit 4aab3ce3cb76 jamtur01/apache2
8ce0ea7a1528
```
该命名中指定了要提交的修改过的容器ID，以及一个目标仓库和镜像名，这里是jamture01/apache2。这里需要注意的是。docker commit提交的只是
创建容器的镜像与当前诸状态之间有差异的部分，这使得该更新非常轻量。

> 可以通过docker ps -l -q命令得到刚创建的容器ID

也可以在提交镜像时指定更多的数据（包括标签）来详细描述所做修改。
```bin/Bash
$ docker commit -m="A new custom image" --author="James Turnbull" 4aab3ce3cb76 jamtur01/apache2:webserver
```

-m 指定提交信息

commit后可以使用`docker push`推送到仓库



