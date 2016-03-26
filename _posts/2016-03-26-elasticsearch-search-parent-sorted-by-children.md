---
layout: post
title:  "elasticsearch: 返回parent文档, 但是按children里面的值排序"
date:   2016-03-26 16:37:38 +0800
modifydate:   2016-03-26 16:37:38 +0800
categories: elasticsearch
categoriy: elasticsearch
tag: [elasticsearch]
---

# 背景, 需求

有业务部门要用elasticsearch做酒店搜索, 大概的数据类型如下.

```js
#parent
POST indexname/hotel/_mapping
{
   "properties": {
      "hotelid": {
         "type": "long"
      },
      "name": {
         "type": "string"
      }
   }
}

#child
POST indexname/room/_mapping
{
   "_parent": {
      "type": "hotel"
   },
   "properties": {
      "roomid": {
         "type": "long"
      },
      "price": {
         "type": "long"
      },
      "area": {
         "type": "float"
      }
   }
}
```

父类型是酒店, 下面有子类型, 是房间, 每个房间的面积和价格不一样.

搜索的时候, 结果是展示酒店, 但以每个房间的最小价格排序.

有一方案是把room做为一个字段(列表)放到hotel里面去, 但这样的话, 每更新一个房间信息, 其实都是更新了整个hotel文档. 所以还是希望做成parent/children结构.

用has_child搜索方式, 再结合自定义打分功能, 可以实现.

# 准备一下数据
插入几条酒店信息

```js
POST indexname/hotel/1
{
   "hotelid": 1,
   "name": "abc"
}
POST indexname/hotel/2
{
   "hotelid": 2,
   "name": "xyz"
}
```

再插入几条房间信息

```js
POST indexname/room/a?parent=1
{
   "price": 100
}
POST indexname/room/b?parent=2
{
   "price": 200
}
POST indexname/room/c?parent=3
{
   "price": 300
}
```

# 搜索

```sh
POST indexname/hotel/_search
{
   "query": {
      "has_child": {
         "type": "room",
         "score_mode": "max",
         "query": {
            "function_score": {
               "query": {},
               "script_score": {
                  "script": "-1*doc['price'].value"
               }
            }
         }
      }
   }
}
```

搜索结果

```js
{
   "took": 9,
   "timed_out": false,
   "_shards": {
      "total": 5,
      "successful": 5,
      "failed": 0
   },
   "hits": {
      "total": 2,
      "max_score": -100,
      "hits": [
         {
            "_index": "indexname",
            "_type": "hotel",
            "_id": "1",
            "_score": -100,
            "_source": {
               "hotelid": 1,
               "name": "abc"
            }
         },
         {
            "_index": "indexname",
            "_type": "hotel",
            "_id": "2",
            "_score": -200,
            "_source": {
               "hotelid": 2,
               "name": "xyz"
            }
         }
      ]
   }
}
```
