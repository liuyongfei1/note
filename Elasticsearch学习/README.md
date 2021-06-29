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



