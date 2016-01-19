#idea发布JavaEE项目到Tomcat需要的配置
本文不涉及idea的使用，和idea中的各种概念，只讲如何在idea中配置和发布JavaWeb项目。

**环境说明:**

● 编辑工具：idea version 14.1.1 或idea version 15.0.1  

● 编译工具:jdk1.7 或者更高  

● 服务器:tomcat7或更高  

##git clone代码并在idea中打开
1、由于项目托管在gitlab上，所以先用git clone命令将项目源码clone到本地硬盘。如果项目已经存在于本地，则可跳过此步骤。  
![image](https://github.com/johnxue2013/tools/blob/master/images/1.png)

2、打开idea，选择open，找到刚才clone下来的项目，选中，点击ok。
![image](https://github.com/johnxue2013/tools/blob/master/images/2.png)

3、此时idea将加载项目代码，以及项目中使用的框架或者其他的管理工具，如maven。如果项目使用maven进行管理，idea则将询问是否自动加载项目中的依赖，选择自动加载就好。
 ![image](https://github.com/johnxue2013/tools/blob/master/images/3.png)
 ![image](https://github.com/johnxue2013/tools/blob/master/images/4.png)
##配置idea
**配置项目**  

打开项目配置面板: 可以通过idea顶部操作栏中的File->Project Structure打开项目配置面板。也可以通过右上角快捷方式打开
 ![image](https://github.com/johnxue2013/tools/blob/master/images/5.png)
 ![image](https://github.com/johnxue2013/tools/blob/master/images/6.png)
1）首先配置JDK  
 ![image](https://github.com/johnxue2013/tools/blob/master/images/7.png)
点击左侧Project，在右侧选择New...,选择JDK，弹出文件列表窗口，选择你本机JDK的安装位置后点击OK。JDK配置完毕。  

2）配置jar包  
![image](https://github.com/johnxue2013/tools/blob/master/images/8.png)

点击左侧Libraries，点击右侧操作部分的左上角＋，选择java，选择需要添加的jar包。
 
选中要添加的jar包（添加某个文件下的所有jar包，选择文件夹就可以了），点击OK。
 
 ![image](https://github.com/johnxue2013/tools/blob/master/images/9.png)
至此项目依赖配置完毕。  

>注：因为项目使用了maven管理jar包，在上面的步骤中，设置了自动加载maven中的依赖，所以第一次配置Libraries依赖时，里面已经包含了maven中配置好的依赖。

3）配置Modules

点击Modules
 
 ![image](https://github.com/johnxue2013/tools/blob/master/images/10.png)
 
>注：idea中的modules，就是eclipse中的project的概念，idea中的Project，等于eclipse中的workspace概念。点击Sources标签，在此处，你可以配置哪些文件夹中的内容是源码，哪些是测试文件，此处可以看出Java文件是蓝色，代表Sources，表示从java文件下的com开始算作buildpath，即位于java文件下的.java文件中的包声明从 package com.XXX.XXX 开始。如果项目中的.java文件中包声明出现编译错误，可以从此处修改解决。 

点击Paths标签
 ![image](https://github.com/johnxue2013/tools/blob/master/images/11.png)
此处配置项目代码.java文件编译之后的.class文件存放处。选择Use module compile output path。下面路径默认就可以。

点击Dependencies
 ![image](https://github.com/johnxue2013/tools/blob/master/images/12.png)
此处配置项目依赖中的jar包Scope，因为有些jar包发布时时不需要的，比如说junit的jar。

4）配置Artifacts  

配置Artifacts就是配置项目部署时需要哪些文件。
点击Artifacts,后点击+，选择Web Application: Exploded
 ![image](https://github.com/johnxue2013/tools/blob/master/images/13.png)
在Name中输入你想输入的名称,一般和项目名相同。
![image](https://github.com/johnxue2013/tools/blob/master/images/14.png)
点击Output Layout标签
Available Elements标签下列出可使用的文件，
如上面的截图,选中’BudFramework’ compile output （BudFramework这个名称是pom.xml文件中配置的）的文件，右击选择Put into /WEB-INF/classes

![image](https://github.com/johnxue2013/tools/blob/master/images/15.png)
如上面的截图,选中所有以Maven开头标示的文件，右击选择Put into /WEB-INF/lib。
 
![image](https://github.com/johnxue2013/tools/blob/master/images/16.png)
目前项目部署的目录结构为
 
![image](https://github.com/johnxue2013/tools/blob/master/images/17.png)
目前只配置了依赖和.class文件，还需要配置其他类似.html、.css、.js、.jsp文件。具体这些文件存在于哪里，就要看项目的具体目录结构了。本项目存在于webapp目录下。

想添加到根目录就先点![image](https://github.com/johnxue2013/tools/blob/master/images/18.png) 后击点击+， 添加文件的话就选择File,添加文件夹就选择Directory Content。选择后选择具体文件，点击OK。

![image](https://github.com/johnxue2013/tools/blob/master/images/19.png)
当前部署目录层次为
 
![image](https://github.com/johnxue2013/tools/blob/master/images/20.png)
因为在src／com中的源码中还存在配置文件,message.properties文件，该文件项目中配置的是于源码保持一致的目录结构，所以开始添加部署时的自定义文件。

先点击WEB-INF下的classes文件夹后点击![image](https://github.com/johnxue2013/tools/blob/master/images/21.png) ，输入文件名，点击OK
 
 ![image](https://github.com/johnxue2013/tools/blob/master/images/22.png)
如有需要，则继续添加其他文件夹。自定义目录层次添加好后。点击+，添加文件。此项目最终目录如下图所示
 
![image](https://github.com/johnxue2013/tools/blob/master/images/23.png)

点击OK。项目配置完毕。
##配置tomcat  
	1 打开tomcat配置面板  
     
     方法1:点击Edit Configurations
 
 ![image](https://github.com/johnxue2013/tools/blob/master/images/24.png)
     方法2: 点击Edit Configurations
 ![image](https://github.com/johnxue2013/tools/blob/master/images/25.png)

	2）点击左上角+,选择Tomcat Server —> Local。如果看不到Tomcat Server，则请先配置idea中的server
 ![image](https://github.com/johnxue2013/tools/blob/master/images/26.png)
 
输入服务器名，一般和项目同名。在Server标签下可以配置服务器监听的端口等设置。
 
![image](https://github.com/johnxue2013/tools/blob/master/images/27.png)
切换到Deployment标签,点击＋，选择Artifact，选择上面步骤中创建的Artifact，点击OK
 
 ![image](https://github.com/johnxue2013/tools/blob/master/images/28.png)
>Tips：你也可以直接点击Fix。快速添加Artifact。

点击OK退出tomcat配置面板

至此整个部署配置完毕，点击 ![image](https://github.com/johnxue2013/tools/blob/master/images/29.png)
绿色三角形，开始运行项目

 ![image](https://github.com/johnxue2013/tools/blob/master/images/30.png)

启动成功后将自动打开浏览器，并渲染项目首页。

完毕
enjoy idea!
2015-12-26 16:25:45
                                                                                                                   








