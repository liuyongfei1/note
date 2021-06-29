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



 	



