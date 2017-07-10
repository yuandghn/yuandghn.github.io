---
layout:     post
title:      "A few things recently got on Falcon plus"
subtitle:   ""
date:       2017-07-07 16:39:26
author:     "Echo Yuan"
#header-img: "img/in-post/hello-world/hello-world.gif"
tags:
    - Falcon-Plus
---
* 预警

  无法做到预警。预警就是当监控项的值超过设定的第一个阀值时触发第一个告警然后通知A团队的人介入，超过设定的第二个阀值后再触发第二个告警然后通知B团队的人介入，以此往后类推如果有需要的话。但是现在所有超过阀值的告警都会被归入第一个告警策略里，而且现在系统里的告警级别也不是这么用的。从template的设置里也可以得到这样的结论，因为只能设置报警触发函数 operator 某个值，而不能设置成一个区间。

* 当某一监控项的告警次数达到max step之后就不会再产生告警了，那如何解除这个状态？

  * 重启judge module是可以的，因为告警数据在内存里有一份。
  * 看源码里面还有个每5小时执行一次cleanStale来清除7天前的JudgeItem的[cronjob](https://github.com/open-falcon/falcon-plus/blob/master/modules/judge/cron/cleaner.go)。
      ```
      func CleanStale() {
        for {
            time.Sleep(time.Hour * 5)
            cleanStale()
        }
      }

      func cleanStale() {
        before := time.Now().Unix() - 3600*24*7

        arr := []string{"0", "1", "2", "3", "4", "5", "6", "7", "8", "9", "a", "b", "c", "d", "e", "f"}
        for i := 0; i < 16; i++ {
            for j := 0; j < 16; j++ {
                store.HistoryBigMap[arr[i]+arr[j]].CleanStale(before)
            }
        }
      }
      ```
    可惜是硬编码的7天~ 把这个改小点儿应该就不用去重启judge module了。当然，如果能同时做到某个告警解除后把相关告警项的current step重置就最好了。

    其实前面都理解错了，哇哈哈~~~  仔细看[代码](https://github.com/open-falcon/falcon-plus/blob/master/modules/judge/store/judge.go)。
    ```
    func sendEventIfNeed(historyData []*model.HistoryData, isTriggered bool, now int64, event *model.Event, maxStep int) {
        if isTriggered {
            ...
        } else {
            // 如果LastEvent是Problem，报OK，否则啥都不做
            if exists && lastEvent.Status[0] == 'P' {
                event.Status = "OK"
                event.CurrentStep = 1
                sendEvent(event)
            }
        }
    }
    ```
    只要有一份正常的数据进来，马上current step就又从1开始计数了，后面告警也就能再次触发了。原来所谓的`告警解除`是指正常数据进来，我之前一直以为需要人工去解除呢。

    还是得先好好研究、吃透代码了再说话。

* Nodata和Graph是紧密相连的

  因为项目用opentsdb作为存储，所以RRD不需要，连带着Graph也不需要，作图可以借助[Grafana](http://grafana.com)或其他的。但是Nodata还是很有用的，可是配置好了之后mock data却一直没上来。从文档里找不到什么有用的信息，只好翻翻源码去了。看了一下发现Nodata会定时通过`/api/v1/graph/lastpoint`接口去拉取你所配置的Nodata的项所对应的最新的3个点的数据，因为我们没有启用Graph，所以也就没数据可返回。于是Nodata认为还没有开始采集数据，所以不做任何处理，自然也就无法产生mock data。

  后面可以再写一篇Nodata module的源码分析。

* 刚发现在[/config/nodata.json](https://github.com/open-falcon/falcon-plus/blob/master/config/nodata.json)和[/config/alarm.json](https://github.com/open-falcon/falcon-plus/blob/master/config/alarm.json)里引用plus api的addr和token没有(全部)使用`%%PLUS_API_HTTP%%`和`%%PLUS_API_DEFAULT_TOKEN%%`这样的placeholder，是疏忽了还是刻意这样写的呢？感觉前者居多，大家用的时候可以改过来。

  不过，突然发现，Nodata用了`%%PLUS_API_HTTP%%`后会出[问题](https://github.com/golang/go/issues/19297)
  ```
  2017/07/10 19:09:00 collector_cron.go:82: fetchItemAndStore fail, size:0, error:parse 127.0.0.1:8080/api/v1/graph/lastpoint: first path segment in URL cannot contain colon
  ```
  请求不到api module了，必须在`%%PLUS_API_HTTP%%`前加上`http://`才可以！

  Alarm module暂时没看到有这个报错，但是保险起见，也加上吧。那要推翻前面的结论了，看来作者是刻意这么写的！

* 改了go文件记得先`make all`或`make module-name`一下，不然pack出来的包应用不上新的代码。但如果只是改了json配置文件，直接pack就行了。