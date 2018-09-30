* 全文搜索
全文搜索是指计算机搜索程序通过扫描文章中的每一个词，对每一个词建立一个索引，指明该词在文章中出现的次数和位置。当用户查询时，搜索程序根据事先建立的索引进行查找，并将查找结果反馈给用户。

## 术语和概念
  * 索引词(term)
  是一个能够被索引的精确值，foo、Foo、FOO几个单词是不同的索引词。索引词是可以通过term查询进行准确的搜索。

  * 文本（text）
  文本是一段普通的非结构化文字。通常，文本会被分析成一个个的索引词，存储在ES的索引库中。为了让文本能够进行搜索，文本自毒辣需要事先进行分析；当对文本中的关键词进行查询的时候，搜索引擎应该根据搜索条件搜索出文本。

  * 分析（analysis）
  分析是将文本转换为索引词的过程，分析的结果依赖于分词器。将文本分析成索引词，这些索引词存储在ES的索引库中。

  * 集群（cluster）
  在所有节点中，一个集群只能有一个唯一的名称，默认为”Elasticsearch“。此名称很重要，因为每个节点只能是集群的一部分，当该节点被设置为相同的集群名称时，就会自动加入集群。当需要多个集群的时候，要确保每个集群的名称是不能重复的，否则，节点可能会加入错误的集群。每个集群都有其不同的集群名称。

  * 节点(node)

  * 路由(routing)
  当存储一个文档的时候，它会存在唯一的珠主分片中，具体哪个分片(shard)是通过散列值进行选择，默认情况下，这个值时由文档的ID生成。如果文档有一个指定的父文档，则从父文档ID中生成，该值可以在存储文档的时候进行修改。

  * 分片(sharding)
  分片是单个Lucene实例，

  * 主分片（primary sharding）
    每个文档都存储在一个分片中，当你存储一个文档的时候，系统会首先存贮在主分片中，然后复制到不同的副本中。默认情况下，一个索引有5个主分片。你可以事先指定分片的数量，**当分片一旦建立，则分片数量不能修改**。但副本数量可以修改

  * 副本分片
    默认情况下每个索引分配5个分片和一个副本。

**每个Elasticsearch分片是一个Lucene的索引。有文档存储数量限制**

  * 索引(index)
  具有相同结构的文档集合。**索引名字全部小写**

  Elastic 会索引所有字段，经过处理后写入一个反向索引（Inverted Index）。查找数据的时候，直接查找该索引。

所以，Elastic 数据管理的顶层单位就叫做 Index（索引）。它是单个数据库的同义词。每个 Index （即数据库）的名字必须是小写。

下面的命令可以查看当前节点的所有 Index。


$ curl -X GET 'http://localhost:9200/_cat/indices?v'


  * 类型(type)
  在索引中，可以定义一个或多个类型，类型是索引的逻辑分区。在一般情况下，一种类型被定义为具有一组公共字段的文档。

  * 文档(document)
  Index 里面单条的记录称为 Document（文档）。许多条 Document 构成了一个 Index。

  Document 使用 JSON 格式表示，下面是一个例子。


  {
    "user": "张三",
    "title": "工程师",
    "desc": "数据库管理"
  }
  同一个 Index 里面的 Document，不要求有相同的结构（scheme），但是最好保持相同，这样有利于提高搜索效率。

  * 主键（ID）
    ID 是一个字符串， 当它和 _index 以及 _type 组合就可以唯一确定 Elasticsearch 中的一个文档。 当你创建一个新的文档，要么提供自己的 _id ，要么让 Elasticsearch 帮你生成。

  > es一般会使用两个端口9200用于提供restful服务，另一个9300用于集群通信。

  - 索引文档
  PUT /megacorp/employee/2
  megacorp是索引名(库名)，employee是类型(type)名，2是ID

  - 检索文档
  GET /megacorp/employee/1
  简单地执行 一个 HTTP GET 请求并指定文档的地址——索引库、类型和ID。 使用这三个信息可以返回原始的 JSON 文档：

  - 更新整个文档编辑
在 Elasticsearch 中文档是 不可改变 的，不能修改它们。 相反，如果想要更新现有的文档，需要 重建索引 或者进行替换.在本章的后面部分，我们会介绍 update API, 这个 API 可以用于 partial updates to a document 。 虽然它似乎对文档直接进行了修改，但实际上 Elasticsearch 按前述完全相同方式执行以下过程：

1. 从旧文档构建 JSON
2. 更改该 JSON
3. 删除旧文档
4. 索引一个新文档
唯一的区别在于, update API 仅仅通过一个客户端请求来实现这些步骤，而不需要单独的 get 和 index 请求。

### 处理冲突
当我们之前讨论 index ， GET 和 delete 请求时，我们指出每个文档都有一个 _version （版本）号，当文档被修改时版本号递增。 Elasticsearch 使用这个 _version 号来确保变更以正确顺序得到执行。如果旧版本的文档在新版本之后到达，它可以被简单的忽略。

我们可以利用 _version 号来确保 应用中相互冲突的变更不会导致数据丢失。我们通过指定想要修改文档的 version 号来达到这个目的。 如果该版本不是当前版本号，我们的请求将会失败。

当我们尝试通过重建文档的索引来保存修改，我们指定 version 为我们的修改会被应用的版本：
```html
<!--我们想这个在我们索引中的文档只有现在的 _version 为 1 时，本次更新才能成功。-->
PUT /website/blog/1?version=1
{
  "title": "My first blog entry",
  "text":  "Starting to get the hang of this..."
}
```

此请求成功，并且响应体告诉我们 _version 已经递增到 2 ：

```json
{
  "_index":   "website",
  "_type":    "blog",
  "_id":      "1",
  "_version": 2
  "created":  false
}
```

然而，如果我们重新运行相同的索引请求，仍然指定 version=1 ， Elasticsearch 返回 409 Conflict HTTP 响应码，和一个如下所示的响应体：

```JSON
{
   "error": {
      "root_cause": [
         {
            "type": "version_conflict_engine_exception",
            "reason": "[blog][1]: version conflict, current [2], provided [1]",
            "index": "website",
            "shard": "3"
         }
      ],
      "type": "version_conflict_engine_exception",
      "reason": "[blog][1]: version conflict, current [2], provided [1]",
      "index": "website",
      "shard": "3"
   },
   "status": 409
}
```

#### 通过外部系统使用版本控制编辑
一个常见的设置是使用其它数据库作为主要的数据存储，使用 Elasticsearch 做数据检索， 这意味着主数据库的所有更改发生时都需要被复制到 Elasticsearch ，如果多个进程负责这一数据同步，你可能遇到类似于之前描述的并发问题。

