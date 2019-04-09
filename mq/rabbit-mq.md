# RabbitMQ 学习笔记

> 消息队列的优点: `实现异步`, `系统解耦`, `流量削峰`

> 发布订阅模式, 广播通讯, 实现AMQP协议, 消息队列本质是解决通讯问题


## rabbit-mq 安装
---
> 在阿里云 CentOS 7.3 系统中安装
### 1. 安装 erlang
```
yum install erlang
```

### 2. 下载 安装包
> 可自行查找下载其它的版本
```
wget http://www.rabbitmq.com/releases/rabbitmq-server/v3.6.6/rabbitmq-server-3.6.6-1.el7.noarch.rpm
```
### 3. 安装 安装包
```
yum install rabbitmq-server-3.6.6-1.el7.noarch.rpm 
```
## rabbit-mq 命令
---
### 启动
```
service rabbitmq-server start
```
### 停止
```
service rabbitmq-server stop
```
### 查看状态
```
service rabbitmq-server status
```

### 启用Web端管理插件
```
rabbitmq-plugins enable rabbitmq_management
```

### 添加用户 setsuna
```
rabbitmqctl add_user setsuna password 
```

### 修改角色
> 一般建议在设置了管理员角色后, 到Web管理平台删除 guest 账户
```
rabbitmqctl set_user_tags setsuna administrator
```

### 修改权限
> 赋予用户 对 virtual hosts : /vhost 的所有权限, 可在Web管理平台更方便的设置
```
rabbitmqctl set_permissions -p /vhost setsuna '.*' '.*''.*'
```

## rabbit-mq 角色
---
### none 
> 不能访问 Web端管理平台, 一般的消费者和生产者使用此角色

### management
> 用户可以通过 AMQP 做的任何事外加：
* 列出自己可以通过AMQP登入的 virtual hosts  
* 查看自己的 virtualhosts 中的 queues, exchanges 和 bindings
* 查看和关闭自己的 channels 和 connections
* 查看有关自己的 virtual hosts 的“全局”的统计信息，包含其他用户在这些 virtual hosts 中的活动。

### policymaker 
> management 可以做的任何事外加：
* 查看、创建和删除自己的virtualhosts所属的policies和parameters

### monitoring
> management 可以做的任何事外加：
* 列出所有 virtualhosts，包括他们不能登录的 virtualhosts
* 查看其他用户的 connections 和 channels
* 查看节点级别的数据如 clustering 和 memory 使用情况
* 查看真正的关于所有 virtualhosts 的全局的统计信息

### administrator   
> policymaker 和 monitoring 可以做的任何事外加:
* 创建和删除 virtualhosts
* 查看、创建和删除 users
* 查看创建和删除 permissions
* 关闭其他用户的 connections

