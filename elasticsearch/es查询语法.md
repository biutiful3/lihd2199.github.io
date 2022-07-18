##### es查询语法

###### 1、查看index的mapping

```sql
show create table loh
```

```http
GET logging_snowflake-audit_2022.06.20/_mapping
```

###### 2、单字段等值查询

```sql
select * from log where logLevel = "ERROR"
```

```jSON
POST logging_snowflake-audit_2022.06.20/_search
{
  "query": {
    "term": {
      "logLevel": {
        "value": "ERROR"
      }
    }
  }
}
```

###### 3、单字段多值查询

```sql
select * from log where logLevel in ("WARN","ERROR")
```

```json
POST logging_snowflake-audit_2022.06.20/_search
{
  "query": {
    "terms": {
      "logLevel": [
        "WARN",
        "ERROR"
      ]
    }
  }
}
```

> 分词查询，不能对keyword字段进行like

###### 4、单字段分词查询

```sql
select * from log where message like "%视频%" or message like '%弹幕%'
```

```json
POST logging_snowflake-audit_2022.06.20/_search
{
  "query": {
    "match": {
      "message": "视频弹幕"
    }
  }
}
```

match不是单纯的like查询，会将“视频弹幕”做分词，查询的结果做like查询

###### 5、单字段分组查询

```sql
select * from log where message like "%视频弹幕%'
```

```json
POST logging_snowflake-audit_2022.06.20/_search
{
  "query": {
    "match_phrase": {
      "message": "视频弹幕"
    }
  }
}
```

match_phrase直接进行like，不会对‘视频弹幕’做分词

###### 6、单字段范围查询

```sql
select * from log where timestamp > '2022-06-20 12:01:40.385'
```

```json
POST logging_snowflake-audit_2022.06.20/_search
{
  "query": {
    "range": {
      "timestamp": {
        "gte": "2022-06-20 12:01:40.385",
        "lte": "2022-06-20 18:01:40.385"
      }
    }
  }
}
```

###### 7、分页查询

```sql
select * from log order by logLevel desc limit 0,20
```

```json
POST logging_snowflake-audit_2022.06.20/_search
{
  "query": {
    "range": {
      "timestamp": {
        "gte": "2022-06-20 12:01:40.385",
        "lte": "2022-06-20 18:01:40.385"
      }
    }
  },
  "from": 0,
  "size": 5,
  "sort": [
    {
      "logLevel": {
        "order": "desc"
      }
    }
  ]
}
```

###### 8、复合条件查询

```sql
select
  *
from
  log
where
  (
    logLevel = 'ERROR'
    and timestamp >= '2022-06-20 12:01:40.385'
    and timestamp <= '2022-06-20 18:01:40.385'
    and message like 'init datasource error'
  )
  and (message not like 'ImAccountCensorService')
```

> must 代表and，  should代表or， must_not代表 not

```json
POST logging_snowflake-audit_2022.06.20/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "term": {
            "logLevel": {
              "value": "ERROR"
            }
          }
        },
        {
          "range":{
            "timestamp": {
              "gte": "2022-06-20 12:01:40.385",
              "lte": "2022-06-20 18:01:40.385"
            }
          }
        },
        {
          "match_phrase": {
            "message": "init datasource error"
          }
        }
      ],
      "must_not": [
        {
          "match_phrase": {
            "message": "ImAccountCensorService"
          }
        },
        {
          "match": {
            "message": "NotWritablePropertyException"
          }
        }
      ]
    }
  }
}
```

> filter 和must类似，不计算分数

```json
POST logging_snowflake-audit_2022.06.20/_search
{
  "query": {
    "bool": {
      "filter": [
        {
          "term": {
            "logLevel": {
              "value": "ERROR"
            }
          }
        },
        {
          "range":{
            "timestamp": {
              "gte": "2022-06-20 12:01:40.385",
              "lte": "2022-06-20 18:01:40.385"
            }
          }
        },
        {
          "match_phrase": {
            "message": "init datasource error"
          }
        }
      ]
    }
  }
}
```

> should和bool复合

```json
POST logging_snowflake-audit_2022.06.20/_search
{
  "query": {
    "bool": {
      "should": [
        {
          "term": {
            "logLevel": {
              "value": "ERROR"
            }
          }
        },
        {
          "term": {
            "threadName": {
              "value": "main"
            }
          }
        },
        {
          "bool": {
            "must": [
              {
                "term": {
                  "loggerName": {
                    "value": "com.xueqiu.dao.im.IMDataSourceManager"
                  }
                }
              },{
                "match_phrase": {
                  "message": "datasource"
                }
              }
            ]
          }
        }
      ]
    }
  }
}
```

###### 9、返回指定查询字段

```sql
select logLevel ,message,timestamp from log
```

```json
POST logging_snowflake-audit_2022.06.20/_search
{
  "_source": ["logLevel","message","timestamp"], 
  "query": {
    "match_all": {}
  }
}
```

###### 10、聚合查询

```sql
select count(1) from log where   group by logLevel
```

```json
POST logging_snowflake-audit_2022.06.20/_search
{
  "aggregations": {
    "logLevelAgg": {
      "terms": {
        "field": "logLevel",
        "size": 10
      }, 
      "aggregations": {
        "countAgg": {
          "value_count": {
            "field": "logLevel"
          }
        }
      }
    }
  }
}
```

