---
title: General Cache Layer Base On Kong
date: 2018-12-22 14:41:18
categories:
- 技术
tags:
- cache
- kong
- ingress
- api gateway
- kubernetes

---

* 在接 RESTful api 接口开发实践过程中, 我们常常需要借助一些缓存策略去提高接口的响应速度以提升用户体验。此外，恰当的缓存策略也有利于降低后方数据库承载的压力。当然，缓存维护也是需要一定成本的，在数据更新的时候我们也需要同时使得缓存更新／失效。在设计开发缓存层的时候，我们应尽量减少对业务层的侵入，设计尽可能通用的缓存层。

* 本示例将基于 Kong CE (https://konghq.com/) 从原理上改造设计一个业务通用的 L7 缓存插件, 参考源于 kong-plugin-response-cache (https://github.com/wshirey/kong-plugin-response-cache) 。

* 注意⚠️: Kong EE 版本是带 Proxy Cache 的, 具体实现方法不详, 有需要的可以自行体验。

## 系统结构
* ![](/images/20181222-144100.png)

* 说明: 在系统结构图中我们可以看到，由于 Kong (proxy) 的介入，我们可以把通用服务(如: 缓存/认证/日志等)与业务服务剥离开, 使得服务职责/功能趋向单一化。我们即将改造设计的缓存层也是以插件的形式运行在代理层的。

## 缓存层考量点

### 缓存内容
* cache key:

>
1. HTTP METHOD (GET/POST...)
2. HTTP URI (/api)
2. HTTP HEADER (Host...)
3. QUERY STRING (page=xxx)
4. HTTP BODY ({"name": "wilder"})

* cache value: http response

* 说明: 按需组织所需要的 cache key (如: HTTP_MEHTOD:HTTP_URI:QS_KEY1=QS_KEY2)并将对应的http response存入到 redis 缓存中。注意⚠️要对cache_key进行排序,不然可能会导致大量缓存击穿以及缓存层被快速占满。 

### 失效触发机制
* 在 kong plugin schema 中定义相关的标志位, 如果该请求满足对应的 "REST" 条件(如: HTTP_METHOD in [POST/DELETE/PATCH/PUT] AND URI = "/api") 则触发对应的失效机制

* 失效方案设计成在 redis 中删除符合特定 PATTERN 条件的 keys, 该 PATTERN 需要在 plugin schema 中定义

* 在写入缓存的时候, 设置对应的 expire time，超时自动失效

## 部署方案(Base On Kubernetes)

### kong (kong proxy)
* 以 DaemonSet/Deployment 的方式将 kong proxy 部署到集群的每个节点上。 在 kong proxy 上层挂在指定端口的 NodePort，将该 Service 挂载到 Load Balancer 后端

* 注意⚠️: 如果需要用 Deployment 在每个节点部署一个／一个以上的 Pod，应该需要自定义调度策略的(这里可以思考一下，有什么场景需要在每个节点上都部署一个 proxy)。

### kong admin
* 以 Deployment 方式部署 vpc/cluster 内访问的 kong admin 即可, 该 admin 接口提供管理服务/路由／插件等能力。

* 注意⚠️: 除非本地开发环境，该 admin 接口万不能部署到公网环境上

### kong admin gui
* 以 Deployment 的方式部署 vpc/cluster 内访问的 pantsel/konga (kong admin gui), 该服务以图形化的形式提供 kong admin  接口管理能力。

* 注意⚠️: 目前版本(DATE: 2018.12.22)只支持 0.14.x 版本的 kong admin, 1.0.0 版本的 kong 有太多的 break changes(如: service schema changed), 所以就目前情况而言，社区版的 GUI 暂不支持 1.0.0 的 kong, 如需要 1.0 的 admin gui, 可以尝试使用 Kong EE

## 效果
### 缓存击穿效果
* ![](/images/20181222-163500.png)

* 说明: 如上所示，缓存击穿的时候首次请求耗时为 5054 ms

### 缓存命中效果
* ![](/images/20181222-163600.png)

* 说明: 如上所示，缓存命中的时候首次请求耗时为 51 ms

### 关联接口导致缓存失效说明
* 该通用缓存的设计可以达到用户在调用特定 api (如: POST /api/xxx) 的时候使得之前产生特定 pattern 的缓存的 key 批量失效。 下图为失效后再次命中缓存的例子, 请注意此次缓存命中中标红的字段 "total" 已经是 23 了:

* ![](/images/20181222-164300.png)