如果你的主数据库已经有了版本号 — 或一个能作为版本号的字段值比如 timestamp — 那么你就可以在 Elasticsearch 中通过增加 version_type=external 到查询字符串的方式重用这些相同的版本号， 版本号必须是大于零的整数， 且小于 9.2E+18 — 一个 Java 中 long 类型的正值。

外部版本号的处理方式和我们之前讨论的内部版本号的处理方式有些不同， Elasticsearch 不是检查当前 _version 和请求中指定的版本号是否相同， 而是检查当前 _version 是否 小于 指定的版本号。 如果请求成功，外部的版本号作为文档的新 _version 进行存储。

外部版本号不仅在索引和删除请求是可以指定，而且在 创建 新文档时也可以指定。

例如，要创建一个新的具有外部版本号 5 的博客文章，我们可以按以下方法进行：

```html
PUT /website/blog/2?version=5&version_type=external
{
  "title": "My first external blog entry",
  "text":  "Starting to get the hang of this..."
}
```

在响应中，我们能看到当前的 _version 版本号是 5 ：

```JSON
{
  "_index":   "website",
  "_type":    "blog",
  "_id":      "2",
  "_version": 5,
  "created":  true
}
```

现在我们更新这个文档，指定一个新的 version 号是 10 ：

```html
PUT /website/blog/2?version=10&version_type=external
{
  "title": "My first external blog entry",
  "text":  "This is a piece of cake..."
}
```
请求成功并将当前 _version 设为 10 ：

```JSON
{
  "_index":   "website",
  "_type":    "blog",
  "_id":      "2",
  "_version": 10,
  "created":  false
}
```

### 文档的部分更新
我们也介绍过文档是不可变的：他们不能被修改，只能被替换。 update API 必须遵循同样的规则。 从外部来看，我们在一个文档的某个位置进行部分更新。然而在内部， update API 简单使用与之前描述相同的 检索-修改-重建索引 的处理过程。 区别在于这个过程发生在分片内部，这样就避免了多次请求的网络开销。通过减少检索和重建索引步骤之间的时间，我们也减少了其他进程的变更带来冲突的可能性。

update 请求最简单的一种形式是接收文档的一部分作为 doc 的参数， 它只是与现有的文档进行合并。对象被合并到一起，覆盖现有的字段，增加新的字段。

### 取回多个文档
Elasticsearch 的速度已经很快了，但甚至能更快。 将多个请求合并成一个，避免单独处理每个请求花费的网络延时和开销。 如果你需要从 Elasticsearch 检索很多文档，那么使用`multi-get`或者`mget` API 来将这些检索请求放在一个请求中，将比逐个文档请求更快地检索到全部文档。

mget API 要求有一个 docs 数组作为参数，每个 元素包含需要检索文档的元数据， 包括 _index 、 _type 和 _id 。如果你想检索一个或者多个特定的字段，那么你可以通过 _source 参数来指定这些字段的名字:
```html
GET /_mget
{
   "docs" : [
      {
         "_index" : "website",
         "_type" :  "blog",
         "_id" :    2
      },
      {
         "_index" : "website",
         "_type" :  "pageviews",
         "_id" :    1,
         "_source": "views"
      }
   ]
}
```

### 路由一个文档到一个分片中
当索引一个文档的时候，文档会被存储到一个主分片中。 Elasticsearch 如何知道一个文档应该存放到哪个分片中呢？当我们创建文档时，它如何决定这个文档应当被存储在分片 1 还是分片 2 中呢？

首先这肯定不会是随机的，否则将来要获取文档的时候我们就不知道从何处寻找了。实际上，这个过程是根据下面这个公式决定的：

shard = hash(routing) % number_of_primary_shards
routing 是一个可变值，默认是文档的 _id ，也可以设置成一个自定义的值。 routing 通过 hash 函数生成一个数字，然后这个数字再除以 number_of_primary_shards （主分片的数量）后得到 余数 。这个分布在 0 到 number_of_primary_shards-1 之间的余数，就是我们所寻求的文档所在分片的位置。

这就解释了为什么我们要在创建索引的时候就确定好主分片的数量 并且永远不会改变这个数量：因为如果数量变化了，那么所有之前路由的值都会无效，文档也再也找不到了。

所有的文档 API（ get 、 index 、 delete 、 bulk 、 update 以及 mget ）都接受一个叫做 routing 的路由参数 ，通过这个参数我们可以自定义文档到分片的映射。一个自定义的路由参数可以用来确保所有相关的文档——例如所有属于同一个用户的文档——都被存储到同一个分片中。我们也会在扩容设计这一章中详细讨论为什么会有这样一种需求。

### 新建、索引、和删除单个文档

在默认设置下，即使仅仅是在试图执行一个_写_操作之前，主分片都会要求 必须要有 _规定数量(quorum)_（或者换种说法，也即必须要有大多数）的分片副本处于活跃可用状态，才会去执行_写_操作(其中分片副本可以是主分片或者副本分片)。这是为了避免在发生网络分区故障（network partition）的时候进行_写_操作，进而导致数据不一致。_规定数量_即：
`
int( (primary + number_of_replicas) / 2 ) + 1
`
consistency 参数的值可以设为 one （只要主分片状态 ok 就允许执行_写_操作）,all`（必须要主分片和所有副本分片的状态没问题才允许执行_写_操作）, 或 `quorum 。默认值为 quorum , 即大多数的**分片副本**状态没问题就允许执行_写_操作。

注意，规定数量 的计算公式中 number_of_replicas 指的是在索引设置中的设定副本分片数，而不是指当前处理活动状态的副本分片数。如果你的索引设置中指定了当前索引拥有三个副本分片，那规定数量的计算结果即：
`
int( (primary + 3 replicas) / 2 ) + 1 = 3
`
如果此时你只启动两个节点，那么处于活跃状态的分片副本数量就达不到规定数量，也因此您将无法索引和删除任何文档。

> 新索引默认有 1 个副本分片，这意味着为满足 规定数量 应该 需要两个活动的分片副本。 但是，这些默认的设置会阻止我们在单一节点上做任何事情。为了避免这个问题，要求只有当 number_of_replicas 大于1的时候，规定数量才会执行。

### 同时查询多索引，多类型
如果不对某一特殊的索引或者类型做限制，就会搜索集群中的所有文档。Elasticsearch 转发搜索请求到每一个主分片或者副本分片，汇集查询出的前10个结果，并且返回给我们。

