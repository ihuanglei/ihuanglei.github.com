---
title: Openvpn 配置
date: 2020-05-26 10:19:10
tags:
categories: vpn
---

> 注意：本教程在 centos7 下操作

### 依赖软件

- 软件版本

```
openvpn 2.4.9
easy-rsa 3.0.7

如果版本不一致，后面的路径会有所差别

```

- 安装扩展源

```Shell
    yum install epel-release -y
```

- 安装 openvpn easy-rsa3

```Shell
    yum install openvpn easy-rsa -y
```

### 服务器配置

- 拷贝配置文件

```Shell
cp /usr/share/doc/openvpn-2.4.9/sample/sample-config-files/server.conf  /etc/openvpn/server/

这里的openvpn版本如果不一致，请注意路径

```

```Shell
cp -r /usr/share/easy-rsa /etc/openvpn/
```

```Shell
cp /usr/share/doc/easy-rsa-3.0.7/vars.example /etc/openvpn/easy-rsa/3/vars

这里的easy-rsa版本如果不一致，请注意路径
```

- 进入 easy-rsa 目录

```Shell
    cd /etc/openvpn/easy-rsa/3
```

- 创建空 PKI

```Shell
./easyrsa init-pki
```

- 创建 CA 证书(不使用密码)

```Shell
./easyrsa build-ca nopass
 (一路回车就可以)
```

- 创建服务端证书(不使用密码)

```Shell
./easyrsa gen-req server nopass
 (一路回车就可以)
```

- 签约服务端证书

```Shell
./easyrsa sign server server
```

- 创建 Diffie-Hellman

```Shell
./easyrsa gen-dh
```

- 整理服务器证书

```Shell
mkdir /etc/openvpn/server/certs
cp /etc/openvpn/easy-rsa/3/pki/dh.pem /etc/openvpn/server/certs
cp /etc/openvpn/easy-rsa/3/pki/ca.crt /etc/openvpn/server/certs
cp /etc/openvpn/easy-rsa/3/pki/issued/server.crt /etc/openvpn/server/certs
cp /etc/openvpn/easy-rsa/3/pki/private/server.key /etc/openvpn/server/certs
```

- 服务器配置文件

```Shell
vim /etc/openvpn/server/server.conf
```

需修改项如下:

```
local 192.168.10.155 # 当前服务器ip地址，请修改

proto tcp  # 去掉前面的;
;proto udp # 前面加上;

ca /etc/openvpn/server/certs/ca.crt        # 修改路径
cert /etc/openvpn/server/certs/server.crt  #
key /etc/openvpn/server/certs/server.key   #

dh /etc/openvpn/server/certs/dh.pem        #

ifconfig-pool-persist /etc/openvpn/ipp.txt #

client-to-client # 去掉前面的;

comp-lzo # 去掉前面的;

mute 20  # 去掉前面的;

log-append openvpn.log # 去掉前面的;

user openvpn # 去掉前面的;修改运行用户
group openvpn　＃ 去掉前面的;修改运行用户组

;explicit-exit-notify 1 # 前面加上;

;tls-auth ta.key 0 # 前面加上;

```

- 启动 openvpn 服务

```Shell
systemctl start openvpn-server@server.service
```

```Shell
查看启动状态
systemctl status openvpn-server@server.service
```

```Shell
添加到自启动
systemctl enable openvpn-server@server.service
```

### 客户端配置

- 客户端配置文件

```Shell
cp /usr/share/doc/openvpn-2.4.9/sample/sample-config-files/client.conf /etc/openvpn/client/
```

- 修改配置文件

```
# 没有用的都可以删除了，留以下内容

client

dev tun

proto tcp
remote 192.168.10.155 1194  # 服务器ip地址和端口号，和服务端配置一致
resolv-retry infinite
nobind
persist-key
persist-tun

ca ca.crt          # 修改配置文件路径
cert client.crt    #
key client.key     #

remote-cert-tls server
cipher AES-256-CBC
verb 3
mute 20
```

- 创建客户证书(不使用密码)

```Shell
创建test用户

./easyrsa gen-req test nopass
 (一路回车就可以)
```

- 签约客户证书

```Shell
./easyrsa import-req /etc/openvpn/easy-rsa/3/pki/reqs/test.req test
```

```Shell
./easyrsa sign client test
```

- 安装 openvpn client

[下载地址](https://github.com/OpenVPN/openvpn-gui/releases)

安装好后导入配置文件
