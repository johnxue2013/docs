# IDEA Http Client使用
## 什么是IDEA Http Client？
是开发工具IDEA中集成的一个工具，允许开发者在IDEA中测试web服务的接口。

![](https://raw.githubusercontent.com/johnxue2013/docs/master/images/Screen%20Shot%202021-12-29%20at%2016.49.00.png)

http请求文件扩展名可以为 `.http`或者`.rest`。

## 创建http请求文件
可以直接手动创建上述指定的扩展名文件，IDEA自动识别出这个文件，也可以使用 

![](https://raw.githubusercontent.com/johnxue2013/docs/master/images/Screen%20Shot%202021-12-29%20at%2017.12.47.png)
进行创建

> 使用第二种创建方式生成的文件会存放在`Scratches and Consoles`文件夹下，这个文件夹和项目无关，是所有项目共享的，一般不建议使用这个文件，各个项目的`.http`文件应该属于各自的项目

## 编写测试接口
打开上述创建的文件，在文件的右上角，IDEA为我们内置了快速创建请求接口的方法
![](https://raw.githubusercontent.com/johnxue2013/docs/master/images/Screen%20Shot%202021-12-29%20at%2017.19.32.png)

编写完测试的接口后，直接像运行java的main方法一样去运行就可以了，运行结果如下
![](https://raw.githubusercontent.com/johnxue2013/docs/master/images/Screen%20Shot%202021-12-29%20at%2017.32.47.png)

> 将要被测试的服务需要先启动


## 创建配置文件与使用
![](https://raw.githubusercontent.com/johnxue2013/docs/master/images/Screen%20Shot%202021-12-29%20at%2017.37.31.png)

点击右上角，可以创建配置文件，保存一些公共的参数，如host，port等变量

> 当使用IDEA界面创建配置文件时，这个配置文件保存在`Scratches and Consoles`文件夹下，也就是多个项目公用的一个文件夹，如果不想多个项目公用，可以将配置文件剪切到项目的跟目录中，这样http client也可以正常运行

![](https://raw.githubusercontent.com/johnxue2013/docs/master/images/123.png)

在`.http`中使用{{变量名}}，可以使用环境变量



## 保存全局变量
当某些接口请求前，需要传递一些认证信息时，我们可以使用全局变量保存token信息，从而在其他接口中使用