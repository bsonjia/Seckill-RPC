# SecKill-RPC

## 简介

在线秒杀 web 应用程序——升级版

用户可以在秒杀开启后进行秒杀活动；秒杀结束，阻止用户秒杀；用户成功秒杀到商品后阻止其二次秒杀；

演示：

![AJuvXd.gif](https://s2.ax1x.com/2019/03/23/AJuvXd.gif)

## 特性

项目分为【前端】和【后端】

前端处理与用户打交道的工程，后端专门处理数据交互

由于是对之前的 [Seckill-Monolithic-Architecure](https://github.com/ittqqzz/Seckill-Monolithic-Architecure) 项目进行升级，所以本项目由三个模块组成：

1. seckill 也就是之前的版本，虽然进行了大量修改，但还是一个经典的 SSM 结构
2. seckillback 不关心与用户的交互，专心处理数据的模块
3. seckillapi 为seckill 以及 seckillback 提供公共类或接口

当上游数据顺利到达下游后，seckill 会将消息发送到 MQ，seckillback 处理队列消息，进行创建订单与扣减库存，所以需要使用到 RabbitMQ 提供消息中间件的服务，以及 Dubbo 提供 RPC 调用服务

运行流程图：

![kiZKFe.png](https://s2.ax1x.com/2019/01/21/kiZKFe.png)

项目整体结构图：

![kiZMJH.png](https://s2.ax1x.com/2019/01/21/kiZMJH.png)

为确保消息 100% 落地，额外添加 brokerDB ，通过定时检查该库 status 字段为非 1 的数据行并进行补偿操作，以避免后端消费 MQ 失败。

## 优化思路

* 秒杀业务分析

  * 特点：瞬时并发量大、库存少、业务流程简单
  * 技术难点：冲击现有的业务、限制 URL 接口开放、防止黑产
  * **主要考虑将货卖完**，所以主要策略是过滤掉 90% 的用户，让剩下的 10% 竞争一次，最终将货物卖完

* 核心优化思路：将请求拦截在上游，降低下游的压力

  * 限流
  * 缓存
  * 削峰
  * 异步

* 前端

  * 前端静态资源使用 CDN 
  * 用 js 限制请求接口的时间与频率
  * 点击秒杀后立刻禁用按钮 

* 后端

  * 使用 Redis 缓存热点数据

  * 限制请求透过率，防止 for 循环攻击，例如，对同一个用户，每 5 秒透过一个请求

  * 使用消息队列，异步处理秒杀请求，为了进一步降低 DB 压力，做一个容量固定的队列，超过队列容量的请求直接丢弃并通知用户秒杀失败，消息被成功消费后通知用户秒杀成功并在多少时间内付款。

    （所以用户秒杀后，要么立即失败，要么异步通知用户秒杀结果，就像购买火车票一样，飞猪 APP 会说以最终短信通知为准，短信通知购票成功就是成功了，短信通知失败那就得再来一次）

  * 后端服务器应该独立，不要和其他业务共用，导致影响其他业务的正常运行，使用分布式部署方式，将【前端】与【后端】分别部署到不同的机器上，可以进一步缓解后端压力

    * 前端请求并投递消息，后端慢慢消费消息并通知用户处理结果（就好比买电影票，购票结果将在稍后通知）

* 上游优化的好的话，mysql 单机应该可以应付过来的把。。**:see_no_evil:**

## Q&A

* 怎么控制用户访问秒杀连接？
    * 第一次访问，请求的是接口暴露服务，如果秒杀开始了就返回执行秒杀的 URL，如果秒杀未开始或者结束了，返回：秒杀未开始 / 已结束的提示信息
    * 第二次访问，此时在访问的要么是执行秒杀的 URL，要么是请求暴露接口，请求暴露接口此时无需再访问后端了，因为第一次访问的时候会把秒杀的时间返回给前端缓存；要么是访问的是执行秒杀，还是由前端判断，是否在第一次得到了暴露的接口，如果得到了，就执行秒杀，当然后端还会再判断时间是否真的到达（因为前端再怎么校验都会被破解） 
* 超卖怎么解决？商品卖完了，但是没有达到预计结束时间如何处理？
    * 超卖的解决：
        * 控制最后进入消息队列的请求数量 <= 商品数量
        * 修改 update 语句：`
          UPDATE seckill SET number = number - 1 WHERE number >= 1 AND seckill_id = 1003`
        * 设定一个原子计数器，在项目上线时就为计数器初始化，比如针对 id 为 1003 的商品，他的库存为 1000，则该原子计数器的初始值为 1000，每次秒杀 1003 商品的消息投递成功时就将该计数器减一，直到为 0，然后淘汰接下来的用户并关闭此商品接口

    * 如何在前端提前结束秒杀
        * 当第一次遇到商品结束的消息后就缓存此消息到 Redis，同时关闭执行秒杀的接口，并返回前端商品提前销售结束的消息，前端立刻改变样式。    
* 哪里会出现并发？怎么优化并发？
    1. 可能会出现并发的地方：
        1. 商品详情页
        2. 访问服务端系统时间
        3. 服务端返回暴露接口
        4. 服务端执行秒杀操作
    2. 【首先】获取系统时间无需优化，就算并发特厉害也无需优化，Java 获取系统时间的速度是微妙级别的
    3. 【然后】获取页面需要优化，将页面资源放到 CDN 上，同时控制按钮不允许多次点击，页面数据缓存到 Redis 里面，开启超时策略与双删一致性方案
    4. 【其他】秒杀地址接口需要优化，应该放到 Redis 里面
        1. 对于每一个商品，他的秒杀地址是限时开放的，存到 CDN 上不合适，同时他的地址又是固定的，要是每来一个请求都计算一遍地址，对系统的压力太大，可以将第一次开放得到的秒杀地址存到 Redis 里面
        2. 所以秒杀地址的优化过程为：`请求地址--->Redis(存在则返回)--->数据库`
    5. 优化要分为前、后
        1. **前端（将请求尽量拦截在系统上游）**
            1. 按钮点击一次后变灰
            2. js 控制用户几秒内可以再次发送请求（为什么说再次发送请求，可能他抢失败了，允许再抢一次），如果说秒杀成功，则严禁二次秒杀
        2. **后端（不要让请求落到数据库上去）**
            1. 一般都要先登录再秒杀，所以每个用户都有 ID，对同一个用户 ID，每 5 秒允许通过一个请求（可通过前端 js 控制）（后端通过限流器：RateLimiter 控制连接数，对频繁访问的 IP 实施封禁）
            2. 使用消息队列（MQ），设置队列长度，只卖 1 万台手机，就只允许队列里面存在 1 万个消息。多余的抛弃，返回重试/售罄。
                【或者放进去 2 万个请求道后端，然后后端开始一新的竞争，但这个时候后端压力只是 2 万级别的，并不会出现问题。】
            3. 热点数据 Redis 缓存起来，例如：一旦秒杀商品放到网页上了，那么它的秒杀开始时间、结束时间、价格等信息都是不会变更了的，用户浏览这些信息的时候用 Redis 缓存起来最好。
            4. （分时间段秒杀，类似于12306分时段放票。这个方案总觉得不太行。。至少是一种思路吧）
            5. 业务逻辑的异步：例如下单业务与支付业务的分离
            6. 成功进入 MQ 的请求先创建订单并计时，然后扣减库存，若超时未支付就丢弃订单并恢复库存。                                    
            7. 对与数据库：
                1. 只要上游过滤了，数据库单机处理应该没问题的，毕竟货不多。。
                2. 不然再分库分表
            8. 架构图（上面已经展示了）


* 一个用户不可以重复秒杀成功
    * 解决的办法是同一个用户 ID 不可以往成功秒杀的表里面插入两条记录，否则报错，整个事务回滚。 

* 系统并发时为何低效：
  其实 MySQL不低效，对同一行数据做 update，每秒可以执行约 4W 次。
  Java 的执行也不低效。
  其实低效的是整个事务流程，即网络延迟和 GC 可能会导致系统瓶颈。
  并发越大 GC 越频繁。
  还可能存在请求来的太快，每一个请求占用的数据库行级锁时间长，导致其他请求必须排队。
  * 优化办法：
    1. 减少数据库行级锁持有时间，可以提高事务处理速度。【根据两段锁协议：热点操作放应到离 commit 近一点的地方。非必须操作异步执行】
    2. 把客户端逻辑放到数据库服务端，可以避免网络延迟和 GC 的影响，具体方法是使用存储过程。
  * 存储过程：
    存储过程可以优化事务行级锁持有的时间。
    存储过程大量应用在银行，在互联网公司没有什么市场，不要过度依赖存储过程。
    存储过程可以以用于简单的逻辑，复杂的不要用。但本业务又比较特殊，可以使用存储过程。

* Maven 循环依赖的问题，通过将共用代码抽取到公共模块 C，A、B 之间不再互相依赖，均引用 C 模块即可

* 解决 dubbo 中遇到 HessianProtocolException: 'xxxException' could not be instantiated 的问题： [点击CSDN连接](https://blog.csdn.net/zipo/article/details/81106604)

## 技术选型

后端：

* 数据库：Redis、MySQL
* 持久层框架：Mybatis
* Web 框架：Spring MVC
* 容器：Spring Boot
* 服务器：Tomcat
* 分布式服务框架：Dubbo
* 注册中心：Zookeeper
* 消息中间件：RabbitMQ
* 依赖管理：Maven
* 版本控制：Git
* 工具：lombok、Protostuff、Guava、Logback
* 运行环境：JDK 8
* 开发工具：IDEA

前端：

* Bootstrap
* jQuery
* jQuery.countdown.min



## 安装

用 IDEA open seckill 工程，然后按下 ctrl + alt + shift + s 打开 Project Structure ，点击 Modules，点击 “+” 号，再点击 import Module，将剩下的两个模块导入

在 sql 包下，执行 seckill.sql 与 seckillback.sql，建立数据库，然后在 seckill 模块下的 jdbc.properties 文件里面修改前端要用到的数据库连接参数，以及 seckillback 模块下的 application.properties 文件里面修改后端用到的数据库

启动 Redis ，默认 host： 127.0.0.1，port ：6379；如需修改请在 spring-dao.xml 文件里面修改

启动 RabbitMQ，使用默认参数

启动 Zookeeper，使用默认参数

项目启动过程：

1. 直接启动 seckillback 模块里面的 SeckillmqreceiveApplication
2. 再为 seckill 模块添加 Tomcat，部署项目并启动
3. 最后访问连接：http://localhost:8080/seckill/list



