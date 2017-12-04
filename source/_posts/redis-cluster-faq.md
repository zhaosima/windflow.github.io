---
title: redis cluster faq
date: 2017-11-15 17:26:33
tags:
---

## Redis cluster FAQ

### create


#### ruby update
````
curl -L get.rvm.io | bash -s stable
rvm install 2.3.0
rvm use 2.3.0 --default

ruby --version
````

#### gems
````
yum install rubygems
gem install redis
````

#### create cluster
````
redis-trib.rb  create --replicas 1 172.131.120.116:6379  172.131.120.116:6380 172.131.120.116:6381  172.131.120.117:6379 172.131.120.117:6380 172.131.120.117:6381
````

#### traps

* 如果端口是6379，那么cluster的默认通信端口是+10000，需要在iptables加上6379和16379的访问权限

* 如果报已经有key在使用，请flushall, 删除每个node的rdb,nodes.conf,aof，然后启动。

* 如果报slot已占用，请在每个node运行cluster reset hard.