然而，经常的情况下，你想在一个或多个特殊的索引并且在一个或者多个特殊的类型中进行搜索。我们可以通过在URL中指定特殊的索引和类型达到这种效果，如下所示：
`
/_search
`
在所有的索引中搜索所有的类型

`
/gb/_search
`
在 gb 索引中搜索所有的类型

`
/gb,us/_search
`
在 gb 和 us 索引中搜索所有的文档
`
/g*,u*/_search
`
在任何以 g 或者 u 开头的索引中搜索所有的类型
`
/gb/user/_search
`
在 gb 索引中搜索 user 类型

`
/gb,us/user,tweet/_search
`
在 gb 和 us 索引中搜索 user 和 tweet 类型

`
/_all/user,tweet/_search
`
在所有的索引中搜索 user 和 tweet 类型

当在单一的索引下进行搜索的时候，Elasticsearch 转发请求到索引的每个分片中，可以是主分片也可以是副本分片，然后从每个分片中收集结果。多索引搜索恰好也是用相同的方式工作的--只是会涉及到更多的分片。

>搜索一个索引有五个主分片和搜索五个索引各有一个分片准确来所说是等价的

### 分页查询
和 SQL 使用 LIMIT 关键字返回单个 page 结果的方法相同，Elasticsearch 接受 from 和 size 参数：
`
size
`
显示应该返回的结果数量，默认是 10

`
from
`
显示应该跳过的初始结果数量，默认是 0

如果每页展示 5 条结果，可以用下面方式请求得到 1 到 3 页的结果：

### 搜索
有两种形式的搜索API
1. “轻量的”查询自渡船版本
2. 另一种是更完整的“请求体”版本，要求使用JSON格式和更丰富的查询表达式作为搜索语言

```html
<!-- 查询在tweet类型中tweet字段包含elasticsearch单词的所有文档 -->
GET /_all/tweet/_search?q=tweet:elasticsearch
```

```html
+name:john +tweet:mary
```
但是查询字符参数需要经过(URL编码)实际上更加难懂
```html
GET /_search?q=%2Bname%3Ajohn+%2Btweet%3Amary
```

> +前缀表示必须与查询条件匹配。类似地， - 前缀表示一定不与查询条件匹配。没有 + 或者 - 的所有其他条件都是可选的——匹配的越多，文档就越相关。

**_all字段**
这个简单搜索返回包含 mary 的所有文档：

```html
GET /_search?q=mary
```
之前的例子中，我们在 tweet 和 name 字段中搜索内容。然而，这个查询的结果在三个地方提到了 mary ：

- 有一个用户叫做 Mary
- 6条微博发自 Mary
- 一条微博直接 @mary

Elasticsearch 是如何在三个不同的字段中查找到结果的呢？

当索引一个文档的时候，Elasticsearch 取出所有字段的值拼接成一个大的字符串，作为 _all 字段进行索引。例如，当索引这个文档时：
```JSON
{
    "tweet":    "However did I manage before Elasticsearch?",
    "date":     "2014-09-14",
    "name":     "Mary Jones",
    "user_id":  1
}
```
这就好似增加了一个名叫 _all 的额外字段：
`
"However did I manage before Elasticsearch? 2014-09-14 Mary Jones 1"
`
除非设置特定字段，否则查询字符串就使用 _all 字段进行搜索。

#### 更复杂的查询
下面的查询针对tweents类型，并使用以下的条件：
- name 字段中包含 mary 或者 john
- date 值大于 2014-09-10
- _all 字段包含 aggregations 或者 geo

`
+name:(mary john) +date:>2014-09-10 +(aggregations geo)
`
查询字符串在做了适当的编码后，可读性很差：
`
?q=%2Bname%3A(mary+john)+%2Bdate%3A%3E2014-09-10+%2B(aggregations+geo)
`

最后，查询字符串搜索允许任何用户在索引的任意字段上执行可能较慢且重量级的查询，这可能会暴露隐私信息，甚至将集群拖垮。

因为这些原因，不推荐直接向用户暴露查询字符串搜索功能，除非对于集群和数据来说非常信任他们。

相反，我们经常在生产环境中更多地使用功能全面的 request body 查询API，除了能完成以上所有功能，还有一些附加功能。但在到达那个阶段之前，首先需要了解数据在 Elasticsearch 中是如何被索引的。

#### 精确值VS全文
Elasticsearch 中的数据可以概括的分为两类：精确值和全文。

精确值 如它们听起来那样精确。例如日期或者用户 ID，但字符串也可以表示精确值，例如用户名或邮箱地址。对于精确值来讲，Foo 和 foo 是不同的，2014 和 2014-09-15 也是不同的。

另一方面，全文 是指文本数据（通常以人类容易识别的语言书写），例如一个推文的内容或一封邮件的内容。

精确值很容易查询。结果是二进制的：要么匹配查询，要么不匹配。这种查询很容易用 SQL 表示

```SQL
WHERE name    = "John Smith"
  AND user_id = 2
  AND date    > "2014-09-15"
```
查询全文数据要微妙的多。我们问的不只是“这个文档匹配查询吗”，而是“该文档匹配查询的程度有多大？”换句话说，该文档与给定查询的相关性如何？

### 映射

索引中每个文档都有 类型 。每种类型都有它自己的 映射 ，或者 模式定义 。映射定义了类型中的域，每个域的数据类型，以及Elasticsearch如何处理这些域。映射也用于配置与类型有关的元数据。

#### 核心简单域类型

- 字符串: string
- 整数 : byte, short, integer, long
- 浮点数: float, double
- 布尔型: boolean
- 日期: date

当你索引一个包含新域的文档--之前未曾出现-- Elasticsearch 会使用 动态映射 ，通过JSON中基本数据类型，尝试猜测域类型。

>这意味着如果你通过引号( "123" )索引一个数字，它会被映射为 string 类型，而不是 long 。但是，如果这个域已经映射为 long ，那么 Elasticsearch 会尝试将这个字符串转化为 long ，如果无法转化，则抛出一个异常。

#### 查看映射
通过 /_mapping ，我们可以查看 Elasticsearch 在一个或多个索引中的一个或多个类型的映射 。在 开始章节 ，我们已经取得索引 gb 中类型 tweet 的映射：
```html
GET /gb/_mapping/tweet
```

#### 自定义映射


默认， string 类型域会被认为包含全文。就是说，它们的值在索引前，会通过 一个分析器，针对于这个域的查询在搜索前也会经过一个分析器。

string 域映射的两个最重要 属性是 index 和 analyzer 。

