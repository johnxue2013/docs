# 如何在idea中断点调试maven的plugin
假设现在开发了一个插件A(项目名称为A-maven-plugin，该插件包含goal=gen),在发布到私服或者执行```mvn
install```安装到本地仓库后。在项目B中引用插件A。

首先在项目B的Terminal中使用命令```mvnDebug A:gen```开启插件A的debug模式。
此时将进入如下状态 :

![image](https://github.com/johnxue2013/docs/blob/master/images/maven/mvnDebug.png)  
如上图显示，此时该插件已处于debug状态，并监听本机8000端口，

此时使用Idea新窗口打开插件的源码(项目A-maven-plugin)，并在需要调试的地方打上断点。
并点击项目运行配置，

![image](https://github.com/johnxue2013/docs/blob/master/images/maven/start-config.png)

进行如下配置:

![image](https://github.com/johnxue2013/docs/blob/master/images/maven/mvnDebug.png)

配置好后点击确定，点击上上图中的debug按钮，就可以在开心的debug了.