## rabbit-mq 配置
---
> 一般来说默认的配置文件路径如下 /etc/rabbitmq/rabbitmq.config 默认没有这个文件, 需自己创建, 更多信息可查看官网的 [配置说明](https://www.rabbitmq.com/configure.html)
```
vi /etc/rabbitmq/rabbitmq.config
```

### 修改服务的默认端口号
```
[
  {rabbit, [
      {tcp_listeners, [5673]}
    ]
  }
].
```

## rabbit-mq 理解
---
### 概念理解
![基本架构图](./assets/rabbit-mq-1.png)

* Exchange：消息交换机，指定消息按什么规则，路由到哪个队列。

* Queue：消息队列，每个消息都会被投入到一个或者多个队列里, 独立运行的进程, 有自己的数据库存储消息, 先进先出。

* Binding：绑定，它的作用是把 exchange 和 queue 按照路由规则 binding 起来。

* Routing Key：路由关键字，exchange根据这个关键字进行消息投递。

* Virtual Host: 虚拟主机，一个broker里可以开设多个vhost，用作不用用户的权限分离。每个virtual host本质上都是一个RabbitMQ Server（但是一个server中可以有多个virtual host），拥有它自己若干的个Exchange、Queue和bings rule等等。其实这是一个虚拟概念，类似于权限控制组。Virtual Host是权限控制的最小粒度, 提高硬件应用率, 资源隔离(不同虚拟机下可定义同名交换机, 同名队列), 安装时会提供默认虚拟机:`/`。

* Producer：消息生产者，就是投递消息的程序。

* Consumer：消息消费者，就是接受消息的程序。

* Connection: 就是一个TCP的连接。Producer和Consumer都是通过TCP连接到RabbitMQ Server的。接下来的实践案例中我们就可以看到，producer和consumer与exchange的通信的前提是先建立TCP连接。仅仅创建了TCP连接，producer和consumer与exchange还是不能通信的。我们还需要为每一个Connection创建Channel。

* Channel: 消息信道, 它是建立在上述TCP连接之上的虚拟连接。数据传输都是在Channel中进行的。AMQP协议规定只有通过Channel才能执行AMQP的命令。一个Connection可以包含多个Channel。有人要问了，为什么要使用Channel呢，直接用TCP连接不就好了么？对于一个消息服务器来说，它的任务是处理海量的消息，当有很多线程需要从RabbitMQ中消费消息或者生产消息，那么必须建立很多个connection，也就是许多个TCP连接。然而对于操作系统而言，建立和关闭TCP连接是非常昂贵的开销，而且TCP的连接数也有限制，频繁的建立关闭TCP连接对于系统的性能有很大的影响，如果遇到高峰，性能瓶颈也随之显现。RabbitMQ采用类似NIO的做法，选择TCP连接复用，不仅可以减少性能开销，同时也便于管理。在TCP连接中建立Channel是没有上述代价的，可以复用TCP连接。对于Producer或者Consumer来说，可以并发的使用多个Channel进行Publish或者Receive。有实验表明，在Channel中，1秒可以Publish10K的数据包。对于普通的Consumer或者Producer来说，这已经足够了。除非有非常大的流量时，一个connection可能会产生性能瓶颈，此时就需要开辟多个connection。


### 交换机说明

#### 直连类型交换机(DIRECT_EXCHANGE): 
![直连类型交换机](./assets/rabbit-mq-2.png)
* binging key(精确的绑定关键字) 
* routing key(路由关键字)

#### 主题类型交换机(TOPIC_EXCHANGE): 
![主题类型交换机](./assets/rabbit-mq-3.png)
* binging key(匹配的绑定关键字): `#` 匹配0个或者多个单词  `*` 匹配一个单词
* routing key(路由关键字)

#### 广播类型的交换机(FANOUT_EXCHANGE): 
* 无需绑定关键字和路由关键字, 发送消息时所有关联的队列都会收到


## rabbit-mq 在 spring-boot 中使用
---
### Maven 依赖
```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```
### 配置文件
```
rabbitmq.host
rabbitmq.port
rabbitmq.virtual-host
rabbitmq.host
rabbitmq.host
rabbitmq.host
```

### 配置类

### 生产者
AmqpTemplate:
    convertAndSend(exchange, routingKey, content)

### 消费者



## rabbit-mq 消息可靠性投递解决方案
---
### 确保消息成功发送到RabbitMQ服务器

### 事务模式
```
try {
    txSelect
    txPublish
    txCommit
} catch (e) {
    txRollback
}
```

### 确认模式
#### 简单确认模式
#### 批量确认模式
#### 异步确认模式

### 确保消息路由到正确的队列
转发给自己 ReturnListener
指定交换机的备份交换机

###
#### 队列持久化
#### 交换机持久化
#### 消息持久化

高可用
集群

### 确保消息从队列正确的投递到消费者
---
#### 自动ACK
消费者接收到消息时就立即返回ACK, 无论业务方法是否执行完
#### 手动ACK
```
try {

} finally {
    // 防止消息在队列里堆积
    channel.basicAck();
}
```

### 生产者怎么知道消费者是否正确消费了消息
#### 消费者回调
* 生产者发送消息时提供回调接口, 消费者接收到消息处理后调用回调API
* 消费者接收到消息后发送应达消息
#### 补偿机制
* 消息重发(重发消息是否与原消息一致? 控制次数(计数器)? 根据流水日志进行对照)
#### 消息幂等性
* 消费者处理消息时的逻辑具有幂等性(流水号 bizId 唯一的) 处理前根据bizId查重