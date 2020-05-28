---
title: Openvpn 内网互通
date: 2020-05-28 09:28:35
tags:
---

> 先说一个具体的场景：
> 上海内网网段： 192.168.10.0/24
> 北京内网网段： 192.168.1.0/24
> 现在要求上海和北京能够互联互通。
> 常规方法就是整个 vpn（vpn 的安装和基本配置可以看我的另外一篇文章），那具体这么弄？
> 我们一步一步来看看

### 过程思考

1. 先安装 vpn
2. 上海和北京分别准备两台服务器，通过 vpn 客户端连接上来
3. vpn 通过添加路由，服务器通过数据包转发来达到通讯

### 参数解释

1. push
2. route
3. iroute

### 详细配置

- 防火墙(iptables)

```Shell
sudo iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE

sudo iptables -nL -t nat
```

- 数据包转发

```Shell
vi /etc/sysctl.conf

net.ipv4.ip_forward = 1   # 没有则添加，有修改为1（0禁止，1开启）

sysctl -p
```
