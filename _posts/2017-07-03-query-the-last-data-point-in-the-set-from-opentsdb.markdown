---
layout:     post
title:      "Query the last data point in the set from OpenTSDB"
subtitle:   ""
date:       2017-07-03 18:38:16
author:     "Echo Yuan"
#header-img: "img/in-post/hello-world/hello-world.gif"
tags:
    - OpenTSDB
---
在业务开发中，我们可能经常会碰到这样的需求：先按某个字段分组，然后在分组结果中再按其他字段排序，最后在每组中分别取出排序后的某条记录甚至某几条记录。

这篇博文是想写在OpenTSDB中怎么做到的，不过，先偏个题，我们来看看在MySQL中应该怎样实现？请参考Stackoverflow上的这个[答案](https://stackoverflow.com/questions/463054/sql-select-nth-member-of-group)。其实不难，一条SQL语句就可以做到。

下面回归正题。

先放上官方文档中对[first/last aggregator](http://opentsdb.net/docs/build/html/user_guide/query/aggregators.html#first-last)的描述:

> These aggregators will return the first or the last data point in the downsampling interval. E.g. if a downsample bucket consists of the series 2, 6, 1, 7 then the first aggregator will return 1 and last will return 7. Note that this aggregator is only useful for downsamplers.

我们先来个基础的查询，即找出某个指标(Metric)在指定时间段内的所有数据。
```
POST http://localhost:4242/api/query
{
    "start": "2017/07/05-09:30:00",
    "end": "2017/07/05-10:00:00",
    "queries": [
        {
            "metric": "bandwidth.latency4",
            "aggregator": "none"
        }
    ]
}
```
这里我们使用了`none`这个aggregator，它的意思是
> Skips group by aggregation. This aggregator is useful for fetching the raw data from storage as it will return a result set for every time series matching the filters. Note that the query will throw an exception if used with a downsampler.

即我们不需要对数据进行group。更多的aggregator请参考[这里](http://opentsdb.net/docs/build/html/user_guide/query/aggregators.html#available-aggregators)，有一些不好理解，需要仔细研读下文档。

Response:
```
[
    {
        "metric": "bandwidth.latency4",
        "tags": {
            "endpoint": "test7-101",
            "ip": "212.9.111.101",
            "bandwidth": "101M"
        },
        "aggregateTags": [],
        "dps": {
            "1499218344": 448,
            "1499218359": 950,
            "1499218374": 880,
            "1499218389": 198,
            "1499218404": 884
        }
    },
    {
        "metric": "bandwidth.latency4",
        "tags": {
            "endpoint": "test7-202",
            "ip": "212.9.111.202",
            "bandwidth": "202M"
        },
        "aggregateTags": [],
        "dps": {
            "1499218344": 845,
            "1499218359": 332,
            "1499218374": 707,
            "1499218389": 951,
            "1499218404": 121
        }
    }
]
```
在Web Query UI上做同样的查询，有图看起来更轻松些：

![graph-on-none-aggregator](/img/in-post/query-the-last-data-point-in-the-set-from-opentsdb/graph-on-none-aggregator.png)

我们先使用[Downsample](http://opentsdb.net/docs/build/html/user_guide/query/downsampling.html)的方式来获取the last data point，在请求体中添加一个`"downsample": "0all-last"`参数即可。
```
{
    "start": "2017/07/05-09:30:00",
    "end": "2017/07/05-10:00:00",
    "queries": [
        {
            "metric": "bandwidth.latency4",
            "aggregator": "none",
            "downsample": "0all-last"
        }
    ]
}
```
Response：
```
[
    {
        "metric": "bandwidth.latency4",
        "tags": {
            "endpoint": "test7-101",
            "ip": "212.9.111.101",
            "bandwidth": "101M"
        },
        "aggregateTags": [],
        "dps": {
            "1499218200": 884
        }
    },
    {
        "metric": "bandwidth.latency4",
        "tags": {
            "endpoint": "test7-202",
            "ip": "212.9.111.202",
            "bandwidth": "202M"
        },
        "aggregateTags": [],
        "dps": {
            "1499218200": 121
        }
    }
]
```
The last data point取出来了，但是timestamp却并不是我们所期望的，那它的值是怎么取的呢？在Downsampling的[文档](http://opentsdb.net/docs/build/html/user_guide/query/downsampling.html)里有描述。
> When using the `0all-` interval, the timestamp of the result will be the start time of the query.

那什么是`start time of the query`呢？其实就是你在request body中指定的这个`"start": "2017/07/05-09:30:00"`参数。

那再去Web Query UI上做Downsample查询会是什么样的呢？勾选上`Downsample`并将其下的选项设置为`last-10m-none`，得到下图：

![graph-on-none-aggregator](/img/in-post/query-the-last-data-point-in-the-set-from-opentsdb/graph-on-downsample.png)

看这张图眼神儿一定要好，不然很难发现那两个data point在哪里。如果你还没发现的话，请仔细观察Y轴。可惜的是这两个data point在图上的timestamp也是`2017/07/05-09:30:00`。

那我们再次验证下，在Web Query UI上将`From`改成`2017/07/05-09:20:00`，`Downsample`设置成`20m-last-none`。

![graph-on-none-aggregator](/img/in-post/query-the-last-data-point-in-the-set-from-opentsdb/graph-on-downsample-2.png)

结果还是一样的。虽然返回了the last data point，但是唯独遗失了data point中timestamp的值，而在某些情况下，这个值又是有意义的。

上面是尝试用Downsample的方式去实现我们的目标，而根正苗红的方式应该是用HTTP API中的[/api/query/last](http://opentsdb.net/docs/build/html/api_http/query/last.html)。

我当前使用的OpenTSDB版本是2.3.0，为了调出api-query-last的期望结果花费了蛮多的时间来搜索、Debug源码和测试。以下描述的是我自己调试的结果。

首先添加以下配置到opentsdb.conf文件里
```
tsd.core.meta.enable_tsuid_tracking = true
```
这是[Metadata](http://opentsdb.net/docs/build/html/user_guide/metadata.html)相关的内容，请自行阅读文档并深入理解。说实话，我自己理解的也不透彻，唯有动手多做些例子来测试下了。和Metadata相关的知识还有[UIDs and TSUIDs](http://opentsdb.net/docs/build/html/user_guide/uids.html)，我们从这些里面应该可以窥见一些OpenTSDB的设计哲学。

既然改了配置，那就重新造些测试数据吧~

```
POST http://localhost:4242/api/query
{
    "start": "2017/07/05-15:50:00",
    "end": "2017/07/05-17:00:00",
    "queries": [
        {
            "metric": "bandwidth.latency5",
            "aggregator": "none"
        }
    ]
}
```
Response:
```
[
    {
        "metric": "bandwidth.latency5",
        "tags": {
            "endpoint": "test8-303",
            "ip": "212.9.111.303",
            "bandwidth": "303M"
        },
        "aggregateTags": [],
        "dps": {
            "1499241290": 837,
            "1499241305": 443,
            "1499241320": 909,
            "1499241335": 496,
            "1499241350": 377
        }
    },
    {
        "metric": "bandwidth.latency5",
        "tags": {
            "endpoint": "test8-404",
            "ip": "212.9.111.404",
            "bandwidth": "404M"
        },
        "aggregateTags": [],
        "dps": {
            "1499241290": 378,
            "1499241305": 914,
            "1499241320": 511,
            "1499241335": 709,
            "1499241350": 992
        }
    }
]
```

用api-query-last取出来看看

```
POST http://localhost:4242/api/query/last
{
    "queries": [
        {
            "metric": "bandwidth.latency5",
            "tags": {
            	"endpoint": "test8-303"
            }
        }
    ],
    "resolveNames":true
}
```
Response:
```
[
    {
        "metric": "bandwidth.latency5",
        "timestamp": 1499241350000,
        "value": "377.0",
        "tags": {
            "endpoint": "test8-303",
            "bandwidth": "303M",
            "ip": "212.9.111.303"
        },
        "tsuid": "000011000003000081000005000082000010000083"
    }
]
```
我们得到了期望的the last data point且它的timestamp也和我们push的时间点相吻合。

那在Request body中去掉tags参数再试试呢？

```
POST http://localhost:4242/api/query/last
{
    "queries": [
        {
            "metric": "bandwidth.latency5"
        }
    ],
    "resolveNames":true
}
```
Response:

```
[
    {
        "metric": "bandwidth.latency5",
        "timestamp": 1499241350000,
        "value": "377.0",
        "tags": {
            "endpoint": "test8-303",
            "bandwidth": "303M",
            "ip": "212.9.111.303"
        },
        "tsuid": "000011000003000081000005000082000010000083"
    },
    {
        "metric": "bandwidth.latency5",
        "timestamp": 1499241350000,
        "value": "992.0",
        "tags": {
            "endpoint": "test8-404",
            "bandwidth": "404M",
            "ip": "212.9.111.404"
        },
        "tsuid": "000011000003000084000005000085000010000086"
    }
]
```
也是正确的。

遗憾的是，到现在仍然不能通过以下方式
```
POST http://localhost:4242/api/query/last
{
    "queries": [
        {
            "metric": "bandwidth.latency5",
            "tags": {
            	"endpoint": "test8-303"
            }
        }
    ],
    "resolveNames":true,
    "backScan":24
}
```
调试出来期望的结果。

后续再努力吧~