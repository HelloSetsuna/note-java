# Firewalld 使用笔记

`[--permanent]` 表示可选择在命令后带 `--permanent` 表明命令的效果永久生效, 否则重启后则失效

## 常用命令
---
#### 启用防火墙
``` conf
systemctl start firewalld
```
#### 禁用防火墙
```
systemctl stop firewalld
```
#### 设置开机启动
```
systemctl enable firewalld
```
#### 停止并禁用开机启动
```
sytemctl disable firewalld
```
#### 查看防火墙状态
```
systemctl status firewalld
```
#### 重启防火墙 无需断开连接
```
firewall-cmd --reload
```
#### 重启防火墙 需断开连接
```
firewall-cmd --complete-reload
```
#### 拒绝所有包
```
firewall-cmd --panic-on
```
#### 取消拒绝状态
```
firewall-cmd --panic-off
```
#### 查看是否拒绝
```
firewall-cmd --query-panic
```
#### 将接口添加到区域 默认接口都在 public
```
firewall-cmd --zone=public --add-interface=eth0 [--permanent]
```
#### 查看区域信息
```
firewall-cmd --get-active-zones
```
#### 查看指定接口所属区域信息
```
firewall-cmd --get-zone-of-interface=eth0
```
#### 设置默认接口区域
```
firewall-cmd --set-default-zone=public
```
#### 查看指定区域所有打开的端口
```
firewall-cmd --zone=public --list-ports
```
#### 查看指定区域定义的所有 `rich-rule`
```
firewall-cmd --zone=public --list-rich-rules
```

## 打开端口
---
```
firewall-cmd --zone=public --add-port=80/tcp [--permanent]
```

## 关闭端口
---
```
firewall-cmd --zone=public --remove-port=80/tcp [--permanent]
```

## 限制端口被指定IP访问
---
```
firewall-cmd --add-rich-rule="rule family="ipv4" source address="x.x.x.x" port protocol="tcp" port="4624" accept" [--permanent]
```
> 如果需要限制端口被多个IP访问, 建议添加多条上述规则方便添加修改, 也可如下方式用逗号隔开设置
```
firewall-cmd --add-rich-rule="rule family="ipv4" source address="x.x.x.x1,x.x.x.x2" port protocol="tcp" port="4624" accept" [--permanent]
```
> 如果需要现在一个端口段被指定IP访问, 可采用如下方式, 多个IP与单个端口类似
```
firewall-cmd --add-rich-rule="rule family="ipv4" source address="x.x.x.x" port protocol="tcp" port="9200-9900" accept" [--permanent]
```