- index
  index 属性控制怎样索引字符串。它可以是下面三个值：

  `analyzed`
  首先分析字符串，然后索引它。换句话说，以全文索引这个域。

  `not_analyzed`
    索引这个域，所以它能够被搜索，但索引的是精确值。不会对它进行分析。

  `no`
  不索引这个域。这个域不会被搜索到。

- analyzer
对于 analyzed 字符串域，用 analyzer 属性指定在搜索和索引时使用的分析器。默认， Elasticsearch 使用 standard 分析器， 但你可以指定一个内置的分析器替代它，例如 whitespace 、 simple 和 `english`：

```html
PUT /gb
{
  "mappings": {
    "tweet" : {
      "properties" : {
        "tweet" : {
          "type" :    "string",
          "analyzer": "english"
        },
        "date" : {
          "type" :   "date"
        },
        "name" : {
          "type" :   "string"
        },
        "user_id" : {
          "type" :   "long"
        }
      }
    }
  }
}
```

### 查询表达式
#### 一个带请求体的 GET 请求？
某些特定语言（特别是 JavaScript）的 HTTP 库是不允许 GET 请求带有请求体的。 事实上，一些使用者对于 GET 请求可以带请求体感到非常的吃惊。

而事实是这个RFC文档 RFC 7231— 一个专门负责处理 HTTP 语义和内容的文档 — 并没有规定一个带有请求体的 GET 请求应该如何处理！结果是，一些 HTTP 服务器允许这样子，而有一些 — 特别是一些用于缓存和代理的服务器 — 则不允许。

对于一个查询请求，Elasticsearch 的工程师偏向于使用 GET 方式，因为他们觉得它比 POST 能更好的描述信息检索（retrieving information）的行为。然而，因为带请求体的 GET 请求并不被广泛支持，所以 search API 同时支持 POST 请求：
```html
POST /_search
{
  "from": 30,
  "size": 10
}
```
类似的规则可以应用于任何需要带请求体的 GET API。

- 查询语句的结构
一个查询语句 的典型结构：
```JSON
{
    QUERY_NAME: {
        ARGUMENT: VALUE,
        ARGUMENT: VALUE,...
    }
}
```

如果是针对某个字段，那么它的结构如下：
```JSON
{
    QUERY_NAME: {
        FIELD_NAME: {
            ARGUMENT: VALUE,
            ARGUMENT: VALUE,...
        }
    }
}
```
举个例子，你可以使用 match 查询语句 来查询 tweet 字段中包含 elasticsearch 的 tweet：
```JSON
{
  "match": {
    "tweet":"elasticsearch"
  }
}
```
完整的查询请求如下：
```html
GET /_search
{
    "query": {
        "match": {
            "tweet": "elasticsearch"
        }
    }
}
```

#### match_all查询
match_all 查询简单的 匹配所有文档。在没有指定查询方式时，它是默认的查询：

#### match查询
无论你在任何字段上进行的是全文搜索还是精确查询，match 查询是你可用的标准查询。

如果你在一个全文字段上使用 match 查询，在执行查询前，它将用正确的分析器去分析查询字符串：
```JSON
{ "match": { "tweet": "About Search" }}
```
如果在一个精确值的字段上使用它， 例如数字、日期、布尔或者一个 not_analyzed 字符串字段，那么它将会精确匹配给定的值：
```JSON
{ "match": { "age":    26           }}
{ "match": { "date":   "2014-09-01" }}
{ "match": { "public": true         }}
{ "match": { "tag":    "full_text"  }}
```

>对于精确值的查询，你可能需要使用 filter 语句来取代 query，因为 filter 将会被缓存。接下来，我们将看到一些关于 filter 的例子。

#### multi_match 查询
multi_match 查询可以在多个字段上执行相同的 match 查询:
```JSON
{
    "multi_match": {
        "query":    "full text search",
        "fields":   [ "title", "body" ]
    }
}
```

如
```html
GET /_search
{
	"query":{
		"multi_match": {
			"query": "forestry",
			"fields":["about", "interests"]
		}
	}
}
```

#### range查询
```JSON
{
  "query": {
    "range": {
      "age": {
        "gte":20,
        "lt": 30
      }
    }
  }
}
```
被允许的操作符如下：

gt
大于

gte
大于等于

lt
小于

lte
小于等于

#### term查询
term 查询被用于精确值 匹配，这些精确值可能是数字、时间、布尔或者那些 not_analyzed 的字符串
```JSON
{ "term": { "age":    26           }}
{ "term": { "date":   "2014-09-01" }}
{ "term": { "public": true         }}
{ "term": { "tag":    "full_text"  }}
```
term 查询对于输入的文本不分析 ，所以它将给定的值进行精确查询。

#### terms查询
terms 查询和 term 查询一样，但它允许你指定多值进行匹配。如果这个字段包含了指定值中的任何一个值，那么这个文档满足条件：
```JSON
{ "terms": { "tag": [ "search", "full_text", "nosql" ] }}
```
和 term 查询一样，terms 查询对于输入的文本不分析。它查询那些精确匹配的值（包括在大小写、重音、空格等方面的差异）。

#### exist查询和missing查询
exists 查询和 missing 查询被用于查找那些指定字段中有值 (exists) 或无值 (missing) 的文档。这与SQL中的 IS_NULL (missing) 和 NOT IS_NULL (exists) 在本质上具有共性：
```JSON
{
    "exists":   {
        "field":    "title"
    }
}
```
这些查询经常用于某个字段有值的情况和某个字段缺值的情况。

### 组合多查询
你可以用 bool 查询来实现你的需求。这种查询将多查询组合在一起，成为用户自己想要的布尔查询。它接收以下参数：

- must
  文档 必须 匹配这些条件才能被包含进来。
- must_not
  文档 必须不 匹配这些条件才能被包含进来。
- should
  如果满足这些语句中的任意语句，将增加 _score ，否则，无任何影响。它们主要用于修正每个文档的相关性得分。
- filter
  必须 匹配，但它以不评分、过滤模式来进行。这些语句对评分没有贡献，只是根据过滤标准来排除或包含文档。

  由于这是我们看到的第一个包含多个查询的查询，所以有必要讨论一下相关性得分是如何组合的。每一个子查询都独自地计算文档的相关性得分。一旦他们的得分被计算出来， bool 查询就将这些得分进行合并并且返回一个代表整个布尔操作的得分。

  下面的查询用于查找 title 字段匹配 how to make millions 并且不被标识为 spam 的文档。那些被标识为 starred 或在2014之后的文档，将比另外那些文档拥有更高的排名。如果 _两者_ 都满足，那么它排名将更高：

  ```json
  {
      "bool": {
          "must":     { "match": { "title": "how to make millions" }},
          "must_not": { "match": { "tag":   "spam" }},
          "should": [
              { "match": { "tag": "starred" }},
              { "range": { "date": { "gte": "2014-01-01" }}}
          ]
      }
  }
  ```
  >如果没有 must 语句，那么至少需要能够匹配其中的一条 should 语句。但，如果存在至少一条 must 语句，则对 should 语句的匹配没有要求。

  #### 增加带过滤器（filtering）的查询
