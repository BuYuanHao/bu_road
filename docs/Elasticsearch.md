# Elasticsearch技术分享

## 引言

### 简介

 ![](/img/Lucene.png)

    Lucene: Lucene是apache下一个开源的全文检索引擎工具包，但它不是一个完整的全文检索引擎，而是一个全文检索引擎的架构，提供了完整的查询引擎和索引引擎，部分文本分析引擎。

![](/img/Elasticsearch.png)

    Elasticsearch：是一个基于Lucene构建的开源、分布式、RESTful搜索引擎。它允许你以极快的速度进行全文搜索、结构化搜索、分析以及实时分析。Elasticsearch以其可扩展性、高可用性和易用性在日志分析、全文搜索、监控数据等领域得到了广泛应用。

### 全文检索

  数据可以分为：
  1. 结构化数据（固定格式、有限长度，比如mysql存的数据）
  2. 非结构化数据：不定长、无固定格式，比如邮件、Word文档、日志等
  3. 半结构化数据：前两者结合，比如 xml、html等

![alt text](/img/es和mysql对比.png)
  查询方式：
  1. mysql的 `like '%name%'`为顺序查询，需要遍历所有记录
  2. es是建立倒排索引：与mysql根据主键id索引到数据相反，数据对应到主键

  ![倒排索引](/img/倒排索引.png)

## 网站


Elasticsearch官方文档：
https://www.elastic.co/guide/en/elasticsearch/reference/current/elasticsearch-intro.html

## 安装

官方推荐云服务和docker部署的方式：
https://www.elastic.co/cn/downloads/elasticsearch

可视化工具kibana：

https://www.elastic.co/cn/downloads/kibana

## 配置

    Elasticsearch 有三个配置文件：
    1. elasticsearch.yml用于配置 Elasticsearch
    2. jvm.options用于配置 Elasticsearch JVM 设置
    3. log4j2.properties用于配置 Elasticsearch 日志记录

```yml
path:
#数据存储位置
  data: "C:\\Elastic\\Elasticsearch\\data"
#日志存储位置
  logs: "C:\\Elastic\\Elasticsearch\\logs"
#节点名称
node.name: logging-prod
```

## 用户
1. lastic 内置超级用户
2. ibana_system:Kibana 用于连接和与 Elasticsearch 通信的用户。
3. logstash_system:Logstash 在 Elasticsearch 中存储监控信息时使用的用户。
4. 可以使用以下命令修改用户密码：

    `elasticsearch-reset-password.bat --username elastic`

## 访问
es地址：http://localhost:9200/ 

kibana地址： http://localhost:5601/

使用命令获取token：

`elasticsearch-create-enrollment-token.bat --scope kibana`

## api

### 基础操作

```
#查询所有索引
GET _cat/indices?v

# 创建索引
put user 




# 查询单个索引
get user 

# 删除单个索引
delete user 

# 添加数据 随机id
post user/_doc
{
  "title":"小米收集",
  "name":"小卜",
  "age": 12
  
}

# 添加数据 指定id
post user/_doc/1001
{
  "title":"小米收集",
  "name":"小卜",
  "age": 12
  
}

# 获取数据 通过id获取
get user/_doc/1001

# 获取数据 全部查询
get user/_search

# 修改数据 (全量)
put user/_doc/1001
{
  "title":"小米收集",
  "name":"小卜",
  "age": 15
}
# 修改数据 (局部)
post user/_update/1001
{
  "doc":{
      "title":"苹果收集"
  }
}
# 删除数据 通过id
delete user/_doc/1001


```

### 查询


```

#url拼接查询
get user/_search?q=name:小卜

#body 条件查询
#math 分词匹配查询 会将单词切分进行模糊匹配查询
get user/_search
{
  "query":{
    "match": {
      "name":"小卜"
    } 
  }
}
#body条件全量查询
get user/_search
{
  "query":{
    "match_all": {
      
    } 
  }
}

#分页查询
get user/_search
{
  "query":{
    "match_all": {
      
    } 
  },
  #起始位置
  "from":0,
  #条数
  "size":1
}
#返回字段限制
get user/_search
{
  "query":{
    "match_all": {
      
    } 
  },
  "_source":["name"]
}
#排序
get user/_search
{
  "query":{
    "match_all": {
      
    } 
  },
  "sort":{
    "age":"desc"
  }
}


#多条件
# must 同时成立 and
get user/_search
{
  "query":{
    "bool": {
      
      "must": [
        {
          "match": {
            "name": "小卜"
          }
        },
        {
          "match": {
            "age": "15"
          }
        }
        
      ]
    } 
  }
}
#should满足一个就行  or
get user/_search
{
  "query":{
    "bool": {
      "should": [
        {
          "match": {
            "age": "12"
          }
        },
        {
          "match": {
            "age": "15"
          }
        }
        
      ]
    } 
  }
}
get user/_search
{
  "query":{
    "bool": {
      "filter": [
        {
          "range": {
            "age": {
              "gte": 13,
              "lte": 20
            }
          }
        }
      ]
    } 
  }
}
# 全等匹配
get user/_search
{
  "query":{
    "match_phrase": {
      "name":"小卜"
    } 
  }
}

#高亮查询
get user/_search
{
  "query":{
    "match": {
      "name":"卜"
    } 
  },
  "highlight":{
    "fields":{
      "name":{}
    }
  }
}
#聚合查询
//统计数量
get user/_search
{
  //聚合操作
  "aggs":{
    //操作名称
    "name_group":{
      //分组操作
      "terms": {
        //分组字段
        "field": "name.keyword"
      }
    }
  },
  //过滤原始数据
  "size":0
}
//平均值
get user/_search
{
  //聚合操作
  "aggs":{
    //操作名称
    "age_avg":{
      //求平均操作
      "avg": {
        //分组字段
        "field": "age"
      }
    }
  },
  //过滤原始数据
  "size":0
}

  
```

### 映射
put  user/_mapping
{
    "properties": {
        "name":{
        	"type": "text",
        	"index": true
        },
        "age":{
        	"type": "keyword",
        	"index": true
        },
        "title":{
        	"type": "keyword",
        	"index": false
        }
    }
}

  