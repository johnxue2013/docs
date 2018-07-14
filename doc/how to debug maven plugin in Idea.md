# How to debug maven plugin in Idea
假设现在开发了一个插件A(项目名称为A-maven-plugin，该插件包含goal=gen),在发布到私服或者执行```mvn
install```安装到本地仓库后。在项目B中引用插件A。

首先在项目B的Terminal中使用命令```mvnDebug A:gen```开启插件A的debug模式。
此时将进入如下状态
![image](https://github.com/johnxue2013/tools/blob/master/images/maven/mavenDebug.png)
首先使用Idea打开插件的源码(项目A-maven-plugin)，并在需要调试的地方打上断点。