如果我们不想因为文档的时间而影响得分，可以用 filter 语句来重写前面的例子
```JSON
{
    "bool": {
        "must":     { "match": { "title": "how to make millions" }},
        "must_not": { "match": { "tag":   "spam" }},
        "should": [
            { "match": { "tag": "starred" }}
        ],
        "filter": {
          "range": { "date": { "gte": "2014-01-01" }}
        }
    }
}
/* range 查询已经从 should 语句中移到 filter 语句 */
```
通过将 range 查询移到 filter 语句中，我们将它转成不评分的查询，将不再影响文档的相关性排名。由于它现在是一个不评分的查询，可以使用各种对 filter 查询有效的优化手段来提升性能。

所有查询都可以借鉴这种方式。将查询移到 bool 查询的 filter 语句中，这样它就自动的转成一个不评分的 filter 了。

如果你需要通过多个不同的标准来过滤你的文档，bool 查询本身也可以被用做不评分的查询。简单地将它放置到 filter 语句中并在内部构建布尔逻辑：
```JSON
{
    "bool": {
        "must":     { "match": { "title": "how to make millions" }},
        "must_not": { "match": { "tag":   "spam" }},
        "should": [
            { "match": { "tag": "starred" }}
        ],
        "filter": {
          "bool": {
              "must": [
                  { "range": { "date": { "gte": "2014-01-01" }}},
                  { "range": { "price": { "lte": 29.99 }}}
              ],
              "must_not": [
                  { "term": { "category": "ebooks" }}
              ]
          }
        }
    }
}
```
通过混合布尔查询，我们可以在我们的查询请求中灵活地编写 scoring 和 filtering 查询逻辑

#### constant_score 查询
尽管没有 bool 查询使用这么频繁，constant_score 查询也是你工具箱里有用的查询工具。它将一个不变的常量评分应用于所有匹配的文档。它被经常用于你只需要执行一个 filter 而没有其它查询（例如，评分查询）的情况下。

可以使用它来取代只有 filter 语句的 bool 查询。在性能上是完全相同的，但对于提高查询简洁性和清晰度有很大帮助。
```JSON
{
    "constant_score":   {
        "filter": {
            "term": { "category": "ebooks" }
        }
    }
}
/* term 查询被放置在 constant_score 中，转成不评分的 filter。这种方式可以用来取代只有 filter 语句的 bool 查询。 */
```

### 排序
为了按照相关性来排序，需要将相关性表示为一个数值。在 Elasticsearch 中， 相关性得分 由一个浮点数进行表示，并在搜索结果中通过 _score 参数返回， 默认排序是 _score 降序。

#### 按照字段的值排序
在这个案例中，通过时间来对 tweets 进行排序是有意义的，最新的 tweets 排在最前。 我们可以使用 sort 参数进行实现
```html
GET /_search
{
    "query" : {
        "bool" : {
            "filter" : { "term" : { "user_id" : 1 }}
        }
    },
    "sort": { "date": { "order": "desc" }}
}
```

首先我们在每个结果中有一个新的名为 sort 的元素，它包含了我们用于排序的值。 在这个案例中，我们按照 date 进行排序，在内部被索引为 自 epoch 以来的毫秒数 。 long 类型数 1411516800000 等价于日期字符串 2014-09-24 00:00:00 UTC 。

其次 _score 和 max_score 字段都是 null 。 计算 _score 的花销巨大，通常仅用于排序； 我们并不根据相关性排序，所以记录 _score 是没有意义的。如果无论如何你都要计算 _score ， 你可以将 track_scores 参数设置为 true 。

#### 多级排序
假定我们想要结合使用 date 和 _score 进行查询，并且匹配的结果首先按照日期排序，然后按照相关性排序：
```JSON
GET /_search
{
    "query" : {
        "bool" : {
            "must":   { "match": { "tweet": "manage text search" }},
            "filter" : { "term" : { "user_id" : 2 }}
        }
    },
    "sort": [
        { "date":   { "order": "desc" }},
        { "_score": { "order": "desc" }}
    ]
}
```
排序条件的顺序是很重要的。结果首先按第一个条件排序，仅当结果集的第一个 sort 值完全相同时才会按照第二个条件进行排序，以此类推。

多级排序并不一定包含 _score 。你可以根据一些不同的字段进行排序， 如地理距离或是脚本计算的特定值。

>Query-string 搜索 也支持自定义排序，可以在查询字符串中使用 sort 参数：GET /_search?sort=date:desc&sort=_score&q=search

#### 多值字段排序
一种情形是字段有多个值的排序， 需要记住这些值并没有固有的顺序；一个多值的字段仅仅是多个值的包装，这时应该选择哪个进行排序呢？

对于数字或日期，你可以将多值字段减为单值，这可以通过使用 min 、 max 、 avg 或是 sum 排序模式 。 例如你可以按照每个 date 字段中的最早日期进行排序，通过以下方法：
```JSON
"sort": {
    "dates": {
        "order": "asc",
        "mode":  "min"
    }
}
```

### 搜索选项

#### preference
偏好这个参数 preference 允许 用来控制由哪些分片或节点来处理搜索请求。 它接受像 _primary, _primary_first, _local, _only_node:xyz, _prefer_node:xyz, 和 _shards:2,3 这样的值, 这些值在 https://www.elastic.co/guide/en/elasticsearch/reference/5.6/search-request-preference.html
详细说明

但是最有用的值是某些随机字符串，它可以避免 bouncing results 问题。

>Bouncing Results:想象一下有两个文档有同样值的时间戳字段，搜索结果用 timestamp 字段来排序。 由于搜索请求是在所有有效的分片副本间轮询的，那就有可能发生主分片处理请求时，这两个文档是一种顺序， 而副本分片处理请求时又是另一种顺序。这就是所谓的 bouncing results 问题: 每次用户刷新页面，搜索结果表现是不同的顺序。 让同一个用户始终使用同一个分片，这样可以避免这种问题， 可以设置 preference 参数为一个特定的任意值比如用户会话ID来解决。

