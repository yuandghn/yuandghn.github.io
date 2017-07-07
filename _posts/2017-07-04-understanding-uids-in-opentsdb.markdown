---
layout:     post
title:      "Understanding UIDs in OpenTSDB"
subtitle:   ""
date:       2017-07-04 21:06:13
author:     "Echo Yuan"
#header-img: "img/in-post/hello-world/hello-world.gif"
tags:
    - OpenTSDB
    - 翻译
---
只是按照自己的理解翻译一下官方的[UIDs](http://opentsdb.net/docs/build/html/user_guide/uids.html)文档，限于英文水平，仅做参考。

#### UIDs and TSUIDs
In OpenTSDB, when you write a timeseries data point, it is always associated with a metric and at least one tag name/value pair. Each metric, tag name and tag value is assigned a unique identifier (UID) the first time it is encountered or when explicitly assigned via the API or a CLI tool. The combination of metric and tag name/value pairs create a timeseries UID or TSUID.

当你向OpenTSDB中写入一个时间序列的数据点时，总有一个metric和至少一组tag name/value对儿与之相关联。每个metric、tag name和tag value都会被赋予一个唯一的标识符(UID)，时机是当它们第一次出现在OpenTSDB中时或者是被通过API/CLI工具显式赋值时。Metric和tag name/value对儿的组合又可以创建出一个timeseries UID 或者叫TSUID。

##### UID
Types of UID objects include:

UID对象包括以下几种类型：

* metric - A metric such as `sys.cpu.0` or `trades.per.second`
* tagk - A tag name such as `host` or `symbol`. This is always the "key" (the first value) in a tag key/value pair.
* tagv - A tag value such as `web01` or `goog`. This is always the "value" (the second value) in a tag key/value pair.

Assignment 赋值

The UID is a positive integer that is unique to the name of the UID object and it's type. Within the storage system there is a counter that is incremented for each metric, tagk and tagv. When you create a new tsdb-uid table, this counter is set to 0 for each type. So if you put a new data point with a metric of sys.cpu.0 and a tag pair of host=web01 you will have 3 new UID objects, each with a UID of 1.

UID是一个正整数且它对于某种类型的UID对象的name来说是惟一的。这句话拆开来说就是，你可以有一个名为`sys.cpu.0`的metric，同样也可以有一个名为`sys.cpu.0`的tagk，它们之间互不影响即便UID是一样的，因为metric和tagk分属于不同的UID类型。在存储系统内部对metric、tagk和tagv这三种类型的UID objects各自分配有一个自增的计数器。当`tsdb-uid`表初建时，每种类型的计数器的初始值都是0。所以，当你put一个新的data point[metric: sys.cpu.0, tagk: host, tagv: web01]进来时，你将会得到3个新的UID objects，每个object的UID都是1。

UIDs are assigned automatically for new tagk and tagv objects when data points are written to a TSD. metric objects also receive new UIDs but only if the auto metric setting has been configured to true. Otherwise data points with new metrics are rejected. The UIDs are looked up in a cached map for every incoming data point. If the lookup fails, then the TSD will attempt to assign a new UID.

当data points写入到TSD中时，会自动给新的tagk和tagv对象赋值UID，但是新的metric对象是不会自动赋值UID的，除非你在配置文件中显示地设置`tsd.core.auto_create_metrics = true`，否则会被拒绝写入。对每个进来的data point所包含的对象的UID的查找是在缓存中进行的，当查找不到时，才会为其赋予新的UID。


Storage 存储

By default, UIDs are encoded on 3 bytes in storage, giving a maximum unique ID of 16,777,215 for each UID type. This is done to reduce the amount of space taken up in storage and to reduce the memory footprint of a TSD. For the vast majority of users, 16 million unique metrics, 16 million unique tag names and 16 million unique tag values should be enough. But if you do need more of a particular type, you can modify the OpenTSDB source code and recompile with 4 bytes or more. As of version 2.2 you can override the UID size via the config file.

默认情况下，UID是被编码为3个字节来存储的，由此我们可以计算出对于每种不同类型的UID对象，分别有16,777,215个UID可以使用。你问咋计算出来的？2的24次方-1，因为0不算哦。之所以这么做，也是为了减少TSD对磁盘和内存空间的占用。毕竟对于大多数人来说，1600w多个不重复的metrics，加上1600w多个不重复的tag names，再加上1600w多个不重复的tag values，怎么着也够用一辈子了。当然如果你确实有特殊需求的话，那就自己去改OpenTSDB的源码吧，然后把它重新编译成使用4个或更多字节的。好消息是，从2.2开始，你可以通过修改配置文件的方式来做这件事了，你说美不美？具体请参见[Properties](http://opentsdb.net/docs/build/html/user_guide/configuration.html#properties)表格中的最后三个属性。

Warning 警告

If you do adjust the byte encoding number, you must start with a fresh tsdb and fresh tsdb-uid table, otherwise the results will be unexpected. If you have data in an existing setup, you must export it, drop all tables, create them from scratch and re-import the data.

所谓，话不能乱说，药不能乱吃，字节数也不能乱改。如果你改了，那最好重新启动一个新的tsdb并刷新tsdb-uid表，否则后果自负，哈哈~ 如果你之前已经有数据了，抱歉，先导出再drop掉所有的表，然后重新导入进去。所以，从一开始你就要想好到底要不要做这件事；如果要做，那一开始就直接做掉，省的以后麻烦。

Display 展示

UIDs can be displayed in a few ways. The most common method is via the HTTP API where the 3 bytes of UID data are encoded as a hexadecimal string. For example, the UID of 1 would be written in binary as 000000000000000000000001. As an array of unsigned byte values, you could imagine it as [0, 0, 1]. Encoded as a hex string, the value would be 000001 where the string is padded with 0s for each byte. The UID of 255 would result in a hex value of 0000FF (or as a byte array, [0, 0, 255]. To convert between a decimal UID to a hex, use any kind of hex conversion tool you prefer and put 0s in front of the resulting value until you have a total of 6 characters. To convert from a hex UID to decimal, simply drop any 0s from the front, then use a tool to convert the hex string to a decimal.

UID可以以多种形式展示。最常见的形式是在HTTP API调用中将3个字节的UID编码为十六进制字符串来返回。举个例子，UID为1的二进制编码为`000000000000000000000001`，当把它当作一个无符号的字节数组看待时，可以把它想象成`[0, 0, 1]`的形式。再编码成十六进制字符串后就是`000001`，每个字节前面的部分以0来填充。同样，UID为255的十六进制形式是`0000FF`。

In some CLI tools and log files, a UID may be displayed as an array of signed bytes (thanks to Java) such as the above example of [0, 0, 1] or [0, 0, -28]. To convert from this signed array to an an array of unsigned bytes, then to hex. For example, -28 would be binary 10011100 which results in a decimal value of 156 and a hex value of 9C.

我觉得原文有误，这里按照我自己的理解写。而在CLI tools和日志文件中，一个UID可能会以有符号的字节数组的形式展示，比如上面两个例子也可以展示为`[0, 0, 1]` or `[0, 0, -127]`。这时候就要先将有符号的字节数组先转换成无符号的，然后再转换成十六进制展示。比如，`-127`的有符号二进制形式是`11111111`，对应的无符号数字是`255`，而`255`转换成十六进制表示就是FF。

Modification 变更

这一段原文里写的不太好理解，我觉得是因为作者没有(对我等小白们)重复地指出这样一个前提：所有的UID对象都是存储在`tsdb-uid`表里的，而所有的data points是存储在`tsdb`表里的，the metric UID and the UID for tagk/v pairs则是`tsdb`表里`Row Key`的组成元素之几。这里说的Modification是针对UID对象的。那再多说说`tsdb-uid`表。[这里](http://opentsdb.net/docs/build/html/user_guide/backends/hbase.html#uid-table-schema)有段话说的好：

> A separate, smaller table called tsdb-uid stores UID mappings, both forward and reverse. Two columns exist, one named name that maps a UID to a string and another id mapping strings to UIDs.

UIDs can be renamed or deleted. Renaming can be accomplished via the CLI and is generally safe but will affect EVERY time series that includes the renamed ID. E.g. if we have a series sys.cpu.user host=web01 and another apache.requests host=web01 and rename the web01 tag value to web01.mysite.org, then both series will now reflect the new host name and all queries referring to the old name must be updated. If a data point comes in that has the previous string, a new UID will be assigned.

UID对象可以被重命名或删除。重命名操作可以通过CLI(Command Line Interface)来完成，通常情况下它是安全的但是会对包含此UID对象的所有time series有影响。举例来说，假如有两个series，`sys.cpu.user host=web01`和`apache.requests host=web01`，它俩引用了同一个tagv `web01`。当你把`web01`重命名成`web01.mysite.org`时，这俩series都会自动响应这个新的host name，因为它们引用的是tagv对象的UID，你名字变归变，UID总是不变的。所以，当一个新的data point携带`web01`过来时，`web01`会被赋予一个新的UID。

Deleting UIDs can be tricky as of version 2.2. Deleting a metric is safe in that users may no longer query for the data and it won't show up in calls to the suggest API. However deleting a tag name or value can cause queries to fail. E.g. if you have time series for the metric sys.cpu.user with hosts web01, web02, web03, etc. and you delete the UID for web02, any query that would scan over data that includes the series sys.cpu.user host=web02 will throw an exception to the user because the data remains in storage. We highly recommend you run an FSCK with a query to repair such issues.

截至2.2版本，删除UID对象是件棘手的事情。删除某个metric UID是安全的行为，因为用户可能不会再对它进行查询而且它也不必出现在[suggest API](http://opentsdb.net/docs/build/html/api_http/suggest.html)的返回中。所谓的suggest API就是查询metric时不用输入完整的内容，这个API会给你提供`auto-complete`的功能并返回与你当前输入的内容相关联的在系统中已存在的metrics。而删除某个tag name/value UID则可能会导致查询失败。为啥会这样呢？想象一下，当你只指定metric来查询时，从`tsdb`表里查到了包含tags的数据，这些数据是用UID拼起来的，现在要将这些UID转换成对应的string表示，你不得从`tsdb-uid`表中查这个映射关系呀，可是好家伙，其中某个UID对象被你删了，那数据拼不成了呀，tsdb不报错报啥呀。不要狡辩，查询某个已经删除了的metric UID报的错是在`tsdb-uid`表这一层发生的，还没进到`tsdb`表里呢。

Why UIDs? 为啥要用UID呢？

This question is asked often enough it's worth laying out the reasons here. Looking up or assigning a UID takes up precious cycles in the TSD so folks wonder if it wouldn't be faster to use the raw name of the metric or computer a hash. Indeed, from a write perspective it would be slightly faster, but there are a number of drawbacks that become apparent.

这个问题经常被问到，看来有必要在这把原因说明白。查找或赋值一个UID会占用TSD宝贵的cycles，所以人们想知道使用原生的字符串或计算一个Hash值是否会更快些。的确，从写入的角度上来看这样会稍微快些，但是也有一些明显的缺点。

##### Raw Names 原生字符串

Since OpenTSDB uses HBase as the storage layer, you could use strings as the row key. Following the current schema, you may have a row key that looked like sys.cpu.0.user 1292148000 host=websv01.lga.mysite.com owner=operations. Ordering would be similar to the existing schema, but now you're using up 70 bytes of storage each hour instead of 19. Additionally, the row key must be written and returned with every query to HBase, so you're increasing your network usage as well. So resorting to UIDs can help save space.

OpenTSDB使用HBase作为存储层，因此你当然可以使用字符串作为row key。以这种方式举例，假设你有一个row key长这样`sys.cpu.0.user 1292148000 host=websv01.lga.mysite.com owner=operations`，排序方式还跟原来的UID一样，但是你的row key此时却必须以70个字节(row key总共70个字符，每个字符一个字节)来存储，而原来只需要19个字节((metric * 1 + tagk * 2 + tagv * 2) * 3 + timestamp * 1 * 4 = 19)。此外，每次查询，这个row key都要在HBase中写入和返回，无端增加了你的网络消耗。所以，使用UID可以帮助你节约存储空间。

##### Hashes

Another idea is to simply bump up the UIDs to 4 bytes then calculate a hash on the strings and store the hash with forward and reverse maps as we currently do. This would certainly reduce the amount of time it takes to assign a UID, but there are a few problems. First, you will encounter collisions where different names return the same hash. You could try different algorithms and even try increasing the hash to 8 bytes, but you'll always have the issue of colliding hashes. Second, you are now adding a hash calculation to every data put since it would have to determine the hash, then lookup the hash in the UID table to see if it's been mapped yet. Right now, each data point only performs the lookup. Third, you can't pre-split your HBase regions as easily. If you know you will have roughly 800 metrics in your system (the tags are irrelevant for this purpose), you can pre-split your HBase table to evenly distribute those 800 metrics and increase your initial write performance.

另一个主意是把UID提升到4个字节(为啥是4个字节，我想应该是由于在Java中hashcode()方法返回的是一个int值吧)并为string计算出一个Hash值，然后再像我们现在所做的那样存储Hash<-->string的双向映射关系。这确实可以减少赋值UID这个过程所花费的时间，但是却有一些其他的问题。首先你会碰到Hash碰撞/冲突的情况当不同的name返回相同的Hash值的情况下。你可能想着可以尝试不同的算法甚至将Hash的存储再提升到8个字节，但是你始终无法避免会遇到这个问题。

##### TSUIDs

When a data point is written to OpenTSDB, the row key is formatted as <metric_UID><timestamp><tagk1_UID><tagv1_UID>[...<tagkN_UID><tagvN_UID>]. By simply dropping the timestamp from the row key, we have a long array of UIDs that combined, form a unique timeseries ID. Encoding the bytes as a hex string will give us a useful TSUID that can be passed around various API calls. Thus from our UID example above where each metric, tag name and value has a UID of 1, our TSUID, encoded as a hexadecimal string, would be 000001000001000001.

当一个data point写入到OpenTSDB中时，它的row key的形式是`<metric_UID><timestamp><tagk1_UID><tagv1_UID>[...<tagkN_UID><tagvN_UID>]`。简单地从其中移除掉timestamp后，我们得到一个长长的UID数组的组合，这也就是我们所说的unique timeseries ID。将这些字节编码成十六进制的字符串后我们可以得到一个可以用于各种API调用的TSUID。拿最开头的那个例子来说，我们的TSUID就是这样一个十六进制的字符串`000001000001000001`。

While this TSUID format may be long and ugly, particularly with all of the 0s for early UIDs, there are a few reasons why this is useful:

虽然这样的TSUID格式有点儿丑长，前面还有老多前置0填充，但它实际用起来还是蛮适合的：

* If you know the width of each UID (by default 3 bytes as stated above), then you can easily parse the UID for each metric, tag name and value from the UID string.

  如果你知道每个UID的宽度(默认是3个字节哦我们前面说过)，那么你就可以轻松地从一个UID字符串中分离解析出来metric UID和每个tag name UID、tag value UID。

* Assigning a unique numeric ID for each timeseries creates issues with lock contention and/or synchronization issues where a timeseries may be missed if the UID could not be incremented.

  为每个timeseries分配一个unique numeric ID的方式可能会带来锁竞争和/或同步的问题当unique numeric ID无法自增的时候，并最终可能会导致这个timeseries遗失。

不知道在某些地方有没有曲解作者要表达的意思，如果有，请一定告知我，:)


