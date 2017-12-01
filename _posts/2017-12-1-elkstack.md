---
layout:     post
title:      "elk stack 配置总结"
subtitle:   " \"devops\""
date:       2017-12-1
author:     "leiyiming"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - devops
---


> elk stack 总结，包括filebeat，logstah，elasticsearch，kibana等总结性文档



## filebeat

---

配置文件：

```yaml
filebeat:
  prospectors:
    - paths:
        - /var/log/envoy/access.log
      tail_files: true
      include_lines: ^\[
      document_type: envoy_access

    - paths:
        - /home/elk/filebeat/filebeat-dev/config/test.log
      tail_files: true
      include_lines: ^\[
      document_type: test_log

output:
  logstash:
      hosts: ["127.0.0.1:5044"] 

```

说明：

* 各个属性的含义，[官方文档](https://www.elastic.co/guide/en/beats/filebeat/current/configuration-filebeat-options.html) 说明很详细。
* `include_lines` 和 `exclude_lines` 可以通过正则表达式来过滤包含或者不包含某一个关键字的行。
* `fields`  字段可以让你自定义字段，通过自定义字段可以标注上想要的信息。
* 非常完整的示例配置：[链接](http://leiyiming.comimg/imgin-post/post-devops/elk/filebeat.yml)



## logstash

---

配置文件：

```yaml
input {
    beats {
        port => 5044
        host => "0.0.0.0"
    }
}
filter {
  if [type] == "envoy_access" {
    grok {
      match => {
        "message" => "\[%{TIMESTAMP_ISO8601:time}\] \"%{WORD:http_method} %{NOTSPACE:method} %{NOTSPACE:http_type}\" %{NUMBER:status} - %{NUMBER:num1} %{NUMBER:num2} %{NUMBER:delay}"
      }
      remove_field => ["http_method"]
    }
    mutate {
      convert => ["delay", "integer"]
      remove_field => ["beat.hostname"]
    }
  }
  if [type] == "test_log" {
    grok{
      match => {
        "message" => "\[%{LOGLEVEL:level} %{TIMESTAMP_ISO8601:time} %{NUMBER:num} %{NOTSPACE:file_name} \(%{WORD:line}\) %{WORD:function}\] %{GREEDYDATA:kvpairs}"
      }
    }
    kv {
      source => "kvpairs"
      remove_field => [ "kvpairs" ] # Delete the field afterwards
    }
  }
}

output {
    stdout {
        codec => rubydebug
    }
	if [type] == "envoy_access" {
        elasticsearch {
                action => "index"
                index => "envoy_access"
                hosts => ["127.0.0.1:9200"]
                manage_template => false
                template_name => "template_envoy_access"
                template_overwrite => true
        }
    }else if [type] == "test_log" {
        elasticsearch {
                index => "test"
                hosts => ["127.0.0.1:9200"]
                template_name => "template_test"
                template_overwrite => true
        }
    }
}
```

说明：

* logstash的插件非常丰富，filter和output部分都大量的插件可用。常用的filter插件有grok（正则表达式过滤）、kv（键值对过滤）等等。使用时也应该考虑到效率问题，能够通过规范制定日志格式的话，可以降低logstash所消耗的资源。
* `output` 中可以使用 `template_name` 指定 elasticsearch 模板，模板能够将filter插件解析出的各个字段添加上索引，这样在 elasticsearch 中才能对字段进行排序或者统计等操作。



## elasticsearch 

---

elasticsrearch 主要作用是查询，具体的使用可以参照官网，这里记录的主要是自定义模板的使用。

创建名为 `template_envoy_access` 的模板：

```yaml
curl -XPUT 'localhost:9200/_template/template_envoy_access?pretty' -d'
{
  "order": 0,
  "template": "envoy_access",
  "settings": {},
  "mappings": {
    "log": {
      "properties": {
        "message": {
          "index": "not_analyzed",
          "type": "string"
        },
        "method": {
          "index": "not_analyzed",
          "type": "string"
        },
        "delay": {
          "index": "not_analyzed",
          "type": "long"
        },
    	"status": {
          "index": "not_analyzed",
          "type": "long"
        },
    	"time": {
          "format": "strict_date_optional_time||epoch_millis",
          "type": "date"
        }
      }
    }
  },
  "aliases": {}
}'
```

动态模板：

```yaml
curl -XPUT 'localhost:9200/_template/template_test?pretty' -d'
{
  "order": 0,
  "template": "template_test",
  "settings": {},
  "mappings": {
    "test": {
      "dynamic_templates": [
        {
          "base": {
            "mapping": {
              "index": "not_analyzed"
            },
            "match": "*",
            "match_mapping_type": "*"
          }
        }
      ],
      "dynamic": true,
      "properties": {
        "message": {
          "index": "not_analyzed",
          "type": "string"
        },
        "level": {
          "index": "not_analyzed",
          "type": "string"
        },
        "num": {
          "index": "not_analyzed",
          "type": "long"
        },
    	"file_name": {
          "index": "not_analyzed",
          "type": "long"
        },
    	"time": {
          "format": "strict_date_optional_time||epoch_millis",
          "type": "date"
        },
        "function": {
          "index": "not_analyzed",
          "type": "long"
        },
        "line": {
          "index": "not_analyzed",
          "type": "long"
        }
      }
    }
  },
  "aliases": {}
}'
```

**静态模板** 会匹配字段名，然后设置字段类型并添加索引，这样就可以进行排序以及在kibana中进行可视化等操作。

**动态模板** 针对格式不固定的日志，它能够自动分析logstash解析出的字段，并动态添加索引，一般可以配合logstash的kv插件使用。

**注意：** 对于 elasticsrearch 模板来说，可以添加字段，但是**不能修改、删除字段** 。删除和修改字段要通过 `aliases` 来穿件新模板，然后将旧模板数据迁移过去，这个过程用户是感知不到的。所以，**创建模板必须添加`aliases` 字段**

删除 envoy_access 索引：

```
curl -XDELETE localhost:9200/envoy_access?pretty
```



## elastalert 

------

elastalert 可以监控某些字段然后进行告警

告警规则rule：

```yaml
name: envoy_alert
use_strftine_index: true
type: frequency
index: envoy*
num_events: 1

timeframe:
  minutes: 1

filter:
- range:
    delay:
      from: 1000
  - query:
      query_string:
        query: "delay:[1000 TO *]"

alert:
- command
command: [
  "/opt/rules/send_envoy_alert.sh"
]
```