#### 超时问题
通常分片处理完它所有的数据后再把结果返回给协同节点，协同节点把收到的所有结果合并为最终结果。

这意味着花费的时间是最慢分片的处理时间加结果合并的时间。如果有一个节点有问题，就会导致所有的响应缓慢。

参数 timeout 告诉 分片允许处理数据的最大时间。如果没有足够的时间处理所有数据，这个分片的结果可以是部分的，甚至是空数据。

搜索的返回结果会用属性 timed_out 标明分片是否返回的是部分结果：
```JSON
...
    "timed_out":     true,  
...
```

### 游标查询

### 索引管理
#### 创建一个索引
到目前为止, 我们已经通过索引一篇文档创建了一个新的索引 。这个索引采用的是默认的配置，新的字段通过动态映射的方式被添加到类型映射。现在我们需要对这个建立索引的过程做更多的控制：我们想要确保这个索引有数量适中的主分片，并且在我们索引任何数据 之前 ，分析器和映射已经被建立好。

为了达到这个目的，我们需要手动创建索引，在请求体里面传入设置或类型映射，如下所示：
```html
PUT /my_index
{
    "settings": { ... any settings ... },
    "mappings": {
        "type_one": { ... any mappings ... },
        "type_two": { ... any mappings ... },
        ...
    }
}
```

>我们会在之后讨论你怎么用 索引模板 来预配置开启自动创建索引。这在索引日志数据的时候尤其有用：你将日志数据索引在一个以日期结尾命名的索引上，子夜时分，一个预配置的新索引将会自动进行创建。

#### 删除一个索引
用以下的请求来 删除索引:
```html
DELETE /my_index
```
你也可以这样删除多个索引：
```html
DELETE /index_one,index_two
DELETE /index_*
```
你甚至可以这样删除 全部 索引：
```html
DELETE /_all
DELETE /*
```
>对一些人来说，能够用单个命令来删除所有数据可能会导致可怕的后果。如果你想要避免意外的大量删除, 你可以在你的 elasticsearch.yml 做如下配置：
action.destructive_requires_name: true
这个设置使删除只限于特定名称指向的数据, 而不允许通过指定 _all 或通配符来删除指定索引库。

#### 索引设置
你可以通过修改配置来自定义索引行为，详细配置参照https://www.elastic.co/guide/en/elasticsearch/reference/5.6/index-modules.html

>Elasticsearch 提供了优化好的默认配置。 除非你理解这些配置的作用并且知道为什么要去修改，否则不要随意修改。
下面是两个 最重要的设置：

- number_of_shards
  每个索引的主分片数，默认值是 5 。这个配置在索引创建后不能修改。
- number_of_replicas
  每个主分片的副本数，默认值是 1 。对于活动的索引库，这个配置可以随时修改。

例如，我们可以创建只有 一个主分片，没有副本的小索引：
```html
PUT /my_temp_index
{
    "settings": {
        "number_of_shards" :   1,
        "number_of_replicas" : 0
    }
}
```
然后，我们可以用 update-index-settings API 动态修改副本数：
```html
PUT /my_temp_index/_settings
{
    "number_of_replicas": 1
}
```
#### 配置分析器
第三个重要的索引设置是 analysis 部分， 用来配置已存在的分析器或针对你的索引创建新的自定义分析器。

在下面的例子中，我们创建了一个新的分析器，叫做**es_std** ， 并使用预定义的 西班牙语停用词列表：

```html
PUT /spanish_docs
{
    "settings": {
        "analysis": {
            "analyzer": {
                "es_std": {
                    "type":      "standard",
                    "stopwords": "_spanish_"
                }
            }
        }
    }
}
```
es_std 分析器不是全局的--它仅仅存在于我们定义的 spanish_docs 索引中。 为了使用 analyze API来对它进行测试，我们必须使用特定的索引名：
```html
GET /spanish_docs/_analyze?analyzer=es_std
El veloz zorro marrón
```
简化的结果显示西班牙语停用词 El 已被正确的移除
```JSON
{
  "tokens" : [
    { "token" :    "veloz",   "position" : 2 },
    { "token" :    "zorro",   "position" : 3 },
    { "token" :    "marrón",  "position" : 4 }
  ]
}
```

#### 自定义分析器
虽然Elasticsearch带有一些现成的分析器，然而在分析器上Elasticsearch真正的强大之处在于，你可以通过在一个适合你的特定数据的设置之中组合字符过滤器、分词器、词汇单元过滤器来创建自定义的分析器。

在 分析与分析器 我们说过，一个 分析器 就是在一个包里面组合了三种函数的一个包装器， 三种函数按照顺序被执行:
- 字符过滤器
  字符过滤器 用来 整理 一个尚未被分词的字符串。例如，如果我们的文本是HTML格式的，它会包含像 `<p>` 或者 `<div>` 这样的HTML标签，这些标签是我们不想索引的。我们可以使用 html清除 字符过滤器 来移除掉所有的HTML标签，并且像把 `&Aacute;` 转换为相对应的Unicode字符 Á 这样，转换HTML实体。

  一个分析器可能有0个或者多个字符过滤器。
- 分词器
  一个分析器 必须 有一个唯一的分词器。 分词器把字符串分解成单个词条或者词汇单元。 标准 分析器里使用的 标准 分词器 把一个字符串根据单词边界分解成单个词条，并且移除掉大部分的标点符号，然而还有其他不同行为的分词器存在。

  例如， 关键词 分词器 完整地输出 接收到的同样的字符串，并不做任何分词。 空格 分词器 只根据空格分割文本 。 正则 分词器 根据匹配正则表达式来分割文本 。

- 词单元过滤器
  经过分词，作为结果的 词单元流 会按照指定的顺序通过指定的词单元过滤器 。

  词单元过滤器可以修改、添加或者移除词单元。我们已经提到过 lowercase 和 stop 词过滤器 ，但是在 Elasticsearch 里面还有很多可供选择的词单元过滤器。 词干过滤器 把单词 遏制 为 词干。 ascii_folding 过滤器移除变音符，把一个像 "très" 这样的词转换为 "tres" 。 ngram 和 edge_ngram 词单元过滤器 可以产生 适合用于部分匹配或者自动补全的词单元
##### 创建一个自定义分析器
和我们之前配置 es_std 分析器一样，我们可以在 analysis 下的相应位置设置字符过滤器、分词器和词单元过滤器:
```html
PUT /my_index
{
    "settings": {
        "analysis": {
            "char_filter": { ... custom character filters ... },
            "tokenizer":   { ...    custom tokenizers     ... },
            "filter":      { ...   custom token filters   ... },
            "analyzer":    { ...    custom analyzers      ... }
        }
    }
}
```
作为示范，让我们一起来创建一个自定义分析器吧，这个分析器可以做到下面的这些事:  


