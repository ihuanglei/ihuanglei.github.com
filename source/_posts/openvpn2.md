---
title: Openvpn 内网互通
date: 2020-05-28 09:28:35
tags:
categories: vpn
---

> 先说一个具体的场景：
> 上海内网网段： 192.168.10.0/24
> 北京内网网段： 192.168.1.0/24
> 现在要求上海和北京能够互联互通。
> 常规方法就是整个 vpn（vpn 的安装和基本配置可以看我的另外一篇文章），那具体这么弄？
> 我们一步步来看

## 第一种方式

我们把 vpn 服务安装在上海或北京的服务器上，我们这里拿上海做例子

- 前提条件

上海必须有一个公网固定 IP，比如 123.123.123.123

- vpn 安装及配置

我们先把 vpn 服务安装到上海的某台服务器上(安装过程可以查找我的另外一篇文章)

然后修改配置文件，注释删掉后大概是这个样子

```
;local a.b.c.d # 如果有多网卡可以指定一个

port 1194 # 端口号
proto tcp # 协议
dev tun   # 路由模式

# 证书位置
ca /etc/openvpn/server/certs/ca.crt
cert /etc/openvpn/server/certs/server.crt
key /etc/openvpn/server/certs/server.key
dh /etc/openvpn/server/certs/dh.pem

# 客户端地址池
server 10.8.0.0 255.255.255.0
ifconfig-pool-persist /etc/openvpn/ipp.txt

# 客户端添加一个路由
push "route 192.168.10.0 255.255.255.0"

client-config-dir /etc/openvpn/ccd
client-to-client
keepalive 10 120
cipher AES-256-CBC
comp-lzo
user openvpn
group openvpn
persist-key
persist-tun
status openvpn-status.log
log-append  openvpn.log
verb 3
mute 20
```

这里要解释一下

```
# 客户端添加一个路由
push "route 192.168.10.0 255.255.255.0"
```

**意思就是：vpn客户端只要访问 192.168.10.0/24 的 IP，从我们 vpn 拨号的这个网口发出去**

试想一下北京某台电脑 IP 为 192.168.1.6 的电脑，访问 192.168.10.6 的时候，是不是就能够通过 vpn 的网口发出去了？

_vpn 的配置暂时说到这，后面还要修改。_

- 防火墙(iptables)

vpn配置的地址池是10.8.0.0/24，客户端的IP地址必然为10.8.0.x，那接收到的请求源IP地址必然也是10.8.0.x，那在北京我们要访问的IP是192.168.10.6，这个IP和我们的vpn不在同一个网段，如何通呢？

我们先说说这个访问的流程，北京访问192.168.10.6，因为加了路由，则请求会从vpn客户端的网口发送到默认网关，vpn服务器会接受这个请求，然后查找192.168.10.6这个IP地址，在vpn的网段中因为没有这个IP，所以访问不了？

那如果我们把这个请求转发出去是不是就可以通了呢？

我们先看看vpn服务器的网卡信息

```Shell
ip a
```

我们可以发现会有一个物理网卡(可能是ethX或ensX)的信息和一个tun的虚拟网卡信息，物理网卡正好是我们192.168.10.0/24的某一个IP，我们只要把从tun来的请求转到物理网卡，那就可以通啦。

我们用iptables在SNAT(源地址目标转换)做一个规则，应该就可以，命令如下

```Shell
sudo iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o [ethX|ensX] -j MASQUERADE

# 查看
sudo iptables -nL -t nat 
```
**解释：在防火墙的nat表当中的POSTROUTING链上添加（-A）一条规则，规则是从（-s）10.8.0.0网段过来的请求，出去（-o OPUTPUT）的时候都走ethX|ensX**

另外服务器要支持IP路由转发，系统配置要修改一下

```Shell
vi /etc/sysctl.conf

net.ipv4.ip_forward = 1   # 没有则添加，有修改为1（0禁止，1开启）

sysctl -p
```

OK! 这回再从北京的那台电脑上
```
ping 192.168.10.6
```
是不是通了？