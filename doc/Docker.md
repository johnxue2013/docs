# 第一本docker书 笔记
- 可以使用以下命令创建长期运行的容器。
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

- 使用Dockerfile构建新镜像
不推荐使用docker commit的方法构建镜像
首先
创建一个文件夹
```Bash
$ mkdir static_web
$ cd static_web
$ touch Dockerfile
$ docker build -t="jameur01/static_web" .
```
static_web这个目录就是我们构建的环境(build environment)，Docker则称此环境为上线文(context)或者构建上下文(build context)。Docker
会在构建镜像时将构建上下文和改上下文中的文件和目录上传到Docker守护进程。这样Docker守护进程就能直接访问你想在镜像中存储的任何代码、文件
或者其他数据。

也可以为镜像设置一个标签(如果没有指定任何标签，Docker将会自动为镜像设置一个latest标签)
```Bash
$ docker build -t="jamru01/static_web:v1" .
Sending build context to Docker daemon  2.048kB
Step 1/8 : FROM ubuntu:12.04
 ---> 5b117edd0b76
Step 2/8 : MAINTAINER James Turnbull "james@example.com"
 ---> Using cache
 ---> 8b292e7eb386
Step 3/8 : RUN apt-get update
 ---> Using cache
 ---> 0f18b40d86b0
Step 4/8 : RUN apt-get install -y nginx
 ---> Using cache
 ---> 6ae649594ecc
Step 5/8 : RUN mkdir -p /usr/share/nginx/html
 ---> Running in b5e399f5e355
Removing intermediate container b5e399f5e355
 ---> ffd6f5171079
Step 6/8 : RUN touch /usr/share/nginx/html/index.html
 ---> Running in cb428b373f68
Removing intermediate container cb428b373f68
 ---> 7262dc5da97b
Step 7/8 : RUN echo 'Hi, I am in your container' > /usr/share/nginx/html/index.html
 ---> Running in edd9ea582b2f
Removing intermediate container edd9ea582b2f
 ---> e2fcb13cec8f
Step 8/8 : EXPOSE 80
 ---> Running in 244f718cd1d3
Removing intermediate container 244f718cd1d3
 ---> 4f19758f46d9
Successfully built 4f19758f46d9
Successfully tagged johnxue2013/static_web:v1
```

上面命令中最后的.告诉Docker到本地目录中去找Dockerfile文件。也可以指定一个Git仓库的原地址来指定Dockerfile的位置
(Git仓库根目录必须存在Dockerfile文件)
> 如果在构建上下文的根目录存在以 .dockerignore命名的文件的话，那么该文件内容会被按行进行分割，类似.gitignore文件
Dockerfile中的每一条指令(FROM RUN等)都应该大写

- 构建不使用缓存
正常情况下，每次build的时候会将之前构建时创建的镜像当做缓存并作为新的开始点。这样会节省大量时间。然而有时候，需要确保构建过程不会使用缓存
。必须如果已经缓存了前面第三部即apt-get update，那么Dcoker不会再次刷新APT包的缓存。此时可以使用--no-cache标志忽略缓存功能，如
```Bash
docker build --no-cache -t "johnxue2013/static_web:v1" .
```

- 启动构建的容器
```Bash
docker run -d -p 80 --name static_web jamtur01/static_web nginx -g "daemon off;"
```

这将在Docker宿主机上随机打开一个端口，这个端口会链接到容器中的80端口上。使用`docker ps -l`，命令来看下容器的端口分配情况
```bin/Bash
$ static_web docker ps -l
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                   NAMES
ea5797ee864b        4f19758f46d9        "nginx -g 'daemon of…"   18 hours ago        Up 18 hours         0.0.0.0:32768->80/tcp   static_web
```

上面显示宿主机的32768端口映射到了容器的80端口

也可以使用docker port命令查看端口映射情况
```bin/Bash
$ static_web docker port ea5797ee864b
80/tcp -> 0.0.0.0:32768
$ static_web docker port ea5797ee864b 80
0.0.0.0:32768
```


