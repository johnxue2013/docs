# Maven

##简介
用于Java平台的**项目构建**、**依赖管理**和**项目信息**管理。
- 项目构建: 一条简单的命令完成项目的编译、运行单元测试、生成文档、打包和部署
- 依赖管理: 通过坐标系统准确定位每一个jar包，maven会自动下载这些jar省去手工
- 项目信息: 管理项目描述、开发者列表、版本控制系统地址、许可证、缺陷管理系统地址等  
> Maven提供了免费的中央仓库，几乎可以找到任何流行的开源类库。      

## 安装配置
参考官网安装[链接][1]  
> 需要先配置好JDK再安装Maven

 maven奉行：约定优于配置( Convention over Configuration)
主要体现在
- 包名`groupid` + `<artifactId>`
- 项目主代码放到`src/main/java`
- 测试代码放到`src/test/java`
- 所有测试方法以test开头
- 项目默认打包类型为jar。即`<packaging>jar</packaging>`



## pom.xml文件  
Maven使用pom.xml配置文件管理项目。结构如下
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.weimob.btrace</groupId>
    <artifactId>btrace-demo</artifactId>
    <version>1.0-SNAPSHOT</version>
</project>
```  
- `<modelVersion>4.0.0</modelVersion>`指定当前pom模型版本，对于maven2和maven3来说值只能是4.0.0
- `<groupId>、<artifactId>`和`<version>`定义了一个项目的基本坐标。任何jar、pom、或者war都是基于这些基本坐标进行区分的  

值规范
`<groupId>`定义项目属于哪个组, 值一般为公司域名的倒序 + 项目名
`<artifactId>`定义项目唯一ID。一般为项目名-模块名.如myapp-util
`<version>`定义了项目的版本  。一般为版本号-[RELEASE | SNAPSHOT]
> 关于RELEASE和SNAPSHOT的区别: SNAPSHOT意味快照，当依赖SNAPSHOT的jar包时，每次build时，总是获取最新的jar包。而使用RELEASE则不是。
 
## 依赖配置
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    
    <modelVersion>4.0.0</modelVersion>
    <groupId>...</groupId>
    <artifactId>...</artifactId>
    <version>...</version>
	  <dependencies>
	  
	    <dependency>
	      <groupId>...</groupId>
	      <artifactId>...</artifactId>
	      <version>...</version>
	      <type>...</type>
	      <scope>...</scope>
	      <optional>...</optional>
	      <exclusions>
		      <exclusion>
		      ...
		      <exclusion>
	      </exclusions>
	    </dependency>
		
		...
  </dependencies>
</project>
```    
根元素`<dependencies>`可以包含一个或多个`<dependency>`以声明一个依赖  
每个依赖可以包含的元素有
- `<groupId>`、`<artifactId>`和`<version>`：依赖的基本坐标。最重要。必须元素  
非必须元素
- `<type>`: 依赖的类型，对应于定义坐标时的`<packaging>`，默认值为jar
- `<scope>`: 依赖范围。默认值为compile
- `<optional>`: 标记依赖是否可选。默认值`false`。
- `<exclusions>`: 用来排除传递性依赖     


##`<scope>` 依赖范围  
依赖范围使用`<scope>`元素标记 。依赖范围实际控制的是classpath。
Maven在`编译`主代码时使用一套classpath，在编译和执行`测试`的时候使用另外一套classpath，最后实际`运行时`，Maven项目的时候又会使用另外一套classpath。

 
`<scope>`的可选值:
1. `compile` : `编译`依赖范围，默认值。对编译、测试、运行三种classpath都有效，如spring-core
2. `test`:  `测试`依赖范围。只对测试classpath有效，在编译主代码或者运行主代码时将无法使用此类。如junit
3. `provided`: 已提供依赖范围。对于编译和测试classpath有效，但在运行时无效。如servlet-api，编译和测试项目的时候都需要该依赖，但在运行项目的时候，由于容器(tomcat、jetty等)已提供，就不需要Maven重复引入一遍。
4. `runtime`:  `运行时`依赖范围。对测试和运行classpath有效，但在编译时无效。如JDBC驱动实现。
5. `system`，系统依赖范围。与provided关系一致。但是使用此依赖必须通过<systemPath>标签显示指定依赖的文件路径。因与本机系统绑定，可能导致构建不够移植。**不推荐使用**  
6. `import`: (Maven2.0.9以上)导入依赖范围。不会对三种classpath产生影响  
  
### classpath
它描述了Java虚拟机在运行一个Class时在哪些路径中加载要运行的类以及运行的类要用到的类。  

可以在启动JVM时指定classpath
```bash
#单个路径
java -classpath /code/workspace/ HelloWord  
#多个classpath
java -cp /code/workspace1;/code/workspace2 HelloWord
```  
JVM将依路径的顺序加载对应的class，先找的先载入。如果都找不到则会出现`java.lang.NoClassDefFoundError`错误。

