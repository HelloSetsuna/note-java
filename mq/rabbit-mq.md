# RabbitMQ 学习笔记

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
