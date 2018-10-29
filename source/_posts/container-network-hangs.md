---
title: Container Network Hangs
date: 2018-10-29 11:34:37
categories:
- 技术
tags:
- docker
- network
- celery
---

项目实践过程中，我们用 Celery 解耦邮箱邮件的解读与分析，将其构建成 Docker 镜像以容器的方式运行在公有云的 k8s 的集群中。

## 环境
* python 3.6， celery 4.1.0， kombu 4.1.0， redis 2.10.6
* k8s 版本 1.4.6， Docker version 1.12.6， OS ubuntu 1604

## 现象
* Celery beat 正常工作， redis 任务持续堆积
* Worker 在某个时间点之后不再消费任务队列的任务， 没有任何错误日志输出
* Redis工作正常， 集群中的其他容器服务正常

## 分析

### Google celery worker hangs 

* [Celery worker hangs without any error](https://stackoverflow.com/questions/30272845/celery-worker-hangs-without-any-error)

### netstat -n

* ![](/images/20181029-224100.png)
* 说明: 在容器内执行上述命令， 其中 993 端口是 imap 服务常用连接端口。

### strace -p pid

* 注意: 因为项目运行的容器非特权容器， strace 等命令无法在容器内执行的。需要到该 pod 被调度到的节点上执行。 查找节点上对应的进程， 详情参考 docker inspect。
* 说明: strace结果表明程序阻塞在 "read(22， ..." 上， 22 是文件描述符的标示

### lsof -i

* ![](/images/20181029-230100.png)
* 说明: ID 22 的文件描述符对应打开的连接是 imap 的服务连接。由此推测程序应该是在 read 22 的时候由于没有设置超时时间被阻塞了。此外， 上图中 imap 的服务器为 gmail 的服务器，国内访问存在一定的障碍，可以进一步加强佐证。

## 验证

### tcpdump -i eth0 tcp -w out.pcap

* 容器内执行上述抓包命令， 与此同时执行``` telnet tk-in-f108.1e100.net imaps```
* 把 out.pcap 文件拷贝到本地，用 wireshark 进行分析
* ![](/images/20181029-231800.png)
* 上图表明， 容器与该 google imap server 连接相对不稳定，在握手阶段发生多次重传以及重复的 ack 。进一步加强佐证。

## 简单修复
* 为了不阻塞其余的任务，在 imap 的 socket 连接中加上 timeout 超时时间限制。
