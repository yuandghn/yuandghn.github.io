---
layout:     post
title:      "How to get Docker container id in Java"
subtitle:   ""
date:       2019-09-20 17:05:00
author:     "Echo Yuan"
tags:
    - Docker
    - Container
---
其实就是拿一下Docker container的hostname
```java
String hostNameFromInetAddr = InetAddress.getLocalHost().getHostName()
String hostNameFromEnv = System.getenv("HOSTNAME")
```