这里使用了-p标志，该标志用来控制Docker在运行时应该公开哪些网络端口给外部(宿主机)。运行一个容器可以通过两种方法在宿主机上分配端口

1. Docker可以在宿主机上随机选择一个位于49153~65535的一个
2. 可以在Docker宿主机中指定一个具体的端口号来映射到容器中的80端口上

-p选项还可以灵活的指定宿主机和容器之间的端口映射关系。比如
```bin/Bash
$ docker run -d -p 8080:80 --name static_web jamtur01/static_web nginx -g "daemon off;"
```
这条命令将容器中的80端口绑定到宿主机的8080端口上

Docker还提供了一个-P(大写)参数，该参数可以用来对外公开在Dockerfile中的EXPOSE指定中设置的所有端口
```bin/Bash
$ docker run -d -P --name static_web jamtur01/static_web nginx -g "daemon_off;"
```
该命令会将容器内的80端口对本地宿主机公开，并且绑定到宿主机的一个随机端口上。

- Dockerfile指令

### CMD
 用于指定一个容器启动时要运行的命令。类似于RUN，只是RUN指令是指定镜像被构建时要运行的命令，而CMD是指定容器被启动时要运行的命令。
 这和使用docker run命令刚启动容器时指定要运行的命令非常相似
 比如
 ```bin/bash
   $ docker run -it jamtur01/static_web /bin/true
 ```
 和
 CMD ["/bin/true"]
 是等价的。
 当然也可以为要运行的命令指定参数，如
 ```bin/bash
  CMD["/bin/bash", "-l"]
 ```

**docker run命令可以覆盖CMD命令**如果在Dockerfile指定了CMD命令，而同时在docker run命令中也指定了要运行的命令，命令中指定的命令会覆盖Dockfile中的CMD指令。

**一个Dockerfile中只能指定一条CMD指令。**如果指定了多条CMD指令，也只有最后一条CMD指令会被使用。

### ENTRYPOINT

ENTRYPOINT指令与CMD类似，只是ENTRYPOINT不易被docker run覆盖。实际上，docker run命令行中指定的任何参数都会被当做参数再次传递给ENTRYPOINT指令中
指定的命令。
如假设Dockerfile中存在ENTRYPOINT ["/usr/sbin/nginx"]
```bin/Bash
docker run -it jamtur01/static_web -g "daemon off;"
```
上面的-g "daemon off;"会被再次当做参数传递给`/usr/sbin/nginx`命令。

可以通过组合使用ENTRYPOINT和CMD指令来完成一些巧妙的工作如Dockerfile中存在如下指令
```
ENTRYPOINT ["/usr/sbin/nginx"]
CMD ["-h"]
```
此时当我们启动一个容器时，任何命令中指定的参数都会被传递给Nginx守护进程。如果不指定任何参数，则CMD指令中的-h参数会被传递给Nginx守护进程，即
Nginx服务器会以`/usr/sbin/nginx -h`的方式启动，该命令用来显示nginx的帮助信息

> 如果确实需要覆盖ENTRYPOINT指令，可以使用docker run --entrypoint标志覆盖ENTRYPOINT指令

### WORKDIR
用来在从镜像创建一个新容器时，在容器内部设置一个工作目录，ENTRYPOINT和/或CMD指定的程序会在这个目录下执行

可以使用该命令为Dockerfile中后续的一些列指令设置工作目录。也可以为最终的容器设置工作目录。如
```Bash
WORKDIR /opt/webapp/db
RUN bundle install
WORKDIR /opt/webapp
ENTRYPOINT ["rackup"]
```
上述命令将工作目录切换为/opt/webapp/db后运行了bundle install命令，之后又将工作目录设置为/opt/webapp，最后设置了ENTRYPOINT指令启动rackup命令。

可以通过-w 标志在运行时覆盖工作目录
```Bash
docker run -ti -w /var/log ubuntu pwd
```

