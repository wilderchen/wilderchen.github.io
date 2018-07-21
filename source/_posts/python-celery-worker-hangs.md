---
title: Python Celery Worker Hangs
date: 2018-07-21 16:41:37
tags:
---
项目实践过程中, 我们使用celery(with redis, concurrency=1)作为消息队列, 但是在 worker 运行一段时间之后(about 10+hours), worker就会不再接受处理任务。

## 环境
>* redis 是云服务商提供的服务, 引擎 2.8, 主从配置
* celery worker 运行在云服务商的 k8s 集群的容器上, k8s 版本1.7.8
* python 3.6, celery 4.1.0, kombu 4.1.0, redis 2.10.6

## 现象
>* celery worker运行一段时间之后, 不再处理redis中的任务。先前的容器日志没有太大异常，后续不再有日志输出。
* celery beat正常工作，持续往redis中发送任务
* flower监控显示 worker offline
* redis工作正常, 集群中的其他容器服务正常

## 分析

### celery 浅析
>* celery实现高度依赖于kombu, 在使用redis作为broker/backend的时候, kombu依赖redis实例化
* redis实例化过程中会创建connection pool与redis server通信
* celery启动最后会开启EventLoop循环遍历建立连接的文件描述符, 一旦有读写事件发生则调用相应的代码处理
* EventLoop里面的文件描述符主要由两部分组成:
a) 业务层次相关的, 负责监听任务队列的文件描述符;
b) celery内部维护的基于pubsub实现的worker相互发现文件描述符

### worker 容器现象
>* netstat -natp
![](/images/20180721-182303.png)
* 无后续日志输出

## 猜想
>* 由于某些原因导致redis server把一些connection杀死了
* celery 没有正常处理这些被杀死的connection

## 定位

### 本地配置环境, 多次重启redis-server
>* 多次重启redis-server之后, 本地netstat命令也出现了跟线上环境高度一致的CLOSE_WAIT现象
* worker 正常工作, celery的connection pool在redis-server重启之后新建了connection以继续消费任务
* worker 在redis-server非正常断开连接之后有对应的timeout日志, 与线上日志不符合

### netstat没有显示出已经关闭的连接
>* 在本地redis server执行client list, 把connection依次kill掉
* 在kill pubsub connection的时候会出现worker不能正常工作, 没有后续日志输出的问题
* 导致此问题的原因是, EventLoop里面的主循环在尝试读取一个被kill掉的connection时候block了

## 修复
>* 在worker启动的时候, 增加timeout 参数设置: {'broker_transport_options': {'socket_timeout': 30}