如果要运行的class中使用其他类库(封装在.jar中)，则执行时使用
```bash
java -cp /code/workspace;/lib/abc.jar;/lib/def.jar SomeApp
```
> classpath的设定是给classload使用的  

依赖范围(scope) | 对于编译classpath有效| 对于测试classpath有效| 对于运行时classpath有效|例子
:--------:|:---------:|:-------:|:-------:|:-------:
compile |Y| Y|Y|spring-core
test |--|Y|--|Junit
provided|Y|Y|--|servlet-api
runtime |--|Y|Y|JDBC驱动实现
system |Y|Y|--|本地的,Maven仓库之外的类库文件

## 依赖范围与依赖传递性
假设A依赖于，B依赖于C，则称A对于B是第一直接依赖，B对于C是第二直接依赖，A对于C是传递性依赖。

第一直接依赖的范围和第二直接依赖的范围决定了传递性依赖的范围。

具体关系如下图所示。![Alt text](./屏幕快照 2017-07-20 09.50.22.png)

> 左边第一行表示第一直接依赖，最上面第一行表示第二直接依赖，交叉点表示传递性依赖范围。  

## 依赖调解  
Maven依赖调解两原则: 
1. 路径最近者优先
2. 当路径相等时，第一声明者优先（在pom.xml中的顺序）。

## `<optional>`可选依赖
假设存在依赖关系： 项目A依赖于项目，B依赖于项目X和Y，**B对X和Y都是可选性依赖**。根据依赖传递性，如果三个依赖的范围都是compile，那么X和Y就是A的compile依赖。然而，由于X和Y都是可选依赖，依赖将不会被传递，也就是说X和Y不会对A产生任何影响。  

使用场景：假设B是一个持久层隔离工具包，支持了多种数据库，包括MySQL、PostgreSQL等，在构建B的时候，需要这两种数据库的驱动，但在使用工具包B的时候，只会依赖一种数据库。此时可将B对于MySQL和PostgreSQL的`<scope>`设置为`true`。此时当A依赖B时，项目A需要**显示声明**对那种数据库的依赖。  
> 对于可选依赖，在理想情况下是不应该使用的。根据单一性原则，一个项目不应该柔和太多功能，上面面的例子，更好的做法是为支持MySQL和PostgreSQL分别键两个项目，使用同样的`<groupId>`不同的`<artifactId>`  

## 最佳实践  
###排除依赖
假设当前项目A有一个第三方依赖B，而这个B又依赖了另外一个类库C的SNAPSHOT版本，那么这个SNAPSHOT版本的C就会成为A的传递性依赖，而SNAPSHOT的不稳定性会直接影响到当前项目的性能。此时可以使用`<exclusions>'和'<exclusions>`排除对C的依赖，然后在A中显示的依赖C的稳定版

###依赖归类  
在pom.xml中经常出现对Spring Framework的依赖，如spring-core:2.5.6、spring-mvc:2.5.6、spring-beans:2.5.6。它们都来自同一项目的不同模块，它们的版本号都相同。将来需要升级时，会一起升级。此时可以使用`<properties>`声明变量。如下
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.weimob.btrace</groupId>
    <artifactId>btrace-demo</artifactId>
    <version>1.0-SNAPSHOT</version>
	<properties>
		<spring-framework.version>2.5.6</spring-framework.version>
	</properties>
	
	<dependency>
	      <groupId>...</groupId>
	      <artifactId>...</artifactId>
	      <version>${spring-framework.version}</version>
	    </dependency>
	    
		<dependency>
	      <groupId>...</groupId>
	      <artifactId>...</artifactId>
	      <version>${spring-framework.version}</version>
	    </dependency>
</project>

