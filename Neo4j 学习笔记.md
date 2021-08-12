# Neo4j 学习笔记
Neo4j是一个世界领先的开源图形数据库(Graph Database)。
- Open Source
- Optional Schema  
- No SQL
- ACID and Transaction Support

> 有些文档写成了No Scheme，根据最新的官方文档,Schema是可选的，非必须的

有两个版本Community Edition(免费)和Enterprise Edition(收费)

## 基本概念 (Concepts)
使用下方的图来介绍Neo4j的基本概念  

![](https://data-repository-01.oss-cn-shanghai.aliyuncs.com/img/20210808111405.png)

### 节点(Node)
节点通常用来表示实体(entities)，最简单的图就是单个Node。如下方这个图  

![](https://data-repository-01.oss-cn-shanghai.aliyuncs.com/img/Screen%20Shot%202021-08-08%20at%2011.16.33.png)

> Node在Neo4j的WebUI中展示为一个圆形

### 标签(Labels)
标签用于分组节点，具有相同标签的Node属于相同的集合，举例来说，所有代表用户的Node可以使用`User`标签来标记，当Node具有标签后，就可以使用一些查询语句来对指定的Node进行一些查询或者更新操作。

一个Node可以有0个或多个标签,如下方的图，在上面的图中节点具有Person和Movie标签，这是一种描述数据的方法，但当我们需要不同维度去操作数据时，我们可以给Node添加个多的标签，如下图
![](https://data-repository-01.oss-cn-shanghai.aliyuncs.com/img/Screen%20Shot%202021-08-08%20at%2011.25.22.png)

### 关系(Relationships)
中文一般译作关系或者边,一个关系连接两个Node，关系将节点组织成结构，使图类似于列表、树、地图或复合实体 —— 其中任何一个都可以组合成更复杂、相互关联丰富的结构。

![](https://data-repository-01.oss-cn-shanghai.aliyuncs.com/img/20210808111405.png)

#### 关系类型(Relationship Types)
一个关系必须具有明确的关系类型，上述的例子中使用了`ACTED_IN`和`DIRECTED`作为关系类型。`roles`作为关系`ACTED_IN`的属性，是一个数组，数组仅有一个值。

下方是一个`ACTED_IN`关系，节点Tom Hanks作为一个起始节点，节点Forrest Gump作为目标节点  

![](https://data-repository-01.oss-cn-shanghai.aliyuncs.com/img/20210808113727.png)

**关系总是有方向的**. 但只需注意它有用的方向。这意味着没有必要在相反的方向添加重复的关系，除非需要它来正确描述你的用例(因为在查询时，关系是可以忽略的)。

另外，一个节点可以有一个指向自身的关系，介绍想要描述TomHanks了解他自己，那么图可以描述为  

![](https://data-repository-01.oss-cn-shanghai.aliyuncs.com/img/Screen%20Shot%202021-08-08%20at%2011.43.16.png)

### 属性 (Properties)
属性是键值对(key-value)，可以添加在Node和Relationship上

### 遍历和路径(Traversals and paths)
遍历是你查询图形以找到问题答案的方式，例如：“我的朋友喜欢哪些音乐但我尚未拥有？”或“如果此电源中断，哪些Web服务会受到影响？”

遍历一个图意味着根据一些规则通过关系来访问节点。在大多数情况下，只会访问图的一个子集。

假设我们想要在下图中找出Tom Hanks主演了哪些电影  

![](https://data-repository-01.oss-cn-shanghai.aliyuncs.com/img/Screen%20Shot%202021-08-08%20at%2011.52.19.png)  

那么遍历将从Tom Hanks节点开始，查询具有`ACTED_IN`关系连接的节点，那么将查询到Forrest Gump这个节点，查询的结果将以一个path形式返回，该path的长度是1  

![](https://data-repository-01.oss-cn-shanghai.aliyuncs.com/img/Screen%20Shot%202021-08-08%20at%2011.54.58.png)

最短的路径长度可以为0，该路径仅包含一个Node，没有关系，如  

![](https://data-repository-01.oss-cn-shanghai.aliyuncs.com/img/Screen%20Shot%202021-08-08%20at%2011.57.55.png)


### Schema
Neo4j中的Schema指的是索引(Index)和约束(Constrains)。我们可以在没有定义Schema的情况下创建节点，关系和它们的属性，当我们需要更高性能的时候可以定义Schema，也就意味着Schema是可选的，非必须的。

#### 索引(Index)
在图形数据库中使用索引的主要原因是找到图形遍历的起点。一旦找到起点，横向依靠图形结构实现高性能查询

索引可以随时添加。但是，请注意，如果数据库中有现有数据，索引需要过一段时间才可用。


```sql
// 如在Actor标签的节点上创建一个索引，用来提高在数据库中根据姓名查询一个演员
CREATE INDEX FOR (a:Actor) ON (a.name)
```

在大多数情况下，在查询数据时无需指定索引，因为相应的索引将自动使用。例如，以下查询将自动使用上述定义的索引：
```sql
MATCH (actor:Actor { name: "Tom Hanks" })
RETURN actor;
```

##### 联合索引 (composite index)
联合索引是具有特定标签的所有节点的多个属性的索引。例如，以下语句将在具有标签`Actor`的所有节点上创建一个联合索引
```sql
CREATE INDEX FOR (a:Actor) ON (a.name, a.born)
```
这些节点同时具有名称和出生属性。请注意，如果某个节点仅有name属性,没有born属性。那么该节点不会添加到索引中。

> 可以使用CALL db.indexes查询已经定义的索引

#### 约束 (Constrains)
约束用于确保数据符合域的规则。如：如果一个节点具有`Actor`标签和属性name，那么name属性的值在具有`Actor`标签的节点中必须唯一
```sql
CREATE CONSTRAINT ON (movie:Movie) ASSERT movie.title IS UNIQUE
```
> 添加约束将隐式的添加该属性的索引。如果约束被删除，但仍需要该索引，则必须显式的创建该索引
> 可以使用 CALL db.constraints 查看已创建的约束

最后看一个比较实际的地铁线路图数据库的截图
![](https://data-repository-01.oss-cn-shanghai.aliyuncs.com/img/Screen%20Shot%202021-08-08%20at%2013.33.48.png)

## Cypher
Neo4j使用Cypher来作为查询语言，类似Mysql的查询语言简写为SQL, Cypher查询语言简写为CQL

## 简介
Cypher为许多数据类型提供支持。具体分为以下几个类别
- 属性类型
Integer, Float, String, Boolean, Point, Date, Time, LocalTime, DateTime, LocalDateTime, and Duration.

- 结构类型  
Node, Relationship, and Path.
结构类型的属性不可以包含null值，设置null值可以删除这个属性

- 集合类型  
List and Map
集合类型的值可以包含null值

上述三种类型中,属性类型都是一些基本类型，无需赘述，主要说明结构类型

### Node
包含以下属性
- Id
- Labels (标签不是Node的一个值，而是Cypher的一个语法形式)
- Map (of properties)

### Relationship
包含以下属性
- Id
- Type
- Map(of properties)
- Id of the start node
- Id of then end node

在Cypher查询语句中关系可以表述为三种: --,-->,<--,
在查询时，如果忘记了冒号,如-[LIKES]->,那么Cypher将理解为这是一个变量，变量名为LIKES，并不会理解成一个关系类型，查询时将查询所有关系

通常情况下，关系类型不允许包含空格等特殊字符，如果一定要包含，可以使用``符号处理
```sql
// 创建包含空格的关系类型
MATCH
  (charlie:Person {name: 'Charlie Sheen'}),
  (rob:Person {name: 'Rob Reiner'})
CREATE (rob)-[:`TYPE INCLUDING A SPACE`]->(charlie)
```

当然，查询这个关系时，也需要使用``

```sql
MATCH (n {name: 'Rob Reiner'})-[r:`TYPE INCLUDING A SPACE`]->()
RETURN type(r)
```

### Path
路径`Path`代表着节点和关系的交替序列

>**节点、关系和路径作为模式匹配的结果返回。在Neo4j中，所有关系都有一个方向。但是，你可以在查询时拥有无向关系的概念。也就是可以在查询时忽略关系的方向**

## 命名规则和建议（Naming rules and recommendations）
- 必须以英文字幕开头，开头不可以是数字。除了下划线_和美元符号$不可以包含其他符号
- 大小写敏感,:PERSON, :Person and :person are three different labels, and n and N are two different variables.
- 关于空格
开头和结尾的空格将自动被删除，如 MATCH ( a ) RETURN a 和 to MATCH (a) RETURN a是相同的

Here are the recommended naming conventions:

|      |  | |
| ----------- | ----------- |------------|
| Node labels      | Camel-case, beginning with an upper-case character |:VehicleOwner rather than :vehicle_owner etc.
| Relationship types   | Upper-case, using underscore to separate words  | :OWNS_VEHICLE rather than :ownsVehicle etc.

### Cypher 语法简介
#### Cypher的注释
使用//
```sql
MATCH (n)
//This is a whole line comment
RETURN n
```

```sql
MATCH (n) RETURN n //This is an end of line comment
```

下方这个不是注释
```sql
MATCH (n) WHERE n.property = '//This is NOT a comment' RETURN n
```

#### CREATE
用来新增数据，类似SQL中的`CREATE`关键字,可以用来新增节点，关系，图
```sql
CREATE (friend:Person {name: 'Mark'})
RETURN friend
```

#### MERGE
类似`CREATE`，只是如果存在时将不会创建

#### SET
用来更新节点或者关系的属性，一般搭配`MATCH`使用，然后使用`SET`来添加、删除、或者更新属性

```sql
MATCH (p:Person {name: 'Jennifer'})
SET p.birthdate = date('1980-01-01')
RETURN p
```
> date()是Cypher一个内置的时间函数，更多时间函数可以参考https://neo4j.com/docs/developer-manual/3.4/cypher/syntax/temporal/

```sql
MATCH (n {name: 'Andy'})
SET n.name = null
RETURN n.name, n.age
```
#### DELETE
可以删除节点、关系等
```sql
// 删除IS_FRIENDS_WITH关系
MATCH (j:Person {name: 'Jennifer'})-[r:IS_FRIENDS_WITH]->(m:Person {name: 'Mark'})
DELETE r
```

```sql
//删除name为'Mark'的节点
MATCH (m:Person {name: 'Mark'})
DELETE m
```  

注意当m节点存在关系时，是无法删除的，需要先解除节点的所有关系后才可以删除
如果想要同时删除节点和节点的关系可以使用`DETACH DELETE`  

```sql
MATCH (m:Person {name: 'Mark'})
DETACH DELETE m
```

RETURN关键词不是必须的，如果不想返回任何结果，你可以这样写
```sql
CREATE (friend:Person {name: 'Mark'})
```

### MATCH
关键词`MATCH`可以用来查询存在的节点，关系，标签、属性，图。类似SQL中个`SELECT`关键词。

```sql
MATCH (n)
RETURN
CASE n.eyes
  WHEN 'blue'  THEN 1
  WHEN 'brown' THEN 2
  ELSE 3
END AS result
```

### RETURN
`RETURN`关键词用来指定一个查询之后，想要返回什么结果，可以返回节点，关系，节点和关系属性或者图

```sql
MATCH (p:Person)
RETURN p
LIMIT 1
```

```sql
//cleaner printed results with aliasing
MATCH (tom:Person {name:'Tom Hanks'})-[rel:DIRECTED]-(movie:Movie)
RETURN tom.name AS name, tom.born AS `Year Born`, movie.title AS title, movie.released AS `Year Released`
```

### 函数
- DISTINCT
```sql
CREATE
  (a:Person {name: 'Anne', eyeColor: 'blue'}),
  (b:Person {name: 'Bill', eyeColor: 'brown'}),
  (c:Person {name: 'Carol', eyeColor: 'blue'})
WITH [a, b, c] AS ps
UNWIND ps AS p
RETURN DISTINCT p.eyeColor
```

> WITH...AS... 把WITH右侧的当成一个查询结果，在这个基础上再做处理,AS为这结果集的别名
> UNWIND...AS... 表示遍历集合，AS为集合中的每一项

结果为：
|   p.eyeColor   |  
| ----------- | 
| "blue"      | 
| "brown"   | 

`DISTINCT`经常和聚合函数一起使用

## 例子
### 快速搭建Neo4j环境
推荐使用Docker
先拉取下镜像:
```bash
docker pull neo4j:3.5.22-community
```

运行容器
```bash
docker run --name neo4j -d \
    --publish=7474:7474 --publish=7687:7687 \
    --volume=/Users/johnxue/docker_data/neo4j/data:/var/lib/neo4j/import \
    neo4j:3.5.22-community
```
> 上面的挂载卷是可选的，如果不需要使用neo4j的import导入功能的话，可以不挂，记得修改成自己本机的路径，如果不想链接neo4j时需要密码，可以加入--env=NEO4J_AUTH=none参数

启动完成后访问http://localhost:7474/browser/ 即可

### 导入数据
下载数据https://github.com/johnxue2013/docs/blob/master/images/subway_data.zip，后执行neo4j的import命令将数据导入

- 建立地铁站点
```sql
LOAD CSV WITH HEADERS  FROM "file:///station.csv" AS line
MERGE (p:Station{id:line.id,name:line.name});
```

- 建立站点连接
```sql
LOAD CSV WITH HEADERS FROM "file:///line1.csv" AS line1
match (from1:Station{name:line1.sn}),(to1:Station{name:line1.en})
merge (from1)-[r1:一号线{jl:line1.jl,xl:line1.xl}]->(to1);

LOAD CSV WITH HEADERS FROM "file:///line2.csv" AS line2
match (from2:Station{name:line2.sn}),(to2:Station{name:line2.en})
merge (from2)-[r2:二号线{jl:line2.jl,xl:line2.xl}]->(to2);

LOAD CSV WITH HEADERS FROM "file:///line4.csv" AS line4
match (from4:Station{name:line4.sn}),(to4:Station{name:line4.en})
merge (from4)-[r4:四号线{jl:line4.jl,xl:line4.xl}]->(to4);

LOAD CSV WITH HEADERS FROM "file:///line5.csv" AS line5
match (from5:Station{name:line5.sn}),(to5:Station{name:line5.en})
merge (from5)-[r5:五号线{jl:line5.jl,xl:line5.xl}]->(to5);

LOAD CSV WITH HEADERS FROM "file:///line6.csv" AS line6
match (from6:Station{name:line6.sn}),(to6:Station{name:line6.en})
merge (from6)-[r6:六号线{jl:line6.jl,xl:line6.xl}]->(to6);

LOAD CSV WITH HEADERS FROM "file:///line7.csv" AS line7
match (from7:Station{name:line7.sn}),(to7:Station{name:line7.en})
merge (from7)-[r7:七号线{jl:line7.jl,xl:line7.xl}]->(to7);

LOAD CSV WITH HEADERS FROM "file:///line8.csv" AS line8
match (from8:Station{name:line8.sn}),(to8:Station{name:line8.en})
merge (from8)-[r8:八号线{jl:line8.jl,xl:line8.xl}]->(to8);

LOAD CSV WITH HEADERS FROM "file:///line9.csv" AS line9
match (from9:Station{name:line9.sn}),(to9:Station{name:line9.en})
merge (from9)-[r9:九号线{jl:line9.jl,xl:line9.xl}]->(to9);

LOAD CSV WITH HEADERS FROM "file:///line10.csv" AS line10
match (from10:Station{name:line10.sn}),(to10:Station{name:line10.en})
merge (from10)-[r10:十号线{jl:line10.jl,xl:line10.xl}]->(to10);

LOAD CSV WITH HEADERS FROM "file:///line13.csv" AS line13
match (from13:Station{name:line13.sn}),(to13:Station{name:line13.en})
merge (from13)-[r13:十三号线{jl:line13.jl,xl:line13.xl}]->(to13);

LOAD CSV WITH HEADERS FROM "file:///line14.csv" AS line14
match (from14:Station{name:line14.sn}),(to14:Station{name:line14.en})
merge (from14)-[r14:十四号线{jl:line14.jl,xl:line14.xl}]->(to14);

LOAD CSV WITH HEADERS FROM "file:///line15.csv" AS line15
match (from15:Station{name:line15.sn}),(to15:Station{name:line15.en})
merge (from15)-[r15:十五号线{jl:line15.jl,xl:line15.xl}]->(to15);

LOAD CSV WITH HEADERS FROM "file:///linebt.csv" AS linebt
match (frombt:Station{name:linebt.sn}),(tobt:Station{name:linebt.en})
merge (frombt)-[rbt:八通线{jl:linebt.jl,xl:linebt.xl}]->(tobt);

LOAD CSV WITH HEADERS FROM "file:///linecp.csv" AS linecp
match (fromcp:Station{name:linecp.sn}),(tocp:Station{name:linecp.en})
merge (fromcp)-[rcp:昌平线{jl:linecp.jl,xl:linecp.xl}]->(tocp);

LOAD CSV WITH HEADERS FROM "file:///lineyz.csv" AS lineyz
match (fromyz:Station{name:lineyz.sn}),(toyz:Station{name:lineyz.en})
merge (fromyz)-[ryz:亦庄线{jl:lineyz.jl,xl:lineyz.xl}]->(toyz);

LOAD CSV WITH HEADERS FROM "file:///linedx.csv" AS linedx
match (fromdx:Station{name:linedx.sn}),(todx:Station{name:linedx.en})
merge (fromdx)-[rdx:大兴线{jl:linedx.jl,xl:linedx.xl}]->(todx);

LOAD CSV WITH HEADERS FROM "file:///linefs.csv" AS linefs
match (fromfs:Station{name:linefs.sn}),(tofs:Station{name:linefs.en})
merge (fromfs)-[rfs:房山线{jl:linefs.jl,xl:linefs.xl}]->(tofs);

LOAD CSV WITH HEADERS FROM "file:///linejc.csv" AS linejc
match (fromjc:Station{name:linejc.sn}),(tojc:Station{name:linejc.en})
merge (fromjc)-[rjc:机场线{jl:linejc.jl,xl:linejc.xl}]->(tojc);
```

- 查看下已建立的站点和关系
```sql
match (n) return n
```
![](https://data-repository-01.oss-cn-shanghai.aliyuncs.com/img/Screen%20Shot%202021-08-08%20at%2013.33.48.png)

- 查询
路径检索

```sql
// 查询九龙山站点出发，步长为1的关系(这里可以理解为地铁的多少号线)
MATCH (a:Station {name: '九龙山'})
RETURN [(a)-[r]->(b) WHERE b:Station | type(r)] as rl
```


以‘霍营’与‘北京南站’地铁站为例，检索具体一下路径：

最少站点路径
```sql
MATCH (p1:Station {name:"霍营"}),(p2:Station{name:"北京南站"}),p=shortestpath((p1)-[*]-(p2)) RETURN p
```

最短路程路径
```sql
MATCH p=(b:Station{name:"霍营"})-[*..20]->(d:Station{name:"北京南站"}) WITH p,reduce(s = 0, r IN relationships(p) | s + r.jl) AS dist return p ORDER BY dist DESC limit 1
```


## 其他常用查询

```sql
// 删除带有Node标签的所有关系
match (n:Node)-[r*]-(m:Node) with r as rls unwind rls as rl delete rl;
```
> [r*]表示任意任意步长的关系

```sql
// 取集合中指定的key的value作为结果集返回
MATCH (a:Person {name: 'Keanu Reeves'})
RETURN [(a)-->(b) WHERE b:Movie | b.released] AS years
```
> 这里的|不是"或"的意思，而是管道

```sql
// 删除所有节点和关系
match (n) detach delete n
```

## 使用Java语言操作Neo4j

Please note the minimum version compatibilities:  

- Neo4j Server 3.x - Java 8
- Neo4j Server 4.x - Java 11

参考代码
https://github.com/johnxue2013/neo4j-java-cypher-demo/tree/master


### 参考资料
- [neo4j Cypher Manual](https://neo4j.com/docs/cypher-manual/current/clauses/set/#set-remove-a-property)
- [Graph Database Concepts](https://neo4j.com/docs/getting-started/current/graphdb-concepts/#graphdb-concepts)
- [动手构建地铁关系网，实现最短路径查询](https://segmentfault.com/a/1190000039188884)
- [Transaction management](https://neo4j.com/docs/java-reference/current/transaction-management/)
- [The Neo4j Java Developer Reference v4.3](https://neo4j.com/docs/java-reference/current/)
- [Using Neo4j from Java](https://neo4j.com/developer/java/)
- [Neo4j Community Edition vs Neo4j Enterprise Edition](https://neo4j.com/licensing/)