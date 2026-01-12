用于全文搜索  
可以快速地存储、搜索、分析海量数据 
## 一. 安装
docker安装  

```SHELL
C:\Users\lwx> curl.exe  localhost:9200         
{
  "name" : "8a04444f6f05",
  "cluster_name" : "elasticsearch",
    "number" : "7.12.0",
    "build_flavor" : "default",
    "build_type" : "docker",
    "build_hash" : "78722783c38caa25a70982b5b042074cde5d3b3a",
    "build_date" : "2021-03-18T06:17:15.410153305Z",
    "build_snapshot" : false,
    "lucene_version" : "8.8.0",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
```

## 二. 基本概念
### 2.1 Node 与 Cluster
Elastic 本质上是一个分布式数据库，允许多台服务器协同工作，每台服务器可以运行多个 Elastic 实例  
单个 Elastic 实例称为一个节点（node）。一组节点构成一个集群（cluster） 
### 2.2 Index
es 会索引所有字段，经过处理后写入一个反向索引（Inverted Index）。查找数据的时候，直接查找该索引  
==es数据管理的顶层单位：Index（单个数据库的同义词），每个Index的名字必须是小写==  

使用如下语句列出所有索引：
```BASH
curl.exe 'localhost:9200/_cat/indices?v'
```

![[Pasted image 20250425170829.png]]
### 2.3 Document
Index里单条的记录成为Document，许多条 Document 构成了一个 Index  
Document 使用 JSON 格式表示，下面是一个例子  
```json
{
  "user": "张三",
  "title": "工程师",
  "desc": "数据库管理"
}
```
同一个 Index 里面的 Document，==不要求有相同的结构（scheme），但是最好保持相同，这样有利于提高搜索效率==  
### 2.4 Type
Document 可以分组，如==weath==这个Index，可以按城市分组/天气分组  
这种分组就叫==Type==，他是==虚拟的逻辑分组，用来过滤Document==  
>[!建议]
 `Avoiding Type Gotchas`（避免类型陷阱）：不同的分组应该具==有相似的结构（schema）==，如`id`字段不能在这个组是字符串，在另一个组是数值  

下面的命令列出Index所包含的Type：
```bash
curl.exe 'localhost:9200/_mapping?pretty=true'
```
Elastic 6.x 版只允许每个 Index 包含一个 Type，7.x 版将会彻底移除 Type  

#### es为什么移除Type
index、type的设计初衷是为了方便管理数据之间的关系，类比database-table设计了index-type模型  
但是在关系型数据库中==table是独立的==，而es中同一个index中不同type是存储在同一个index中，因此不同type中相同名字的字段的定义（mapping）必须一致

### 2.5 数据类型
- 字符串类型：text、 keyword  
- 数值类型：long, integer, short, byte, double, float, half float, scaled float
- 日期类型：date
- 布尔值类型：boolean
- 二进制类型：binary

创建字段的类型：
![[Pasted image 20250425180952.png]]

---
## 三、新建和删除Index
新建一个叫`weather`的Index  
```bash
curl -X PUT 'localhost:9200/weather'
```
服务器返回一个 JSON 对象，里面的`acknowledged`字段表示操作成功
```json
{
  "acknowledged":true,
  "shards_acknowledged":true
}
```
我们发出 DELETE 请求，删除这个 Index。
```bash
curl -X DELETE 'localhost:9200/weather'
```

## 四. 数据操作
### 4.1 新增操作
如，向`/accounts/person`发送请求，就可以新增一条人员记录
```bash
curl -X PUT 'localhost:9200/accounts/person/1' -d '
{
  "user": "张三",
  "title": "工程师",
  "desc": "数据库管理"
}' 
```
服务器返回的 JSON 对象，会给出 Index、Type、Id、Version 等信息  
```json
{
  "_index":"accounts",
  "_type":"person",
  "_id":"1",
  "_version":1,
  "result":"created",
  "_shards":{"total":2,"successful":1,"failed":0},
  "created":true
}
```
==`/accounts/person/1`，最后的`1`是该条记录的 Id。它不一定是数字，任意字符串（比如`abc`）都可以==  

新增记录的时候，也可以不指定 Id，这时要改成 POST 请求  
```bash
curl -X POST 'localhost:9200/accounts/person' -d '
{
  "user": "李四",
  "title": "工程师",
  "desc": "系统管理"
}'
```
### 4.2 查看记录
向`/Index/Type/Id`发出 GET 请求，就可以查看这条记录。
```bash
curl 'localhost:9200/accounts/person/1?pretty=true'
```
URL的参数==pretty=true==表示以易读的格式返回  
```json
{
  "_index" : "accounts",
  "_type" : "person",
  "_id" : "1",
  "_version" : 1,
  "found" : true,
  "_source" : {
    "user" : "张三",
    "title" : "工程师",
    "desc" : "数据库管理"
  }
}
```
`found`字段表示查询成功，`_source`字段返回原始记录
### 4.3 删除记录
```bash
curl -X DELETE 'localhost:9200/accounts/person/1'
```
### 4.4 更新记录
```bash
curl -X PUT 'localhost:9200/accounts/person/1' -d '
{
    "user" : "张三",
    "title" : "工程师",
    "desc" : "数据库管理，软件开发"
}' 

{
  "_index":"accounts",
  "_type":"person",
  "_id":"1",
  "_version":2,
  "result":"updated",
  "_shards":{"total":2,"successful":1,"failed":0},
  "created":false
}
```

## 五、数据查询
### 5.1 返回所有数据
使用`GET`方法，直接请求`Index/Type/_search`
```bash
curl 'localhost:9200/accounts/person/_search'

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
`took`：操作耗时（ms）
`timed_out`：是否超时
`hits`：命中的记录
- `total`：返回记录数
- `max_source`：最高的匹配程度
- `hits`：返回的记录
### 5.2 全文搜索
es的查找需要GET请求带有结构体
```bash
curl 'localhost:9200/accounts/person/_search'  -d '
{
  "query" : { "match" : { "desc" : "软件" }},
  "size": 1,
  "from": 1
}'
```
使用`Match`查询，指定的匹配条件是`desc`字段中包含"软件"这个词  
es默认一次返回10条结果，通过`size`改变这个设置  
`from`指定位移，从位置`?`开始  
### 5.3 逻辑运算
如果有多个搜索关键字， ==es 认为它们是`or`关系==
```bash
curl 'localhost:9200/accounts/person/_search'  -d '
{
  "query" : { "match" : { "desc" : "软件 系统" }}
}'
```
如果要==执行多个关键词的`and`搜索，必须使用布尔查询==
```bash
curl 'localhost:9200/accounts/person/_search'  -d '
{
  "query": {
    "bool": {
      "must": [
        { "match": { "desc": "软件" } },
        { "match": { "desc": "系统" } }
      ]
    }
  }
}'
```
