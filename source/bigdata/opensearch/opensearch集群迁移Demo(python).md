---
title: opensearch集群迁移Demo(Python)
categories: OpenSeacrh
tags: 
    - Python
---
# opensearch集群迁移Demo

声明：
1. 旧集群URL：URL1，新集群URL：URL2
2. note：以下GET和PUT都可都是表示HTTP请求，下边是python的例子

``` python
import requests
# get
a = requests.get(url, auth=awsauth, json=document, headers=headers)
print(a.status_code)
print(a.text)

# put
b = requests.put(url, auth=awsauth, json=document, headers=headers)
print(b.status_code)
print(b.text)
```


1. python 使用boto3上传数据
```python
import boto3
import requests
from requests_aws4auth import AWS4Auth

region = 'cn-north-1' # e.g. us-west-1
service = 'es'
credentials = boto3.Session().get_credentials()
awsauth = AWS4Auth(credentials.access_key, credentials.secret_key, region, service, session_token=credentials.token)

host = 'URL1' # the OpenSearch Service domain, including https://
# index = 'test'
# type = '_doc'
url = ''

headers = { "Content-Type": "application/json" }

id = "1"
# Create the JSON document
document = { "id": id, "timestamp": "23232323", "message": "msg" }
# Index the document
r = requests.put(url + id, auth=awsauth, json=document, headers=headers)
print("code", r.status_code)
print("text", r.text)
```

2. postman查询数据是否写入(也可以使用python的request.get()方法)

查看数据：

GET URL1/test/_doc/_search

```json
{
    "took": 18,
    "timed_out": false,
    "_shards": {
        "total": 5,
        "successful": 5,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": {
            "value": 1,
            "relation": "eq"
        },
        "max_score": 1.0,
        "hits": [
            {
                "_index": "test",
                "_type": "_doc",
                "_id": "1",
                "_score": 1.0,
                "_source": {
                    "id": "1",
                    "timestamp": "23232323",
                    "message": "msg"
                }
            }
        ]
    }
}
```

3. 注册存储库，制作当前快照

a) 注册存储库
PUT URL1/_snapshot/repo1
{
"type": "s3",
"settings": {
"bucket": "ostest1",
"region": "cn-north-1",
"role_arn": "arn:aws-cn:iam::account_id:role/opensearch_snap"
}
}

返回结果:
```json
{
    "acknowledged": true
}
```

b) 查看注册的存储库：

GET URL1/_snapshot/

返回结果:
```json
{
    "cs-automated-enc": {
        "type": "s3"
    },
    "repo1": {
        "type": "s3",
        "settings": {
            "bucket": "ostest1",
            "region": "cn-north-1",
            "role_arn": "arn:aws-cn:iam::account_id:role/opensearch_snap"
        }
    }
}
```

c) 制作手动快照
PUT URL1/_snapshot/repo1/1

返回结果:
```json
{
    "accepted": true
}
```

d) 查询快照备份状态, 如下结果表示备份已经完成，没有正在进行的快照

GET URL1/_snapshot/repo1/_status

返回结果:
```json
{
    "snapshots": []
}
```

e) 查看自动快照的备份：

GET URL1/_snapshot/cs-automated-enc/_all?pretty


4. 新建集群从旧存储库拉取数据

a) 新域中注册相同的快照存储库，在IAM用户中加入对新集群做ESHttpGet和ESHttpPut的权限。

PUT URL2/_snapshot/repo2
{
"type": "s3",
"settings": {
"bucket": "ostest1",
"region": "cn-north-1",
"readonly": true,
"role_arn": "arn:aws-cn:iam::account_id:role/opensearch_snap"
}
}

```json
{
    "acknowledged": true
}
```

b) 查看所有的手动快照

GET URL2/_snapshot/repo2/_all

```json
{
    "snapshots": [
        {
            "snapshot": "1",
            "uuid": "0ThYVD33SGyEhIGHzJ38NA",
            "version_id": 135238227,
            "version": "1.2.4",
            "indices": [
                ".kibana_3556_os_1",
                "test",
                ".kibana_1",
                "opensearch_dashboards_sample_data_ecommerce",
                ".opendistro_security"
            ],
            "data_streams": [],
            "include_global_state": true,
            "state": "SUCCESS",
            "start_time": "2022-07-21T02:31:55.210Z",
            "start_time_in_millis": 1658370715210,
            "end_time": "2022-07-21T02:31:57.611Z",
            "end_time_in_millis": 1658370717611,
            "duration_in_millis": 2401,
            "failures": [],
            "shards": {
                "total": 9,
                "failed": 0,
                "successful": 9
            }
        }
    ]
}
```

c) 还原数据, 这里以还原test这个索引为例

POST URL2/_snapshot/repo2/1/_restore
{"indices": "test"}



5. 新集群查看拉取数据和旧集群是否一致

GET URL2/test/_doc/_search

```json
{
    "took": 42,
    "timed_out": false,
    "_shards": {
        "total": 5,
        "successful": 5,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": {
            "value": 1,
            "relation": "eq"
        },
        "max_score": 1.0,
        "hits": [
            {
                "_index": "test",
                "_type": "_doc",
                "_id": "1",
                "_score": 1.0,
                "_source": {
                    "id": "1",
                    "timestamp": "23232323",
                    "message": "msg"
                }
            }
        ]
    }
}
```


6. 新集群注册新的储存库制作快照

这里需要在与开始新建的snapshot的role上加上新的存储桶testos2的权限。

a) 注册储存库

PUT URL2/_snapshot/repo3
```json
{
  "type": "s3",
  "settings": {
    "bucket": "testos2",
    "region": "cn-north-1",
    "role_arn": "arn:aws-cn:iam::account_id:role/opensearch_snap"
  }
}
```


b) 制作手动快照

PUT URL2/_snapshot/repo3/3



[GitHub - elasticsearch-dump/elasticsearch-dump: Import and export tools for elasticsearch](https://github.com/elasticsearch-dump/elasticsearch-dump)