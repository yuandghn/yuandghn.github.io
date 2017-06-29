---
layout:     post
title:      "A simple example on MySQL trigger"
subtitle:   ""
date:       2017-06-29 17:50:53
author:     "Echo Yuan"
#header-img: "img/in-post/hello-world/hello-world.gif"
tags:
    - MySQL
    - Open-Falcon
    - Falcon-Plus
---
请先看MySQL trigger[官方文档](https://dev.mysql.com/doc/refman/5.7/en/triggers.html)中的定义及注意事项，这里不再赘述；官方文档中的语法和示例在[这里](https://dev.mysql.com/doc/refman/5.7/en/trigger-syntax.html)，大家可以参考下。

* trigger是跟从属于表的，表删除了trigger也就没有了。
* trigger的name在同一个schema namespace下必须是惟一的，然而在不同的schema下可以有相同的trigger name。
* 从MySQL 5.7.2起，你可以在一张表上定义多个具有相同trigger event(如INSERT或UPDATE)和action time(如BEFORE或AFTER)的triggers，比如在一张表上定义两个BEFORE UPDATE triggers，默认情况下是根据它们被创建的顺序依次来执行的，当然你也可以通过在FOR EACH ROW子句后以FOLLOWS或PRECEDES + trigger name的方式来指定顺序。在5.7.2之前你是不能这么玩的。
* 使用`OLD`和`NEW`两个关键字来访问你的trigger想作用于的行列。它们不区别大小写但是有不同的作用场景。
  * 在INSERT event中，你只能使用`NEW`，因为没有old行可以给你用；
  * 在DELETE event中，你只能使用`OLD`，因为不会产生新行；
  * 在UPDATE event中，你可以同时使用`OLD`和`NEW`，它们分别可以引用更新前的column和更新后的column；
  * 使用`OLD`引用的column是只读的，你不能修改它的值。
  * In a BEFORE trigger, the NEW value for an AUTO_INCREMENT column is 0, not the sequence number that is generated automatically when the new row actually is inserted. 自己翻译始终不如看原文来的爽，:)
* 使用`BEGIN ... END`结构，可以让你的trigger执行多个语句，比如在trigger里做个条件判断啥的。

还有其他的限制大家自己看文档吧~

下面回到我自己的应用里，也就是[Falcon+](https://github.com/open-falcon/falcon-plus)。之所以有自定义trigger的需求，是因为想对告警数据做更方便的统计。

在我们的业务场景中，`endpoint`字段里存储的是`商家-门店id`，如`waipoyinxiang-26`。那如果我想分别统计各个商家的告警数和商家下各个门店的告警数的话，以现有的表结构无法方便快速的达到这个目的。你可能会说用[substring](https://dev.mysql.com/doc/refman/5.7/en/string-functions.html#function_substring-index)呀，反正endpoint上有索引，但是这样搞我觉得不好。

稍微优雅点儿，为event_cases表再多加两个字段`tenancy`和`store_id`，它们的值来源于对`endpoint`字段的[substring](https://dev.mysql.com/doc/refman/5.7/en/string-functions.html#function_substring-index)。这两个字段加到event_cases表里确实有点儿违和，哈哈~ 但是我又不想新建表，逃~

```
ALTER TABLE event_cases ADD tenancy varchar(32) DEFAULT NULL;

ALTER TABLE event_cases ADD store_id varchar(32) DEFAULT NULL;

CREATE INDEX tenancy_storeid ON event_cases (tenancy, store_id);

CREATE TRIGGER convert_endpoint_to_tenancy_and_storeid BEFORE INSERT ON event_cases
FOR EACH ROW SET
NEW.tenancy = SUBSTRING_INDEX(NEW.endpoint, '-', 1),
NEW.store_id = SUBSTRING_INDEX(NEW.endpoint, '-', -1);
```

除了加trigger，还想过要不就去改[alarm module](https://github.com/open-falcon/falcon-plus/blob/master/modules/alarm/model/event/event_operation.go)的源码吧，技术上问题倒不大，就是侵入性太强，想想还是算了。。。




