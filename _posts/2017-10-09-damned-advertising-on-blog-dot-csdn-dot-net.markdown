---
layout:     post
title:      "Damned advertising on blog.csdn.net"
subtitle:   ""
date:       2017-10-09 14:20:50
author:     "Echo Yuan"
#header-img: "img/in-post/hello-world/hello-world.gif"
tags:
    - Advertising
---
有时候会在blog.csdn.net上看些文章，尽管我的Chrome里已经安装了AdBlock和ABP，但仍难以完全屏蔽页面广告的骚扰。

![damned-advertising](/img/in-post/damned-advertising-on-blog-dot-csdn-dot-net/damned-advertising.png)

最讨厌那些没有关闭按钮的广告或者点击了关闭按钮却毫无反应的广告。感叹一句，趋利也不能趋成这样呀~

刚开始还忍了一段时间，后来实在不堪忍受不断轮播的广告来转移和影响我的视线了。

试过用AdBlock单独block某个广告位置，当时是生效的，可是一刷新页面就又回来了，难为csdn如此攻防了。

这就没办法了吗？

某天突然灵光一现，广告的轮播效果一般都是用JavaScript来实现的吧，那我禁止JS脚本在blog.csdn.net上运行不就可以了? 嘎嘎~  好像是个办法吖！

我想，牛X的Chrome应该支持在指定域名上禁用JS脚本，这样就不会误伤其他网站了。

打开Settings找一找，果然如此！

![damned-advertising](/img/in-post/damned-advertising-on-blog-dot-csdn-dot-net/block-javascript-on-a-site.png)

刷一下页面，清爽世界！



