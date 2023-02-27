---
title: Disney
date: 2023-02-27 16:00:00
comments: false
---

## 自我介绍

Hello everyone. My name is LiuQiang. I come from Hegang City, Heilongjiang province. I graduated {% label warning@ /ˈɡrædʒueɪtɪd/ %} from Harbin university, majoring in software engineering {% label warning@ /ˌendʒɪˈnɪrɪŋ/ %}. And graduated in 2016 {% label warning@ tow thousand and sixteen %}. After graduation {% label warning@ /ˌɡrædʒuˈeɪʃ(ə)n/ %}, I came to ShangHai. I have been working for more than six years now, as a Java development engineer. In order to develop better, I went to the Monster Charge in March 2021 {% label warning@ tow thousand and twenty one %}, and it has been until now. I am a cheerful and optimistic person. In my spare time, I like playing basketball and I also like riding bikes. Thank you for this oppotunity for interview. Thank you all !


## 系统架构 - 通用

{% note danger %}
1. 说明作用是什么？比如网关的作用、BFF的作用
2. 使用哪些技术点？为了解决什么问题
3. 抓住重点，原因？为什么？
{% endnote %}

### BFF之前

![20230226235705](https:image.codingoer.top/blog/20230226235705.png)

访问端包括APP、PC、小程序，用户请求依次经过WAF防火墙，Nginx负载均衡，API网关层。然后到达BFF层，BFF层调用各个领域Service完成用户相关操作。

1. 使用Nginx能够对Http做分流策略，承担**高负载**压力，
2. API网关可以对HTTP请求做**限流**，身份验证，配置黑白名单等。根据业务领域划分了多个网关，多个BFF。
3. BFF使用`sentinal`做限流，合并接口请求，针对热点接口做{% label primary@redis缓存 %}，使用{% label primary@多线程 %}的方式调用各个领域微服务完成接口聚合，屏蔽后端实现细节。针对一些耗时接口采用异步的方式提供给前端。通过RPC或者Fegin的方式调用各个service。

API网关关键词：HTTP限流、鉴权
BFF关键词：限流、异步、合并请求、Redis缓存、接口聚合、屏蔽后端细节、接口规范。

### Service

![20230227002407](https:image.codingoer.top/blog/20230227002407.png)

基础服务也就是service负责具体业务实现，服务之间相互依赖相关调用。

1. 各个service之间的调用会设置超时时间和熔断，防止服务崩溃。当发生熔断时会返回一些mock数据。
2. 针对具体业务场景，会通过MQ的方式来实现服务之间的解耦，增加系统的吞吐量。
3. 在service中，针对耗时的接口采用异步的方式，并配合使用多线程来加快服务响应。
4. 针对一些常用配置做JVM缓存或Redis缓存。对节点接口做缓存。
5. 耗时，处理复杂，实时性不高的场景可以使用定时任务、延迟队列、

Service关键词：熔断、解耦、缓存、异步。

### 基础组件

![20230227004630](https:image.codingoer.top/blog/20230227004630.png)

1. MySQL读写分离，针对数据量大的场景做分库分表。核心流程走写库，一般查询走读库。
2. APP、PC列表查询走ES。MySQL达到一定的数据量的时候出现瓶颈

## 系统架构 - Enmonster

{% note danger %}
1. 我负责了那些功能模块？
2. 先概括到具体、从总到细，逻辑要清晰
3. 这些功能点使用了哪些技术？
4. 讲的细致，拖时间
{% endnote %}

各个service之间的调用是通过Fegin来实现的，使用eruka作为注册中心。使用apollo作为配置中心。

### TOB业务

### TOC业务

我主要负责订单中心相关的工作，包括下单服务、订单状态变化、状态变化消息通知、订单查询、价格计算。

1. 下单首先校验定价策略是否有效、验证和价格计算，调用分布式ID服务生成订单ID，采用雪花算法生成订单ID，生成订单。
2. 订单状态

### 性能指标

- 机器数：网关、BFF 10台，service 6台 
- QPS：5000

## 系统架构 - 1haitao

{% note danger %}
1. 我负责了那些功能模块？
2. 先概括到具体、从总到细，逻辑要清晰
3. 这些功能点使用了哪些技术？
{% endnote %}

各个service之间的调用是通过gRPC来实现的，使用zookeeper作为注册中心。采用SpringCloud config作为service的配置中心。

我主要负责爬虫、商品库、购物车、物流、等微服务的开发与维护。

### 爬虫

![20230227010623](https:image.codingoer.top/blog/20230227010623.png)

商品是经过爬虫爬取后入库。爬虫分为前置节点、执行节点、下发节点。

- 下发节点

下发节点负责处理任务逻辑，使用gRPC异步调用执行节点。Redis缓存封装一层。下发节点作为爬虫服务的入口，对爬取后的结果也会做Redis缓存，会设置缓存时间。

- 执行节点

执行节点用来执行任务，获取可执行的前置节点列表，同样通过gRPC异步调用前置节点。执行节点会对前置节点进行动态更新，选择最优的前置节点。

- 前置节点

前置节点使用多线程来爬取商品。前置节点爬取完商品后，对数据进行校验，加工处理。Redis缓存结果

### 商品库

![20230227011413](https:image.codingoer.top/blog/20230227011413.png)

商品经过爬虫之后，写入MySQL。使用canal订阅MySQL binlog，经过RocketMQ的转发同步到ES中。因为MySQL不支持复杂的搜索，在数据量比较大的时候会出现一些瓶颈。
在ES中维护了活动、商品、品牌等数据。支持普通搜索、品牌搜索、聚合搜索、权重搜索等。预留刷索引的接口，可以对全量数据进行刷新，当ES索引结构发生变化时对索引进行重建。

### 性能指标

QPS：平时2000.高峰期5000左右。 平均RT：200ms
服务器：8台  针对活动期准备：扩容机器一倍

## 模拟问答

### 如何达到xxxQPS

1. 前端资源CND静态化、资源预加载，预热
2. 多线程，线程同步，异步化
3. 多级缓存和本地缓存
4. MQ削峰和解耦
5. 熔断、限流、降级
6. 读写分离，分库分表
7. 应用集群部署
8. 系统分层设计，分布式
9. ES搜索

### 应对活动期

根据各个系统的流量进行分析，单台机器可以承载的最大并发量，预估活动时的范围，加机器。

### 缓存

1. 本地缓存与Redis的缓存的区别？
使用Redis缓存有网络开销，Redis是分布式缓存，有并发竞争问题。容易出现缓存雪崩等问题。

2. 缓存与数据库一致性问题？
旁路缓存模式、读写穿透模式、异步缓存写入模式（写缓存加锁）最终一致性和强一致性

3. 缓存雪崩，大量Key失效？
设置随机值

### MQ

1. 消息堆积原因和处理？
2. 消费失败？

## 复杂系统化解

### QQ订阅推送场景

QQ服务了大量的移动互联网用户，订阅提醒功能超大流量。

1. 《使用召唤》手游早上10点发布，提醒预约用户领取礼包。
2. 春节刷一刷领红包，晚上8:05开始，提醒用户订阅参与。

### 分析

https://juejin.cn/post/7197955458961965117


## 高并发系统设计

## 高并发、高可用