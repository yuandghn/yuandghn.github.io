---
layout:     post
title:      "Agent是如何确保Transfers的高可用性的"
subtitle:   ""
date:       2017-06-04 21:38:26
author:     "Echo Yuan"
#header-img: "img/in-post/hello-world/hello-world.gif"
tags:
    - Open-Falcon
---
QQ讨论组里有人在问
>
agent设置的transfer的地址，如果是多个，那么着多个transfer是负载还是主备？

我觉得严格意义上这不算负载均衡也不是主备，就是HA(High Availability)。

那怎么知道agent是如何从多个transfer addr中选择一个来push数据呢？看falcon-plus/modules/agent/g/transfer.go的源码吧。
![falcon-plus/modules/agent/g/transfer.go](/img/in-post/how-does-the-agent-guarantee-the-high-availability-of-transfers/send-mertrics.png)

rand.Perm(n int)的注释
> Perm returns, as a slice of n ints, a pseudo-random permutation of the integers [0,n) from the default Source.

这是返回一个伪随机的无序数组，数组的各个元素值在区间[0,n)内。比如你设置了3个transfer addr，那返回的数组可能是[1,0,2]也可能是[0,2,1]。然后再依次以数组元素作为下标取到对应的transfer addr，并向它push数据，首次push成功即break掉循环，这样就不会继续向其他的transfer addr push同样的数据了；如果失败则继续尝试其他的transfer addr。

所以transfer的HA是由agent自己来保证的。配置文件里也是要求把transfer addr一个个的列出来。现在假设transfer addr只配置了一个域名，而域名后挂了N个transfer来做负载均衡，那这样能保证transfer的HA吗？很明显不能百分之百保证。因为agent只会push数据到这一个域名，如果正好打到域名后一台有问题的transfer上，数据就会丢失而且agent无能为力即使在还有N-1个transfer可用的情况下。