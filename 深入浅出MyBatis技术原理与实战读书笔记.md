## 深入浅出MyBatis技术原理与实战读书笔记
### 2.2mybatis基本构成
- `SqlSessionFactoryBuilder`(构造器):它会根据配信息或者代码来生成SqlSessionFactory(工厂接口)
- `SqlSessionFactory`:依靠工厂来生成SqlSession
- `SqlSession`:是一个既可以发送Sql去执行并返回结果，也可以获取Mapper的接口。
- `SqlMapper`:它是MyBatis型设计的组件，它是 由一个Java接口和XML文件（或注解）构成的，需要给出对应的sql和映射规则，它负责发送SQL去执行，并返回结果。  

  ```xml
  <?xml version="1.0" encoding="UTF-8" ?>
  <!DOCTYPE configuration PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
  		"http://mybatis.org.dtd/mybatis-3-config.dtd">
  <configuration>
  	<environments default="develop">
  		<environment id="develop">
  			<transactionManager type="JDBC"></transactionManager>
  			<dataSource type="POOLED">
  				<property name="driver" value="com.mysql.jdbc.Driver"/>
  				<property name="url" value="jdbc:mysql://localhost:3306/mybatis"/>
  				<property name="username" value="root"/>
  				<property name="password" value="root"/>
  			</dataSource>
  		</environment>
  	</environments>

  <!--引入映射器	-->
  	<mappers>
  		<mapper resource="mapper/RoleMapper.xml"></mapper>
  	</mappers>
  </configuration>
  ```

 MyBatis的配置文件中`<transactionManager>`标签的type属性有三个值：  
 1. JDBC:采用JDBC方式管理实务，在独立编码中我们常常使用。
 2. MANAGED:采用容器方式管理事务，在JNDI数据源中使用。
 3. 自定义，由使用者自定义数据库事务管理办法，适用于特殊应用  


 `<dataSource>`标签的type属性是提供我们对数据库连接方式的配置，值有
  1. UNPOOOLED:非连接池数据源库(UnPooledDataSsource类实现)
  2. POOLED:连接池数据库(PooledDataSource类实现)
  3. JNDI:JNDI数据源(JNDIDataSource实现)
  4. 自定义数据源

数据库事务交由 `SqlSession`控制的，可以使用`SqlSession` commit或者rollback，在大部分情况下，我们交由Spring来控制它。

### 引入映射器(Mapper)的方法
引入映射器的方法很多，如
1. 用文件路径引入映射器  

  ```XML
  <?xml version="1.0" encoding="UTF-8" ?>
  <!DOCTYPE configuration PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
  		"http://mybatis.org.dtd/mybatis-3-config.dtd">
  <configuration>
  	<environments default="develop">
  		<environment id="develop">
  			<transactionManager type="JDBC"></transactionManager>
  			<dataSource type="POOLED">
  				<property name="driver" value="com.mysql.jdbc.Driver"/>
  				<property name="url" value="jdbc:mysql://localhost:3306/mybatis"/>
  				<property name="username" value="root"/>
  				<property name="password" value="root"/>
  			</dataSource>
  		</environment>
  	</environments>

  <!--用文件路径引入映射器 	-->
  	<mappers>
  		<mapper resource="mapper/RoleMapper.xml"></mapper>
  	</mappers>
  </configuration>
  ```

2. 用包名引入映射器    

  ```xml
  <?xml version="1.0" encoding="UTF-8" ?>
  <!DOCTYPE configuration PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
      "http://mybatis.org.dtd/mybatis-3-config.dtd">
  <configuration>
    <environments default="develop">
      <environment id="develop">
        <transactionManager type="JDBC"></transactionManager>
        <dataSource type="POOLED">
          <property name="driver" value="com.mysql.jdbc.Driver"/>
          <property name="url" value="jdbc:mysql://localhost:3306/mybatis"/>
          <property name="username" value="root"/>
          <property name="password" value="root"/>
        </dataSource>
      </environment>
    </environments>

  <!--用包名引入映射器 -->
    <mappers>
      <mapper resource="mapper/mapper/"></mapper>
    </mappers>
  </configuration>
  ```
3. 用类注册引入映射器

  ```xml
  <?xml version="1.0" encoding="UTF-8" ?>
  <!DOCTYPE configuration PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
  		"http://mybatis.org.dtd/mybatis-3-config.dtd">
  <configuration>
  	<environments default="develop">
  		<environment id="develop">
  			<transactionManager type="JDBC"></transactionManager>
  			<dataSource type="POOLED">
  				<property name="driver" value="com.mysql.jdbc.Driver"/>
  				<property name="url" value="jdbc:mysql://localhost:3306/mybatis"/>
  				<property name="username" value="root"/>
  				<property name="password" value="root"/>
  			</dataSource>
  		</environment>
  	</environments>

    <!--用类注册引入映射器	-->
  	<mappers>
  		<mapper class="com.demo.mapper.RoleMapper"></mapper>
  	</mappers>
  </configuration>
  ```

4. 用RoleMapper.xml引入  

  ```xml
  <?xml version="1.0" encoding="UTF-8" ?>
  <!DOCTYPE configuration PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
  		"http://mybatis.org.dtd/mybatis-3-config.dtd">
  <configuration>
  	<environments default="develop">
  		<environment id="develop">
  			<transactionManager type="JDBC"></transactionManager>
  			<dataSource type="POOLED">
  				<property name="driver" value="com.mysql.jdbc.Driver"/>
  				<property name="url" value="jdbc:mysql://localhost:3306/mybatis"/>
  				<property name="username" value="root"/>
  				<property name="password" value="root"/>
  			</dataSource>
  		</environment>
  	</environments>

    <!--用RoleMapper.xml引入	-->
  	<mappers>
  		<mapper url="file:///mapper/RoleMapper.xml"></mapper>
  	</mappers>
  </configuration>
  ```

#### MyBatis语句中的#{}和${}的区别

  #{}: 解析为一个 JDBC 预编译语句（prepared statement）的参数标记符，一个 #{ } 被解析为一个参数占位符 。

   ${}: 仅仅为一个纯碎的 string 替换，在动态 SQL 解析阶段将会进行变量替换。

  如name值为cy
  ```sql
  select id,name,age from student where name=#{name}   -- name='cy'

  select id,name,age from student where name=${name}    -- name=cy
  ```
  使用#可以很大程度上防止sql注入。(语句的拼接)
