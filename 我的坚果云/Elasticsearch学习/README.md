## 简介

全文搜索属于最常见的需求，开源的[Elasticsearch](https://www.elastic.co/) 是目前全文搜索引擎的首选。

它可以快速的存储、搜索和分析海量数据。维基百科、Stack Overflow、Github都采用它。

Elastic的底层是开源库 [Lucene](https://lucene.apache.org/)。但你没法直接使用Lucene，你必须自己写代码去调用它的接口。

Elastic是对Lucene的封装，提供了RESET API接口，开箱即用。

## 安装

Elastic 需要 Java 8 环境。

安装完 Java，就可以跟着[官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/current/zip-targz.html)安装 Elastic。直接下载压缩包比较简单。

详见：http://www.ruanyifeng.com/blog/2017/08/elasticsearch.html

## 基本概念

### Node与Cluster

Elastic本质上是一个分布式数据库，允许多台服务器协同工作，每台服务器可以运行多个Elastic实例。

单个Elastic实例称为一个节点，一组节点称为一个集群。

### Index

Elastic会索引所有字段，经过处理后写入一个反向索引（Inverted Index）。查找该数据的时候，直接查找该索引。

所以，Elastic数据管理的顶层单位就是Index（索引）。它单个数据库的同义词。

每个Index（即数据库）的名字必须是小写。

### Document

Index里的单条记录叫做Document，许多Document构成了Index。

Document使用JSON表示：

```json
{
  "user": "张三",
  "title": "工程师",
  "desc": "数据库管理"
}
```

同一个Index里的Document，不要求有相同的结构，但最好有相同的结构，这样有利于提高搜索效率。

### Type

Index可以分组，比如 *weather* 这个Index里面，可以按城市分组（北京和上海），也可以按照气候分组。

这种分组就叫做 *Type*。

它是虚拟的逻辑分组，用来过滤Document。

不同的Type应该有相同的结构。

## 新建和删除Index

新建 Index，可以直接向 Elastic 服务器发出 PUT 请求。

```bash
curl -x PUT 'localhost:9200/weather'
curl -x DELETE 'localhost:9200/weather'
```

## 中文分词设置

首选，需要安装中文分词插件，这里使用的是 [ik](https://github.com/medcl/elasticsearch-analysis-ik/)，也可以考虑其他插件（比如 [smartcn](https://www.elastic.co/guide/en/elasticsearch/plugins/current/analysis-smartcn.html)）。

```bash
$ ./bin/elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v5.5.1/elasticsearch-analysis-ik-5.5.1.zip
```

上面代码安装的是5.5.1版的插件，与 Elastic 5.5.1 配合使用。

接着，重新启动 Elastic，就会自动加载这个新安装的插件。

然后，新建一个Index，指定需要分词的字段。这一步根据数据结构而已，下面的命令只针对文本。基本上，凡事要搜索的字段，都需要重新设置一下：

```bash
$ curl -X PUT 'localhost:9200/accounts' -d '
{
"mappings": {
    "person": {
      "properties": {
        "user": {
          "type": "text",
          "analyzer": "ik_max_word",
          "search_analyzer": "ik_max_word"
        },
        "title": {
          "type": "text",
          "analyzer": "ik_max_word",
          "search_analyzer": "ik_max_word"
        },
        "desc": {
          "type": "text",
          "analyzer": "ik_max_word",
          "search_analyzer": "ik_max_word"
        }
      }
    }
  }
}
```

上面的代码中，首先新建一个名称为account的Type，里面有一个名称为person的Type。Person有三个字段：

- user
- title
- desc

这三个字段都是中文，而且类型都是文本（text），所以需要指定中文分词器，不能使用默认的英文分词器。

Elastic的分词器称为 [analyzer](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis.html)。我们对每个字段指定分词器。

```bash
"user": {
  "type": "text",
  "analyzer": "ik_max_word",
  "search_analyzer": "ik_max_word"
}
```

上面代码中：

- analyzer是字段文本的分词器；
- search_analyzer是搜索词的分词器；
- ik_max_word是ik提供的，可以对文本进行最大数量的分词。

## 数据操作

新增记录

查看记录

删除记录

更新记录

## 数据查询

### 返回所有记录

使用get方式，直接请求 /Index/Type/_search，就会返回所有记录：

```bash
$ curl 'localhost:9200/accounts/person/_search'
{
  "took":2,
  "timed_out":false,
  "_shards":{"total":5,"successful":5,"failed":0},
  "hits":{
    "total":2,
    "max_score":1.0,
    "hits":[
      {
        "_index":"accounts",
        "_type":"person",
        "_id":"AV3qGfrC6jMbsbXb6k1p",
        "_score":1.0,
        "_source": {
          "user": "李四",
          "title": "工程师",
          "desc": "系统管理"
        }
      },
      {
        "_index":"accounts",
        "_type":"person",
        "_id":"1",
        "_score":1.0,
        "_source": {
          "user" : "张三",
          "title" : "工程师",
          "desc" : "数据库管理，软件开发"
        }
      }
    ]
  }
}
```

### 全文搜索

Elastic的查询非常特别，使用自己的查询语法，要求get请求带有数据体：

```bash
curl 'localhost:9200/accounts/person/_search' -d '
   {
      "query": {"match" : {"desc" : "软件"}}
   }
'
```

上面代码是使用 [Match 查询](https://www.elastic.co/guide/en/elasticsearch/reference/5.5/query-dsl-match-query.html)，指定的匹配条件是：desc字段里包含"软件这个分词"，返回结果如下：

```json
{
  "took":3,
  "timed_out":false,
  "_shards":{"total":5,"successful":5,"failed":0},
  "hits":{
    "total":1,
    "max_score":0.28582606,
    "hits":[
      {
        "_index":"accounts",
        "_type":"person",
        "_id":"1",
        "_score":0.28582606,
        "_source": {
          "user" : "张三",
          "title" : "工程师",
          "desc" : "数据库管理，软件开发"
        }
      }
    ]
  }
}
```

#### 逻辑运算

如果有多个搜索关键字，Elastic认为它们是"or"关系：

```bash
curl 'localhost:9200/accounts/person/_search' -d '
	{
		"query": {"match": {"desc": "软件 系统"}}
	}
'
```

上面代码搜索的是"软件or系统"。

## 常常遇见的问题

### 搜索时间过长

通常，当节点收到搜索请求时，会将该请求传达给索引中每个分片的副本。

自定义路由允许将相关数据存储在相同的分片上，以便您只需搜索单个分片来满足查询。

例如：

```bash
curl  -XPUT "localhost:9200/blog_index" -d '
	"mappings": {
    "blogger": {
      "_routing": {
        "required": true 
      }
    }
  }
'
```

当您准备索引与blogger1相关的文档时，请指定路由值：

```bash
curl -XPUT "localhost:9200/blog_index/blogger/1?routing=blogger1" '{
	"comment": "blogger1 made this cool comment"
}'
```

现在，为了搜索blogger1的注释，您需要记住在查询中指定路由值，如下所示：

```bash
curl -XGET "localhost:9200/blog_index/_search?routing=blogger1" -d '
{
  "query": {
    "match": {
      "comment": {
        "query": "cool comment"
      }
    }
  }
}'
```



 	



