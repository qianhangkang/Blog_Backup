---
title: 关于ES倒排索引v6.2.4的一些小坑
date: 2018-04-29 21:47:59
tags: ['查询','ES']
---

刚开始学习，项目中准备用~~~

<!--more-->

## Q1

> no [query] registered for [filtered]

Stack Overflow上有大兄弟已经给出了解答

> The `filtered` query has been deprecated and removed in ES 5.0. You should now use the bool/must/filter query instead.

so

```
{
    "query" : {
        "bool": {
            "must": {
                "match" : {
                    "last_name" : "smith" 
                }
            },
            "filter": {
                "range" : {
                    "age" : { "gt" : 30 } 
                }
            }
        }
    }
}
```



## Q2

> Fielddata is disabled on text fields by default. Set `fielddata=true` on [`your_field_name`] in order to load fielddata in memory by uninverting the inverted index. Note that this can however use significant memory.

这个错误是因为ES官方默认关闭了fielddata，原因是fielddata会耗尽很多的heap堆内存，特别是加载high cardinality text 的字段。一旦fielddata被加载进堆内存，它将会在段的整个生命周期都存在。同时，加载fielddata也是代价高昂的，它将导致用户操作时存在停顿。这就是fielddata为什么默认关闭的原因。

**解决方案**

解决的问题就转化为了如何开启fielddata参数？

下面是官方文档给的例子

```
PUT my_index/_mapping/_doc
{
  "properties": {
    "my_field": { 
      "type":     "text",
      "fielddata": true
    }
  }
}
```

套用之前的官网例子上的URL就是

`PUT /megacorp/_mapping/employee`







# 参考链接

https://stackoverflow.com/questions/40519806/no-query-registered-for-filtered

https://elasticsearch.cn/book/elasticsearch_definitive_guide_2.x/index.html