1. 使用 html清除 字符过滤器移除HTML部分。
2. 使用一个自定义的 映射 字符过滤器把 & 替换为 " and " ：
```JSON
"char_filter": {
    "&_to_and": {
        "type":       "mapping",
        "mappings": [ "&=> and "]
    }
}
```
3. 使用 标准 分词器分词。
4. 小写词条，使用 小写 词过滤器处理。
5. 使用自定义 停止 词过滤器移除自定义的停止词列表中包含的词：
```JSON
"filter": {
    "my_stopwords": {
        "type":        "stop",
        "stopwords": [ "the", "a" ]
    }
}
```
我们的分析器定义用我们之前已经设置好的自定义过滤器组合了已经定义好的分词器和过滤器：
```JSON
"analyzer": {
    "my_analyzer": {
        "type":           "custom",
        "char_filter":  [ "html_strip", "&_to_and" ],
        "tokenizer":      "standard",
        "filter":       [ "lowercase", "my_stopwords" ]
    }
}
```
汇总起来，完整的 创建索引 请求 看起来应该像这样：
```JSON
PUT /my_index
{
    "settings": {
        "analysis": {
            "char_filter": {
                "&_to_and": {
                    "type":       "mapping",
                    "mappings": [ "&=> and "]
            }},
            "filter": {
                "my_stopwords": {
                    "type":       "stop",
                    "stopwords": [ "the", "a" ]
            }},
            "analyzer": {
                "my_analyzer": {
                    "type":         "custom",
                    "char_filter":  [ "html_strip", "&_to_and" ],
                    "tokenizer":    "standard",
                    "filter":       [ "lowercase", "my_stopwords" ]
            }}
}}}
```

这个分析器现在是没有多大用处的，除非我们告诉 Elasticsearch在哪里用上它。我们可以像下面这样把这个分析器应用在一个 string 字段上：
```html
PUT /my_index/_mapping/my_type
{
    "properties": {
        "title": {
            "type":      "string",
            "analyzer":  "my_analyzer"
        }
    }
}
```

#### 动态映射
当 Elasticsearch 遇到文档中以前 未遇到的字段，它用 dynamic mapping 来确定字段的数据类型并自动把新的字段添加到类型映射。

有时这是想要的行为有时又不希望这样。通常没有人知道以后会有什么新字段加到文档，但是又希望这些字段被自动的索引。也许你只想忽略它们。如果Elasticsearch是作为重要的数据存储，可能就会期望遇到新字段就会抛出异常，这样能及时发现问题。

幸运的是可以用 dynamic 配置来控制这种行为 ，可接受的选项如下：
- true
  动态添加新的字段--缺省
- false
  忽略新的字段
- strict
  如果遇到新字段抛出异常

配置参数 dynamic 可以用在根 object 或任何 object 类型的字段上。你可以将 dynamic 的默认值设置为 strict , 而只在指定的内部对象中开启它, 例如：
```html
PUT /my_index
{
    "mappings": {
        "my_type": {
            "dynamic":      "strict",
            "properties": {
                "title":  { "type": "string"},
                "stash":  {
                    "type":     "object",
                    "dynamic":  true
                }
            }
        }
    }
}
<!-- 如果遇到新字段，对象 my_type 就会抛出异常。而内部对象 stash 遇到新字段就会动态创建新字段。 -->
```

使用上述动态映射， 你可以给 stash 对象添加新的可检索的字段：
```html
PUT /my_index/my_type/1
{
    "title":   "This doc adds a new field",
    "stash": { "new_field": "Success!" }
}
```

但是对根节点对象 my_type 进行同样的操作会失败：
```html
PUT /my_index/my_type/1
{
    "title":     "This throws a StrictDynamicMappingException",
    "new_field": "Fail!"
}
```

>把 dynamic 设置为 false 一点儿也不会改变 _source 的字段内容。 _source 仍然包含被索引的整个JSON文档。只是新的字段不会被加到映射中也不可搜索。

#### 自定义动态映射
如果你想在运行时增加新的字段，你可能会启用动态映射。 然而，有时候，动态映射 规则 可能不太智能。幸运的是，我们可以通过设置去自定义这些规则，以便更好的适用于你的数据。
##### 日期检测
当 Elasticsearch 遇到一个新的字符串字段时，它会检测这个字段是否包含一个可识别的日期，比如 2014-01-01 。 如果它像日期，这个字段就会被作为 date 类型添加。否则，它会被作为 string 类型添加。

有些时候这个行为可能导致一些问题。想象下，你有如下这样的一个文档：
```JSON
{ "note": "2014-01-01" }
```
假设这是第一次识别 note 字段，它会被添加为 date 字段。但是如果下一个文档像这样：
```JSON
{ "note": "Logged out" }
```
这显然不是一个日期，但为时已晚。这个字段已经是一个日期类型，这个 不合法的日期 将会造成一个异常。

日期检测可以通过在根对象上设置 date_detection 为 false 来关闭：
```html
PUT /my_index
{
    "mappings": {
        "my_type": {
            "date_detection": false
        }
    }
}
```
使用这个映射，字符串将始终作为 string 类型。如果你需要一个 date 字段，你必须手动添加。

#### 动态模板

### 分片内部原理
#### 近实时搜索
在 Elasticsearch 中，写入和打开一个新段的轻量的过程叫做 refresh 。 默认情况下每个分片会每秒自动刷新一次。这就是为什么我们说 Elasticsearch 是 近 实时搜索: 文档的变化并不是立即对搜索可见，但会在一秒之内变为可见。

这些行为可能会对新用户造成困惑: 他们索引了一个文档然后尝试搜索它，但却没有搜到。这个问题的解决办法是用 refresh API 执行一次手动刷新:
```html
<!--	刷新（Refresh）所有的索引  -->
POST /_refresh
<!-- 	只刷新（Refresh） blogs 索引。 -->
POST /blogs/_refresh
```

>尽管刷新是比提交轻量很多的操作，它还是会有性能开销。 当写测试的时候， 手动刷新很有用，但是不要在生产环境下每次索引一个文档都去手动刷新。 相反，你的应用需要意识到 Elasticsearch 的近实时的性质，并接受它的不足。

