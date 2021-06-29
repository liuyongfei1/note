## 简介

全文搜索属于最常见的需求，开源的[Elasticsearch](https://www.elastic.co/) 是目前全文搜索引擎的首选。

它可以快速的存储、搜索和分析海量数据。维基百科、Stack Overflow、Github都采用它。

Elastic的底层是开源库 [Lucene](https://lucene.apache.org/)。但你没法直接使用Lucene，你必须自己写代码去调用它的接口。

Elastic是对Lucene的封装，提供了RESET API接口，开箱即用。