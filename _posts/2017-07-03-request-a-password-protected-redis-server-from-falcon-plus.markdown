---
layout:     post
title:      "Request a password-protected Redis server from Falcon Plus"
subtitle:   ""
date:       2017-07-03 18:10:46
author:     "Echo Yuan"
#header-img: "img/in-post/hello-world/hello-world.gif"
tags:
    - Open-Falcon
    - Falcon-Plus
    - Redigo
---
在[Falcon+](https://github.com/open-falcon/falcon-plus)的modules中，Judge和Alarm需要与Redis server进行交互，其Data source name的配置是简化的：
```
# in config/confgen.sh file
[%%REDIS%%]="127.0.0.1:6379"

# in config/judge.json file
"alarm": {
    "enabled": true,
    "minInterval": 300,
    "queuePattern": "event:p%v",
    "redis": {
        "dsn": "%%REDIS%%",
        "maxIdle": 5,
        "connTimeout": 5000,
        "readTimeout": 5000,
        "writeTimeout": 5000
    }
}
```
如果Redis server设置了[密码保护](https://redis.io/commands/auth)，那Falcon+就连接不上了。以[Judge module](https://github.com/open-falcon/falcon-plus/blob/master/modules/judge/g/redis.go)的代码为例。
```
dsn := Config().Alarm.Redis.Dsn
maxIdle := Config().Alarm.Redis.MaxIdle
idleTimeout := 240 * time.Second

connTimeout := time.Duration(Config().Alarm.Redis.ConnTimeout) * time.Millisecond
readTimeout := time.Duration(Config().Alarm.Redis.ReadTimeout) * time.Millisecond
writeTimeout := time.Duration(Config().Alarm.Redis.WriteTimeout) * time.Millisecond

RedisConnPool = &redis.Pool{
    MaxIdle: maxIdle,
    IdleTimeout: idleTimeout,
    Dial: func() (redis.Conn, error) {
        c, err := redis.DialTimeout("tcp", dsn, connTimeout, readTimeout, writeTimeout)
        if err != nil {
            return nil, err
        }
        return c, err
    },
    TestOnBorrow: PingRedis,
}
```
其实，Falcon+使用的go-redis-client [Redigo](https://github.com/garyburd/redigo/)本身是支持连接到password-protected Redis server的。以下是代码片段，完整的参考代码在[这里](https://github.com/garyburd/redigo/blob/master/redis/pool.go)。
```
// Use the Dial function to authenticate connections with the AUTH command or
// select a database with the SELECT command:
//
//  pool := &redis.Pool{
//    // Other pool configuration not shown in this example.
//    Dial: func () (redis.Conn, error) {
//      c, err := redis.Dial("tcp", server)
//      if err != nil {
//        return nil, err
//      }
//      if _, err := c.Do("AUTH", password); err != nil {
//        c.Close()
//        return nil, err
//      }
//      if _, err := c.Do("SELECT", db); err != nil {
//        c.Close()
//        return nil, err
//      }
//      return c, nil
//    }
//  }
```
Falcon+将来是否会加入Redis password auth未知，但如果现在你确实有这个需求，那就改一下它的代码来用吧。


