title:  "ElasticSearch上手指南"
date: 2016-02-25
categories:
- search
tags:
- search
---

> 一句话说服你用elasticsearch，“github用elasticsearch实时检索1300亿行代码”

## ElasticSearch的基本特征
- 几乎实时的索引，大概有1秒的延迟，就能使被索引的数据被检索到
- 一台elastic server只能加入到一个cluster
- 索引名，例如一个blog的索引
- type，是一个索引名的一个维度(category)，例如blog的数据(blog/data)， blog的评论(blog/comment)
- 文档，一个文档是索引名+type的能够被索引的数据
- 分片与备份，如果一个cluster有2台elastic server，那么默认情况下一个索引有5个主分片和5个备份


## ElasticSearch的安装与启动
- [官方网站下载直通](https://www.elastic.co/downloads/elasticsearch)
- 最好使用最近版OracleJDK，JDK 7也行
- 启动之前修改`bin/elasticsearch.yml`中的`cluster.name`和`node.name`，前者用来自动加入相同`cluster.name`的集群
- `bin/elasticsearch`启动，或者`bin/elasticsearch -d`以守护进程启动
- 启动完毕后`http://localhost:9200`

## 图形用户界面


## CRUD
- `Create`: `PUT /indexName/type/id {...}` 如果url中不提供id，es会生成一个。同一个url如果PUT多次会累加`_version`，并且新的版本会替换老的版本
- `Update`: `POST /indexName/type/id/_update {'doc':{'field':value}}`
- `Delete`: `DELETE url`
- `Read`: `GET url`
- 批量索引：`POST /indexName/type/_bulk {"index":{"_id":n}\n{json}}`

## Search
### 例子0：搜所有
``` javascript
curl -XPOST 'localhost:9200/bank/_search?pretty' -d '
{
  "query": { "match_all": {} },
  "sort": { "balance": { "order": "desc" } }
}'
```
### 例子1：模糊匹配keywords
``` javascript
Post /indexName/type/_search
{
  "query": {
    "match":{
      "field": keywords
    }
  },
  "size":n
}
```
### 例子2：完整匹配keywords
``` javascript
Get /indexName/type/_search
{
  "query": {
    "match_phrase":{
      "field": keywords
    }
  },
  "from":fromNo,
  "size":n
}
```
### 例子3：多个条件组合
``` javascript
Get /indexName/type/_search
{
  "query": {
    "bool":{
      "must": [
        "match": {
          "field": keyword1
        }
        "match_phrase":{

        }
      ]
    }
  }
}
```
### 例子4：返回部分字段
``` javascript
curl -XPOST 'localhost:9200/bank/_search?pretty' -d '
{
  "query": { "match_all": {} },
  "_source": ["account_number", "balance"]
}'
```
### 例子5：Filter的例子
``` javascript
curl -XPOST 'localhost:9200/bank/_search?pretty' -d '
{
  "query": {
    "bool": {
      "must": { "match_all": {} },
      "filter": {
        "range": {
          "balance": {
            "gte": 20000,
            "lte": 30000
          }
        }
      }
    }
  }
}'
```
### 例子6：aggregation例子
``` javascript
curl -XPOST 'localhost:9200/bank/_search?pretty' -d '
{
  "size": 0,
  "aggs": {
    "group_by_state": {
      "terms": {
        "field": "state"
      }
    }
  }
}'
```