并不是所有的情况都需要每秒刷新。可能你正在使用 Elasticsearch 索引大量的日志文件， 你可能想优化索引速度而不是近实时搜索， 可以通过设置 refresh_interval ， 降低每个索引的刷新频率：
``html
<!-- 	每30秒刷新 my_logs 索引 -->
PUT /my_logs
{
  "settings": {
    "refresh_interval": "30s"
  }
}
```

refresh_interval 可以在既存索引上进行动态更新。 在生产环境中，当你正在建立一个大的新索引时，可以先关闭自动刷新，待开始使用该索引时，再把它们调回来：
```html
<!-- 关闭自动刷新。 -->
PUT /my_logs/_settings
{ "refresh_interval": -1 }

<!-- 每秒自动刷新。 -->
PUT /my_logs/_settings
{ "refresh_interval": "1s" }
```

**refresh_interval 需要一个 持续时间 值， 例如 1s （1 秒） 或 2m （2 分钟）。 一个绝对值 1 表示的是 1毫秒 --无疑会使你的集群陷入瘫痪。**

#### 持久化变更
如果没有用 fsync 把数据从文件系统缓存刷（flush）到硬盘，我们不能保证数据在断电甚至是程序正常退出之后依然存在。为了保证 Elasticsearch 的可靠性，需要确保数据变化被持久化到磁盘。

在 动态更新索引，我们说一次完整的提交会将段刷到磁盘，并写入一个包含所有段列表的提交点。Elasticsearch 在启动或重新打开一个索引的过程中使用这个提交点来判断哪些段隶属于当前分片。

即使通过每秒刷新（refresh）实现了近实时搜索，我们仍然需要经常进行完整提交来确保能从失败中恢复。但在两次提交之间发生变化的文档怎么办？我们也不希望丢失掉这些数据。

Elasticsearch 增加了一个 translog ，或者叫事务日志，在每一次对 Elasticsearch 进行操作时均进行了日志记录。通过 translog
1. 一个文档被索引之后，就会被添加到内存缓冲区，并且 追加到了 translog
2. 刷新（refresh）分片每秒被刷新（refresh）一次：
  - 这些在内存缓冲区的文档被写入到一个新的段中，且没有进行 fsync 操作。
  - 这个段被打开，使其可被搜索。
  - 内存缓冲区被清空。
3. 这个进程继续工作，更多的文档被添加到内存缓冲区和追加到事务日志
4. 每隔一段时间--例如 translog 变得越来越大--索引被刷新（flush）；一个新的 translog 被创建，并且一个全量提交被执行
  - 所有在内存缓冲区的文档都被写入一个新的段。
  - 缓冲区被清空。
  - 一个提交点被写入硬盘。
  - 文件系统缓存通过 fsync 被刷新（flush）。
  - 老的 translog 被删除。

在刷新（flush）之后，段被全量提交，并且事务日志被清空

translog 提供所有还没有被刷到磁盘的操作的一个持久化纪录。当 Elasticsearch 启动的时候， 它会从磁盘中使用最后一个提交点去恢复已知的段，并且会重放 translog 中所有在最后一次提交后发生的变更操作。

translog 也被用来提供实时 CRUD 。当你试着通过ID查询、更新、删除一个文档，它会在尝试从相应的段中检索之前， 首先检查 translog 任何最近的变更。这意味着它总是能够实时地获取到文档的最新版本。
##### flush API
这个执行一个提交并且截断 translog 的行为在 Elasticsearch 被称作一次 flush 。 分片每30分钟被自动刷新（flush），或者在 translog 太大的时候也会刷新。请查看 translog 文档 来设置，它可以用来 控制这些阈值：
flush API 可以 被用来执行一个手工的刷新（flush）:
```html
<!--刷新（flush） blogs 索引。  -->
POST /blogs/_flush
<!-- 	刷新（flush）所有的索引并且并且等待所有刷新在返回前完成。 -->
POST /_flush?wait_for_ongoing
```
>这就是说，在重启节点或关闭索引之前执行 flush 有益于你的索引。当 Elasticsearch 尝试恢复或重新打开一个索引， 它需要重放 translog 中所有的操作，所以如果日志越短，恢复越快。

#### 段合并
由于自动刷新流程每秒会创建一个新的段 ，这样会导致短时间内的段数量暴增。而段数目太多会带来较大的麻烦。 每一个段都会消耗文件句柄、内存和cpu运行周期。更重要的是，每个搜索请求都必须轮流检查每个段；所以段越多，搜索也就越慢。

Elasticsearch通过在后台进行段合并来解决这个问题。小的段被合并到大的段，然后这些大的段再被合并到更大的段。

##### optimize API
optimize API大可看做是 强制合并 API 。它会将一个分片强制合并到 max_num_segments 参数指定大小的段数目。 这样做的意图是减少段的数量（通常减少到一个），来提升搜索性能。
>optimize API 不应该 被用在一个动态索引————一个正在被活跃更新的索引。后台合并流程已经可以很好地完成工作。 optimizing 会阻碍这个进程。不要干扰它！

在特定情况下，使用 optimize API 颇有益处。例如在日志这种用例下，每天、每周、每月的日志被存储在一个索引中。 老的索引实质上是只读的；它们也并不太可能会发生变化。

在这种情况下，使用optimize优化老的索引，将每一个分片合并为一个单独的段就很有用了；这样既可以节省资源，也可以使搜索更加快速：
```html
<!-- 合并索引中的每个分片为一个单独的段 -->
POST /logstash-2014-10/_optimize?max_num_segments=1
```

**
请注意，使用 optimize API 触发段合并的操作一点也不会受到任何资源上的限制。这可能会消耗掉你节点上全部的I/O资源, 使其没有余裕来处理搜索请求，从而有可能使集群失去响应。 如果你想要对索引执行 `optimize`，你需要先使用分片分配（查看 迁移旧索引）把索引移到一个安全的节点，再执行。**


## 聚合
###高阶概念
需要掌握的概念
- 桶（Buckets）
  满足特定条件的文档的集合
- 指标（Metrics）
  对桶内的文档进行统计计算

  这一张由于中文版是基于Elasticsearch 2.x写的，语法在6.4.1上运行报错。因此先跳过，待阅读英文原版。

  ## 扩容设计
  ### 扩容的单元
  在 动态更新索引，我们介绍了一个分片即一个 Lucene 索引 ，一个 Elasticsearch 索引即一系列分片的集合。 你的应用程序与索引进行交互，Elasticsearch 帮助你将请求路由至相应的分片。

  一个分片即为 扩容的单元 。 一个最小的索引拥有一个分片。 这可能已经完全满足你的需求了 — 单个分片即可存储大量的数据 — 但这限制了你的可扩展性。
