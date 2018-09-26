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
term 查询对于输入的文本不 分析 ，所以它将给定的值进行精确查询。

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
