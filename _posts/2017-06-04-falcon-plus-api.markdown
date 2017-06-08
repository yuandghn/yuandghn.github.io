---
layout:     post
title:      "Falcon plus api"
subtitle:   ""
date:       2017-06-04 22:18:26
author:     "Echo Yuan"
#header-img: "img/in-post/hello-world/hello-world.gif"
tags:
    - Open-Falcon
---
这个API也蛮折腾的，搞了两三天，总算从坑里跳出来了。

API在这里[http://open-falcon.org/falcon-plus/](http://open-falcon.org/falcon-plus/) 哦，可不是[http://docs.openfalcon.apiary.io](http://docs.openfalcon.apiary.io) 。不明白为什么有两套，[book](https://book.open-falcon.org/zh_0_2/api/index.html)里引用的是错的，而Falcon+的[Github](https://github.com/open-falcon/falcon-plus)的README里引用的是对的。稍不留心就会上错道，然后API怎么调也调不通，
![](/img/cry.png)

如果要查询[/api/v1/graph/history](http://open-falcon.org/falcon-plus/#/graph_histroy)，那么首先要认证，token是放在header里面的
```
key: ApiToken
value: {"sig": "sig_places_here", "name": "your_name”}
```
其中，sig可以通过`/auth/login`接口获得，也可以在服务器间使用`plus_api_default_token`来代替，它是在confgen.sh文件中配置的。
然后post，content type是application/json，内容
```
{
    "start_time": 1495870185,
    "end_time": 1495870767,
    "hostnames": ["xxx-134"],
    "counters": ["bandwitdh.latency/bandwitdh=50M,ip=212.9.111.54"],
    "step": 15,
    "consol_fun": "GAUGE"
}
```

最坑的是counters，它不是代表mertric哦，而是mertric/tags！！！还好自己先看api的log，再在dashboard里点了点才弄明白这回事儿。