```

###优化依赖  
由于Maven能够自动解析所有项目的直接依赖和传递性依赖，并且根据规则正确判断每个依赖范围，最后得到的那些依赖成为已解析依赖(Resolved Dependency)。

1. `mvn dependency:list`
查看项目当前已解析的依赖以及其依赖范围
![list](https://github.com/johnxue2013/docs/blob/master/images/maven/dependency.list.png)
> 直接在pom.xml声明的依赖定义为顶层依赖，顶层依赖的依赖定义为第二层依赖，以此类推。  

2. `mvn dependency:tree`  
可以查看项目的多层依赖
![tree](https://github.com/johnxue2013/docs/blob/master/images/maven/dependency.tree.png)

3. `mvn denpendency:analyze`
![analyse](https://github.com/johnxue2013/docs/blob/master/images/maven/analyse.png)
> 结果分为两部分
- `Used undeclared dependencies found`:
意为项目中使用到的，但是没有显式声明的依赖。这种依赖意味着潜在的风险。**应当显式声明任何项目中直接用到的依赖**。
- `Unused declared dependencies found`
意为项目中未使用的，但显式声明的依赖。对于这类依赖，不应该直接简单的删除其依赖声明，而是应该仔细分析。

## 仓库
Maven仓库分为两类
- 本地仓库
- 远程仓库 
	- 中央仓库(默认)
	- 私服
	- 其他公共库

### 本地仓库
Maven会将pom.xml中声明的依赖下载到用户本地文件系统。这个存放目录称为本地仓库。配置地址在
`<maven-home>/conf/settings.xml`的`<localRepository>/Users/johnxue/.m2/repository</localRepository>`标签中指定。  

一个构建只有在本地仓库中之后，其他Maven项目才能使用，一般构建有Maven自动下载到本地仓库中。如果想要将本地项目的构建安装到本地仓库中可以使用`mvn clean install`。  

> 本地仓库只能有一个，远程仓库可以有多个

###私服
代理广域网上的远程仓库，供局域网内的Maven用户使用。当Maven需要下载构建的使用，它从私服请求，如果私服上不存在，则从外部的远程仓库下载，缓存在私服上后，再为Maven的下载请求提供服务。另外一般公司都会有自己的私有构建，也可以上传到私服供大家使用。
![server](https://github.com/johnxue2013/docs/blob/master/images/maven/server.png)

##远程仓库的配置
当中央仓库无法满足项目的需求，就需要在POM中配置其他远程仓库。
```xml
<project>
	...
	<repositories>
        <repository>
            <id>jboss</id>
            <name>JBOSS Repository</name>
            <url>http://http://repository.jboss.com/maven2</url>
            <releases>
                <enabled>true</enabled>
            </releases>
            <snapshots>
                <enabled>false</enabled>
            </snapshots>
            <layout>default</layout>
        </repository>
    </repositories>
    ...
</project>
```
>在`<repositories>`元素下可以声明一个或多个远程仓库，`<id>`元素的值必须是唯一的。
```xml
<releases>
     <enabled>true</enabled>
 </releases>
 <snapshots>
     <enabled>false</enabled>
 </snapshots>
```
表示开启此仓库的发布版本支持，关闭快照版本的支持。因此Maven只会从该仓库下载发布版构件，而不会下载快照版本构件。  

当访问的远程仓库需要认证才能访问，此时需要在`<maven-home>/conf/settings.xml`中配置
```xml
	<settings>
		...
        <server>
            <id>releases</id>
            <username>admin</username>
            <password>123456aA</password>
        </server>
        <server>
        ...
      </settings>
```
`<id>`的值必须与POM中需要认证的repository元素的id完全一致。

## 部署至远程仓库
Maven可以将项目生成的构件部署到仓库中供他人使用。
配置如下
修改项目的pom.xml文件添加`<distributionManagement>`：
```xml
	<project>
		...
		<distributionManagement>
			<!--发布版本构件地址-->
	      <repository>
	         <id>releases</id>
	         <url>${releases.repo}</url>
	      </repository>
	      <!--快照版本构件地址-->
	      <snapshotRepository>
	         <id>snapshots</id>
	         <url>${snapshots.repo}</url>
	      </snapshotRepository>
	   </distributionManagement>
	   ...
   </project>
```  
配置正确后运行`mvn clean deploy`命令，Maven会将项目构建输出的构件部署到对应的远程仓库  

##生命周期和插件
日常使用中，命令行的输入就对应了Maven的生命周期。Maven的生命周期是抽象的，其行为由具体的插件（plugin）来完成。生命周期和插件两者协同工作密不可分。

###生命周期
对所有构建过程进行抽象和统一。  
Maven共有三套互相独立的生命周期分别是
- clean ：清理项目
- default：构建项目（定义了构建项目的所有阶段）
- site：构建项目站点  
每个生命周期包含一些阶段(phase)，这些phase是有顺序的，且后面的阶段依赖前面的阶段

> default包含的阶段
- validate
- initialize
- generate-sources
- process-sources
- generate-resources
- process-resources
- compile
- process-classes
- generate-test-sources
- process-test
...

### 插件目标
一个插件(plugin)会有一个或多个目标(goal)。当运行如`mvn dependency:tree`意思是运行该插件的tree目标。  

###插件绑定
Maven的生命周期与插件互相绑定，用以完成实际构建任务。即生命周期的阶段与插件的目标互相绑定，以完成某个具体的构件任务。
> Maven已经为主要的生命周期阶段绑定了很多插件目标，几乎不用配置

##Maven多模块(聚合)
为了能够使用一条命令构建两个项目，需要创建另外一个Maven项目，且该项目的pom.xml有特殊的要求
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.weimob.activity</groupId>
  <artifactId>activity-smashgoldenegg</artifactId>
  <version>0.0.1-SNAPSHOT</version>
  <packaging>pom</packaging>
  <name>activity-smashgoldenegg</name>
  
 <modules>
    <module>activity-smashgoldenegg-interface</module>
    <module>activity-smashgoldenegg-pc</module>
    <module>activity-smashgoldenegg-mobile</module>
  </modules>

</project>
```  

此pom.xml中的`<packaging>`值必须为pom，通过`modules`标签构成了Maven的多模块。

## 继承



[1]:https://maven.apache.org/install.html "Maven安装步骤"








