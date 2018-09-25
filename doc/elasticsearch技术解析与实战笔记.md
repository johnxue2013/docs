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