### ENV
用来在镜像构建过程中设置环境变量
```Bash
ENV RVM_PATH /home/rvm/
```
这个新的环境变量可以在后续的任何RUN指令中使用，这就如同在命令前面指定了环境变量前缀一样

```Bash
RUN gem install unicorn
```

也可以在其他指令中直接引用
```Bash
ENV TARGET_DIR /opt/app
WORKDIR $TARGET_DIR
```

> 进入容器或可以使用env命令查看环境变量

### USER
USER指令用来指定改镜像会以什么样的用户去运行，比如
```Bash
USER nginx
```
基于该镜像启动的容器会以nginx用户的身份来运行。

也可以在docker run -u标识来覆盖该指令的值
 > 如果不通过USER指令指定用户，默认用户为root

 ### VOLUME
用来想基于镜像创建的容器添加卷。一个卷可以是存在于一个或者多个容器内特定的目录。
```Bash
VOLUME ["/opt/project", "/data"]
```
> 使用VOLUME只是在容器中设置了挂载点/opt/project,并不能指定宿主机中的挂在路径。此时Docker会自动绑定宿主机上的一个目录。通过
docker inspect命令可以查看到

> 可以通过docker run -v {本地路径}:{容器挂载点}，直接指定挂在的路径

```Bash
xqh@ubuntu:~/myimage$ docker inspect test1
[
{
    "Id": "1fd6c2c4bc545163d8c5c5b02d60052ea41900a781a82c20a8f02059cb82c30c",
.............................
    "Mounts": [
        {
            "Name": "0ab0aaf0d6ef391cb68b72bd8c43216a8f8ae9205f0ae941ef16ebe32dc9fc01",
            "Source": "/var/lib/docker/volumes/0ab0aaf0d6ef391cb68b72bd8c43216a8f8ae9205f0ae941ef16ebe32dc9fc01/_data",
            "Destination": "/data",
            "Driver": "local",
            "Mode": "",
            "RW": true
        }
    ],
...........................

```

卷是在一个或多个容器中特殊指定的目录，卷会绕过联合文件系统，为持久化数据和共享数据提供几个有用的特性：
* 卷可以在容器间共享和重用
* 共享卷时不一定要运行响应的容器
* 对卷的修改会直接在卷上反应出来
* 更新镜像时不会包含对卷的修改
* 卷会一直存在，知道没有容器使用它们(容器可以没在运行，但容器要存在，如果容器也不存在，卷就会被删除)

> 如果删除了最后一个使用卷的容器，卷就不见了。所有删除可能有数据的卷的容器时要小心。
上面 Mounts下的每条信息记录了容器上一个挂载点的信息，"Destination" 值是容器的挂载点，"Source"值是对应的主机目录。

可以看出这种方式对应的主机目录是自动创建的，其目的不是让在主机上修改，而是让多个容器共享。


### ADD

### COPY
与ADD类似，但COPY值关心在构建上下文中复制本地文件，而不会做文件提取和解压的工作

### ONBUILD
为镜像添加触发器(trigger)。当一个镜像被用作其他镜像的基础镜像时，该镜像中的触发器将被执行。

触发器会在构建过程中插入新指令，可以认为这些指令是紧跟在FROM之后的指令。触发器可以使任何构建指令。
```Bash
ONBUILD ADD . /app/src
ONBUILD RUN cd /app/src && make
```
> ONBUILD指令可以在镜像上运行docker inspect {container ID | container name}命令来查看

### 进入容器的方法
有多种方法，此处列出两种，使用`docker attach`和`docker exec`。
docker attach的方式进入容器后，当多窗口同时使用该命令进入该容器时，所有窗口都会同步显示，如果有一个窗口阻塞，那么其他窗口也无法再进行操作。

在Docker1.3.x版本之后，提供了exec命令用于进入容器，用法如：
```Bash
docker exec -it <cotainer ID> /bin/bash
```

