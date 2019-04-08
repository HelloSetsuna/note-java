# RabbitMQ 学习笔记

> 消息队列的优点: `实现异步`, `系统解耦`, `流量削峰`

> 发布订阅模式, 广播通讯, 实现AMQP协议, 消息队列本质是解决通讯问题
Broker: 
VirtualHost: 一个Broker下面可创建多个虚拟机, 提高硬件应用率, 资源隔离(不同虚拟机下可定义同名交换机, 同名队列), 安装时会提供默认虚拟机:`/`
交换机: 解决消息的灵活路由问题, 地址列表, 查找和队列的绑定关系, 将消息分发到符合绑定关系的队列上
队列: 独立运行的进程, 有自己的数据库存储消息, 先进先出 
TCP长连接 包括多个 虚拟连接: 消息信道(Channel) API

直连类型交换机(DIRECT_EXCHANGE): 
binging key(精确的绑定关键字) 
routing key(路由关键字)

主题类型交换机(TOPIC_EXCHANGE): 
binging key(匹配的绑定关键字): # 匹配0个或者多个单词  * 匹配一个单词
routing key(路由关键字)

广播类型的交换机(FANOUT_EXCHANGE): 
无需绑定关键字和路由关键字, 发送消息时所有关联的队列都会收到

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
...


## rabbit-mq 在 spring-boot 中使用
---
### 配置类

### 生产者
AmqpTemplate:
    convertAndSend(exchange, routingKey, content)

### 消费者

### 配置文件
```
rabbitmq.host
rabbitmq.port
rabbitmq.virtual-host
rabbitmq.host
rabbitmq.host
rabbitmq.host
```

## rabbit-mq 消息可靠性投递解决方案

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