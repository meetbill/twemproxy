# twemproxy (nutcracker) [![Build Status](https://secure.travis-ci.org/twitter/twemproxy.png)](http://travis-ci.org/twitter/twemproxy)


<!-- vim-markdown-toc GFM -->

* [Twemproxy 学习](#twemproxy-学习)
* [Build](#build)
* [Features](#features)
* [Help](#help)
* [Zero Copy 零拷贝](#zero-copy-零拷贝)
* [Configuration](#configuration)
* [Observability](#observability)
* [Pipelining](#pipelining)
* [Deployment](#deployment)
* [Packages](#packages)
    * [Ubuntu](#ubuntu)
        * [PPA Stable](#ppa-stable)
        * [PPA Daily](#ppa-daily)
* [Utils](#utils)
* [Companies using Twemproxy in Production](#companies-using-twemproxy-in-production)
* [Issues and Support](#issues-and-support)
* [Committers](#committers)
* [License](#license)

<!-- vim-markdown-toc -->

Nutcracker，又称 Twemproxy（读音："two-em-proxy"）是支持 [memcached](http://www.memcached.org/) 和 [redis](http://redis.io/) 协议的快速、轻量级代理；它的建立旨在减少后端缓存服务器上的连接数量；再结合管道技术（pipelining*）、及分片技术可以横向扩展分布式缓存架构；

## Twemproxy 学习

[学习笔记](https://github.com/meetbill/twemproxy/wiki)

## Build

To build twemproxy from [distribution tarball](https://drive.google.com/open?id=0B6pVMMV5F5dfMUdJV25abllhUWM&authuser=0):

    $ ./configure
    $ make
    $ sudo make install

To build twemproxy from [distribution tarball](https://drive.google.com/open?id=0B6pVMMV5F5dfMUdJV25abllhUWM&authuser=0) in _debug mode_:

    $ CFLAGS="-ggdb3 -O0" ./configure --enable-debug=full
    $ make
    $ sudo make install

To build twemproxy from source with _debug logs enabled_ and _assertions enabled_:

    $ git clone git@github.com:twitter/twemproxy.git
    $ cd twemproxy
    $ autoreconf -fvi
    $ ./configure --enable-debug=full
    $ make
    $ src/nutcracker -h

A quick checklist:

+ Use newer version of gcc (older version of gcc has problems)
+ Use CFLAGS="-O1" ./configure && make
+ Use CFLAGS="-O3 -fno-strict-aliasing" ./configure && make
+ `autoreconf -fvi && ./configure` needs `automake` and `libtool` to be installed

## Features

+ Fast.
+ Lightweight.
+ Maintains persistent server connections.
+ Keeps connection count on the backend caching servers low.
+ Enables pipelining of requests and responses.
+ Supports proxying to multiple servers.
+ Supports multiple server pools simultaneously.
+ Shard data automatically across multiple servers.
+ Implements the complete [memcached ascii](notes/memcache.md) and [redis](notes/redis.md) protocol.
+ Easy configuration of server pools through a YAML file.
+ Supports multiple hashing modes including consistent hashing and distribution.
+ Can be configured to disable nodes on failures.
+ Observability via stats exposed on the stats monitoring port.
+ Works with Linux, *BSD, OS X and SmartOS (Solaris)

## Help

    Usage: nutcracker [-?hVdDt] [-v verbosity level] [-o output file]
                      [-c conf file] [-s stats port] [-a stats addr]
                      [-i stats interval] [-p pid file] [-m mbuf size]

    Options:
      -h, --help             : this help  查看帮助文档，显示命令选项
      -V, --version          : show version and exit  查看 nutcracker 版本
      -t, --test-conf        : test configuration for syntax errors and exit  测试配置文件的正确性
      -d, --daemonize        : run as a daemon  以守护进程运行
      -D, --describe-stats   : print stats description and exit 打印状态描述
      -v, --verbose=N        : set logging level (default: 5, min: 0, max: 11) 设置日志级别
      -o, --output=S         : set logging file (default: stderr) 设置日志输出路径，默认为标准错误输出
      -c, --conf-file=S      : set configuration file (default: conf/nutcracker.yml) 指定配置文件路径
      -s, --stats-port=N     : set stats monitoring port (default: 22222) 设置状态监控端口
      -a, --stats-addr=S     : set stats monitoring ip (default: 0.0.0.0) 设置状态监控 IP
      -i, --stats-interval=N : set stats aggregation interval in msec (default: 30000 msec) 设置状态聚合间隔
      -p, --pid-file=S       : set pid file (default: off) 指定进程 pid 文件路径
      -m, --mbuf-size=N      : set size of mbuf chunk in bytes (default: 16384 bytes) 设置 mbuf 块大小，默认 16K

## Zero Copy 零拷贝

在 twemproxy 中，传入请求和传出响应的所有内存都在 mbuf 中分配。接收请求的 mbuf 同时会用于转发到 backend，类似地，从 backend 接收响应的 mbuf 同时也会用于转发到 client，这样做就避免了内存拷贝。

此外，mbuf 使用内存池，一旦分配就不再释放，当一个请求结束时，它所使用的 mbuf 会放回内存池。一个 mbuf 占 16K，这个大小需要在 I/O 性能和连接并发数之间做取舍，mbuf 尺寸越大，对 socket 的读写系统调用次数越少，但整个系统可支持的并发数也越小。如果希望支持更高的 client 并发请求数，可以把 mbuf 的尺寸设置小一点（通过 -m 选项）。

## Configuration

Twemproxy can be configured through a YAML file specified by the -c or --conf-file command-line argument on process start. The configuration file is used to specify the server pools and the servers within each pool that twemproxy manages. The configuration files parses and understands the following keys:

+ **listen**: The listening address and port (name:port or ip:port) or an absolute path to sock file (e.g. /var/run/nutcracker.sock) for this server pool.
+ **client_connections**: The maximum number of connections allowed from redis clients. Unlimited by default, though OS-imposed limitations will still apply.
+ **hash**: The name of the hash function. Possible values are:
  + one_at_a_time
  + md5
  + crc16
  + crc32 (crc32 implementation compatible with [libmemcached](http://libmemcached.org/))
  + crc32a (correct crc32 implementation as per the spec)
  + fnv1_64
  + fnv1a_64
  + fnv1_32
  + fnv1a_32
  + hsieh
  + murmur
  + jenkins
+ **hash_tag**: hash_tag 允许根据 key 的一个部分来计算 key 的 hash 值。hash_tag 由两个字符组成，一个是 hash_tag 的开始，另外一个是 hash_tag 的结束，在 hash_tag 的开始和结束之间，是将用于计算 key 的 hash 值的部分，计算的结果会用于选择服务器。
  + 例如：如果 hash_tag 被定义为”{}”，那么 key 值为"user:{user1}:ids"和"user:{user1}:tweets"的 hash 值都是基于”user1”，最终会被映射到相同的服务器。而"user:user1:ids"将会使用整个 key 来计算 hash，可能会被映射到不同的服务器。
+ **distribution**: The key distribution mode. Possible values are:
  + ketama 一致性 hash 算法，会根据服务器构造出一个 hash ring，并为 ring 上的节点分配 hash 范围。ketama 的优势在于单个节点添加、删除之后，会最大程度上保持整个群集中缓存的 key 值可以被重用。
  + modula 根据 key 值的 hash 值取模，根据取模的结果选择对应的服务器；
  + random 无论 key 值的 hash 是什么，都随机的选择一个服务器作为 key 值操作的目标；
+ **timeout**: 单位是毫秒，是连接到 server 的超时值。默认是永久等待。
+ **backlog**: 监听 TCP 的 backlog（连接等待队列）的长度，默认是 512。
+ **preconnect**: 是一个 boolean 值，指示 twemproxy 是否应该预连接 pool 中的 server。默认是 false。
+ **redis**: 是一个 boolean 值，用来识别到服务器的通讯协议是 redis 还是 memcached。默认是 false。
+ **redis_auth**: Authenticate to the Redis server on connect.
+ **redis_db**: The DB number to use on the pool servers. Defaults to 0. Note: Twemproxy will always present itself to clients as DB 0.
+ **server_connections**: The maximum number of connections that can be opened to each server. By default, we open at most 1 server connection.
+ **auto_eject_hosts**: 是一个 boolean 值，用于控制 twemproxy 是否应该根据 server 的连接状态重建群集。这个连接状态是由 server_failure_limit 阀值来控制。 默认是 false。
+ **server_retry_timeout**: 单位是毫秒，控制服务器连接的时间间隔，在 auto_eject_host 被设置为 true 的时候产生作用。默认是 30000 毫秒。
+ **server_failure_limit**: 控制连接服务器的次数，在 auto_eject_host 被设置为 true 的时候产生作用。默认是 2。
+ **servers**: 一个 pool 中的服务器的地址、端口和权重的列表，包括一个可选的服务器的名字，如果提供服务器的名字，将会使用它决定 server 的次序，从而提供对应的一致性 hash 的 hash ring。否则，将使用 server 被定义的次序。


For example, the configuration file in [conf/nutcracker.yml](conf/nutcracker.yml), also shown below, configures 5 server pools with names - _alpha_, _beta_, _gamma_, _delta_ and omega. Clients that intend to send requests to one of the 10 servers in pool delta connect to port 22124 on 127.0.0.1. Clients that intend to send request to one of 2 servers in pool omega connect to unix path /tmp/gamma. Requests sent to pool alpha and omega have no timeout and might require timeout functionality to be implemented on the client side. On the other hand, requests sent to pool beta, gamma and delta timeout after 400 msec, 400 msec and 100 msec respectively when no response is received from the server. Of the 5 server pools, only pools alpha, gamma and delta are configured to use server ejection and hence are resilient to server failures. All the 5 server pools use ketama consistent hashing for key distribution with the key hasher for pools alpha, beta, gamma and delta set to fnv1a_64 while that for pool omega set to hsieh. Also only pool beta uses [nodes names](notes/recommendation.md#node-names-for-consistent-hashing) for consistent hashing, while pool alpha, gamma, delta and omega use 'host:port:weight' for consistent hashing. Finally, only pool alpha and beta can speak the redis protocol, while pool gamma, delta and omega speak memcached protocol.

    alpha:
      listen: 127.0.0.1:22121
      hash: fnv1a_64
      distribution: ketama
      auto_eject_hosts: true
      redis: true
      server_retry_timeout: 2000
      server_failure_limit: 1
      servers:
       - 127.0.0.1:6379:1

    beta:
      listen: 127.0.0.1:22122
      hash: fnv1a_64
      hash_tag: "{}"
      distribution: ketama
      auto_eject_hosts: false
      timeout: 400
      redis: true
      servers:
       - 127.0.0.1:6380:1 server1
       - 127.0.0.1:6381:1 server2
       - 127.0.0.1:6382:1 server3
       - 127.0.0.1:6383:1 server4

    gamma:
      listen: 127.0.0.1:22123
      hash: fnv1a_64
      distribution: ketama
      timeout: 400
      backlog: 1024
      preconnect: true
      auto_eject_hosts: true
      server_retry_timeout: 2000
      server_failure_limit: 3
      servers:
       - 127.0.0.1:11212:1
       - 127.0.0.1:11213:1

    delta:
      listen: 127.0.0.1:22124
      hash: fnv1a_64
      distribution: ketama
      timeout: 100
      auto_eject_hosts: true
      server_retry_timeout: 2000
      server_failure_limit: 1
      servers:
       - 127.0.0.1:11214:1
       - 127.0.0.1:11215:1
       - 127.0.0.1:11216:1
       - 127.0.0.1:11217:1
       - 127.0.0.1:11218:1
       - 127.0.0.1:11219:1
       - 127.0.0.1:11220:1
       - 127.0.0.1:11221:1
       - 127.0.0.1:11222:1
       - 127.0.0.1:11223:1

    omega:
      listen: /tmp/gamma 0666
      hash: hsieh
      distribution: ketama
      auto_eject_hosts: false
      servers:
       - 127.0.0.1:11214:100000
       - 127.0.0.1:11215:1

Finally, to make writing a syntactically correct configuration file easier, twemproxy provides a command-line argument -t or --test-conf that can be used to test the YAML configuration file for any syntax error.

## Observability

Observability in twemproxy is through logs and stats.

Twemproxy exposes stats at the granularity of server pool and servers per pool through the stats monitoring port. The stats are essentially JSON formatted key-value pairs, with the keys corresponding to counter names. By default stats are exposed on port 22222 and aggregated every 30 seconds. Both these values can be configured on program start using the -c or --conf-file and -i or --stats-interval command-line arguments respectively. You can print the description of all stats exported by  using the -D or --describe-stats command-line argument.

    $ nutcracker --describe-stats

    pool stats:
      client_eof          "# eof on client connections"
      client_err          "# errors on client connections"
      client_connections  "# active client connections"
      server_ejects       "# times backend server was ejected"
      forward_error       "# times we encountered a forwarding error"
      fragments           "# fragments created from a multi-vector request"

    server stats:
      server_eof          "# eof on server connections"
      server_err          "# errors on server connections"
      server_timedout     "# timeouts on server connections"
      server_connections  "# active server connections"
      requests            "# requests"
      request_bytes       "total request bytes"
      responses           "# responses"
      response_bytes      "total response bytes"
      in_queue            "# requests in incoming queue"
      in_queue_bytes      "current request bytes in incoming queue"
      out_queue           "# requests in outgoing queue"
      out_queue_bytes     "current request bytes in outgoing queue"

Logging in twemproxy is only available when twemproxy is built with logging enabled. By default logs are written to stderr. Twemproxy can also be configured to write logs to a specific file through the -o or --output command-line argument. On a running twemproxy, we can turn log levels up and down by sending it SIGTTIN and SIGTTOU signals respectively and reopen log files by sending it SIGHUP signal.

## Pipelining

Twemproxy 可以同时接收很多 client 端的请求，并仅通过一个或几个连接回源，这种结构很适合使用流水线处理请求和响应，从而节省 TCP 往返时间。
例如，Twemproxy 正在同时代理 3 个 client 端的请求，分别是：'get key\r\n'、'set key 0 0 3\r\nval\r\n'和'delete key\r\n' '，Twemproxy 可以将这 3 个请求打包成一个消息发送给后端的 redis： 'get key\r\nset key 0 0 3\r\nval\r\ndelete key\r\n'。

pipelining 也是 Twemproxy 高性能的原因之一。

## Deployment

If you are deploying twemproxy in production, you might consider reading through the [recommendation document](notes/recommendation.md) to understand the parameters you could tune in twemproxy to run it efficiently in the production environment.

## Packages

### Ubuntu

#### PPA Stable

https://launchpad.net/~twemproxy/+archive/ubuntu/stable

#### PPA Daily

https://launchpad.net/~twemproxy/+archive/ubuntu/daily

## Utils
+ [collectd-plugin](https://github.com/bewie/collectd-twemproxy)
+ [munin-plugin](https://github.com/eveiga/contrib/tree/nutcracker/plugins/nutcracker)
+ [twemproxy-ganglia-module](https://github.com/ganglia/gmond_python_modules/tree/master/twemproxy)
+ [nagios checks](https://github.com/wanelo/nagios-checks/blob/master/check_twemproxy)
+ [circonus](https://github.com/wanelo-chef/nad-checks/blob/master/recipes/twemproxy.rb)
+ [puppet module](https://github.com/wuakitv/puppet-twemproxy)
+ [nutcracker-web](https://github.com/kontera-technologies/nutcracker-web)
+ [redis-twemproxy agent](https://github.com/Stono/redis-twemproxy-agent)
+ [sensu-metrics](https://github.com/sensu-plugins/sensu-plugins-twemproxy/blob/master/bin/metrics-twemproxy.rb)
+ [redis-mgr](https://github.com/idning/redis-mgr)
+ [smitty for twemproxy failover](https://github.com/areina/smitty)
+ [Beholder, a Python agent for twemproxy failover](https://github.com/Serekh/beholder)
+ [chef cookbook](https://supermarket.getchef.com/cookbooks/twemproxy)
+ [twemsentinel] (https://github.com/yak0/twemsentinel)

## Companies using Twemproxy in Production
+ [Twitter](https://twitter.com/)
+ [Wikimedia](https://www.wikimedia.org/)
+ [Pinterest](http://pinterest.com/)
+ [Snapchat](http://www.snapchat.com/)
+ [Flickr](https://www.flickr.com)
+ [Yahoo!](https://www.yahoo.com)
+ [Tumblr](https://www.tumblr.com/)
+ [Vine](http://vine.co/)
+ [Wayfair](http://www.wayfair.com/)
+ [Kiip](http://www.kiip.me/)
+ [Wuaki.tv](https://wuaki.tv/)
+ [Wanelo](http://wanelo.com/)
+ [Kontera](http://kontera.com/)
+ [Bright](http://www.bright.com/)
+ [56.com](http://www.56.com/)
+ [Digg](http://digg.com/)
+ [Gawkermedia](http://advertising.gawker.com/)
+ [3scale.net](http://3scale.net)
+ [Ooyala](http://www.ooyala.com)
+ [Twitch](http://twitch.tv)
+ [Socrata](http://www.socrata.com/)
+ [Hootsuite](http://hootsuite.com/)
+ [Trivago](http://www.trivago.com/)
+ [Machinezone](http://www.machinezone.com)
+ [Path](https://path.com)
+ [AOL](http://engineering.aol.com/)
+ [Soysuper](https://soysuper.com/)
+ [Vinted](http://vinted.com/)
+ [Poshmark](https://poshmark.com/)
+ [FanDuel](https://www.fanduel.com/)
+ [Bloomreach](http://bloomreach.com/)
+ [Hootsuite](https://hootsuite.com)
+ [Tradesy](https://www.tradesy.com/)
+ [Uber](http://uber.com) ([details](http://highscalability.com/blog/2015/9/14/how-uber-scales-their-real-time-market-platform.html))
+ [Greta](https://greta.io/)

## Issues and Support

Have a bug or a question? Please create an issue here on GitHub!

https://github.com/twitter/twemproxy/issues

## Committers

* Manju Rajashekhar ([@manju](https://twitter.com/manju))
* Lin Yang ([@idning](https://github.com/idning))

Thank you to all of our [contributors](https://github.com/twitter/twemproxy/graphs/contributors)!

## License

Copyright 2012 Twitter, Inc.

Licensed under the Apache License, Version 2.0: http://www.apache.org/licenses/LICENSE-2.0
