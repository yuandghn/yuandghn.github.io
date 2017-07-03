---
layout:     post
title:      "Install OpenTSDB"
subtitle:   ""
date:       2017-06-07 18:01:08
author:     "Echo Yuan"
#header-img: "img/in-post/install-opentsdb/opentsdb-logo.png"
tags:
    - OpenTSDB
    - Open-Falcon
---
#### [OpenTSDB](http://opentsdb.net/) - The Scalable Time Series Database
*Store and serve massive amounts of time series data without losing granularity.*

1. Download [https://github.com/OpenTSDB/opentsdb/releases/download/v2.3.0/opentsdb-2.3.0.tar.gz](https://github.com/OpenTSDB/opentsdb/releases/download/v2.3.0/opentsdb-2.3.0.tar.gz)
2. `tar -zxf opentsdb-2.3.0.tar.gz`
3. `./build.sh`

    But you will encounter some errors like below:
    ```
    /data/jdk1.8.0_112/jre/bin/java -cp third_party/javacc/javacc-6.1.2.jar javacc -OUTPUT_DIRECTORY:./src/net/opentsdb/query/expression/parser ../src/parser.jj; echo PWD: `pwd`;
    Error: Could not find or load main class javacc
    PWD: /root/work/opentsdb/opentsdb-2.3.0/build
    ...
    ...
    javac: file not found: ./src/net/opentsdb/query/expression/parser/*.java
    Usage: javac <options> <source files>
    use -help for a list of possible options
    make[1]: *** [.javac-stamp] Error 2
    make[1]: Leaving directory `/root/work/opentsdb/opentsdb-2.3.0/build'
    make: *** [all] Error 2
    ```
    [How to resolve](https://github.com/OpenTSDB/opentsdb/issues/931):
    ```Shell
    rm -rf build
    mkdir build
    cp -r third_party ./build
    ./build.sh
    ```

4. `env COMPRESSION=NONE HBASE_HOME=path/to/hbase-1.X.X ./src/create_table.sh`

    Be sure you have installed HBase and got it running on you local.
5. Copy the `./src/opentsdb.conf` file to `.`, then modify it. Required properties must be set.
    ```
    # --------- NETWORK ----------
    # The TCP port TSD should use for communications
    # *** REQUIRED ***
    tsd.network.port = 4242

    # Where TSD should write it's cache files to
    # *** REQUIRED ***
    tsd.http.cachedir = /Users/.../data/opentsdb/cachedir

    # ----------- HTTP -----------
    # The location of static files for the HTTP GUI interface.
    # *** REQUIRED ***
    tsd.http.staticroot = path-to-opentsdb-2.3.0/build/staticroot

    # A comma separated list of Zookeeper hosts to connect to, with or without port specifiers, default is "localhost".
    # If HBase and Zookeeper are not running on the same machine, specify the host and port here.
    #tsd.storage.hbase.zk_quorum = localhost

    # --------- CORE ----------
    # Whether or not to automatically create UIDs for new metric types, default is False.
    # Recommend setting it as true.
    tsd.core.auto_create_metrics = true
    ```

6. With the config file written, you can start a tsd with the command: `./build/tsdb tsd`
7. Access the TSD's web interface through [http://127.0.0.1:4242](http://127.0.0.1:4242)

    ![opentsdb-web-gui](/img/in-post/install-opentsdb/opentsdb-is-running.png)
8. Running opentsdb as background job if you want to. Logs will go into nohup.out file.
```
nohup ./build/tsdb tsd &
```

If you want to integrate OpenTSDB with Open-Falcon, please turn it on in the Open-Falcon's transfer.json file.
```
"tsdb": {
    "enabled": true,
    "batch": 200,
    "connTimeout": 1000,
    "callTimeout": 5000,
    "maxConns": 32,
    "maxIdle": 32,
    "retry": 3,
    "address": "127.0.0.1:4242"
}
```

