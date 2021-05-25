---
layout:     post
title:      "How to understand span-not query in Elasticsearch"
subtitle:   ""
date:       2021-05-25 16:00:00
author:     "Echo Yuan"
tags:
    - Elasticsearchs
    - pan not query
---
最近在学习Elasticsearch，在看到[span not](https://www.elastic.co/guide/en/elasticsearch/reference/7.2/query-dsl-span-not-query.html) query的时候一头雾水，官方也没给出更详细的例子。如鲠在喉，难受。

经过一番搜索和实践，得出了一点儿经验。

##### 先定义Mapping
```json
PUT /span_not_query_test

{
  "mappings": {
    "properties": {
      "content": {
        "type": "text"
      }
    }
  }
}
```

##### 造两条数据
```json
PUT /span_not_query_test/_doc/1

{
  "content":"the quick red fox jumps over the sleepy cat"
}

PUT /span_not_query_test/_doc/2

{
  "content":"the quick brown fox jumps over the lazy dog"
}
```

##### 例子1
```json
POST /span_not_query_test/_search

{
  "query": {
    "span_not": {
      "include": {
        "span_term": {
          "content": {
            "value": "quick"
          }
        }
      },
      "exclude": {
        "span_term": {
          "content": {
            "value": "the"
          }
        }
      }
    }
  }
}
```
结论：
    
`exclude.span_term.content.value` == `quick`，无文档返回；否则，会返回两个文档。

##### 例子2
```json
POST /span_not_query_test/_search

{
  "query": {
    "span_not": {
      "include": {
        "span_near": {
          "clauses": [
            {
              "span_term": {
                "content": {
                  "value": "quick"
                }
              }
            },
            {
              "span_term": {
                "content": {
                  "value": "over"
                }
              }
            }
          ],
          "slop": 3,
          "in_order": true
        }
      },
      "exclude": {
        "span_term": {
          "content": {
            "value": "lazy"
          }
        }
      }
    }
  }
}
```
实验结果如下：

1. `exclude.span_term.content.value` in [`quick`, `fox`, `jumps`, `over`]，无文档返回；
2. `exclude.span_term.content.value` == `red`，只返回了`the quick brown fox jumps over the lazy dog`这一个文档；
3. `exclude.span_term.content.value` == `brown`，只返回了`the quick red fox jumps over the sleepy cat`这一个文档；
4. `exclude.span_term.content.value` in [`the`, `over`, `lazy`, `dog`, `sleepy`, `cat`]，即如果是content中`quick`之前的任意terms或`over`之后的任意terms，都会返回这两个文档。

结论：

`exclude`和`must_not`的工作方式不一样，它并不会把符合自身条件的docs查询出来然后再从`include`的结果中remove掉它们，而只是在条件这一层面上判断是否包含在`include`的条件范围内。

##### 参考资料
[https://stackoverflow.com/questions/24260103/spannotquery-giving-unexpected-results-exclude-is-ignored](https://stackoverflow.com/questions/24260103/spannotquery-giving-unexpected-results-exclude-is-ignored)
[https://elasticsearch.cn/article/13677](https://elasticsearch.cn/article/13677)
