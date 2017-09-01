---
layout:     post
title:      "That's you - AdBlock"
subtitle:   ""
date:       2017-08-31 20:00:19
author:     "Echo Yuan"
#header-img: "img/in-post/hello-world/hello-world.gif"
tags:
    - AdBlock
---

最近在用Chrome测试一个管理后台，好奇怪的有些页面是空白的或者请求被中断了，而在Firefox里却没出现这个问题，一开始还以为是页面程序的问题呢，因为还在嵌iframe，逃~

后来发现不是。

从Chrome的`Developer Tools`里看`Network`面板，只能看到这些失败的请求的`Status`是`(failed)`，获取不到啥有用的信息；再去`Console`面板里瞅瞅，发现请求失败的原因是`net::ERR_BLOCKED_BY_CLIENT`。

是哪个`CLIENT`在作祟呢？[Google一下](https://stackoverflow.com/questions/23341765/getting-neterr-blocked-by-client-error-on-some-ajax-calls)发现基本上都是指向`AdBlock`/`AdBlock Plus`这个"罪魁祸首"。

![titter](/img/titter.png)

把被插件block掉的请求的domain exclude一下就好了。

由此也能间接地看出我更喜欢用Firefox来处理工作上的事情，而Chrome多用于生活，因为Firebug用起来更舒服、更习惯。





