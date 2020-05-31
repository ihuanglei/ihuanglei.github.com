---
title: Golang配置
date: 2020-04-07 13:46:37
tags:
categories: golang
---

>首先下载编译器，因为要翻墙，所以找个国内的地址。 根据你的操作系统，选择最新的一个版本，直接下载一个免安装的包 下载，下载完后解压到某个目录

+ mac系统：gox.x.x.darwin-amd64.tar.gz
+ linux系统：gox.x.x.linux-arm64.tar.gz
+ window系统：gox.x.x.windows-amd64.zip

### Mac && Linux 

配置环境变量 
vim ~/.bash_profile

```Shell
# 设置GOROOT 
export GOROOT=/当前go解压的目录
# 设置GOPATH (新版本已不用设置，由于博主是从老版本过来的所以保留之前的) 
export GOPATH=/当前go工作的目录
# 因为google被禁了所以要设置代理 
export GOPROXY=https://goproxy.io
# 使用go mod 
export GO111MODULE=on
export PATH=$PATH:$GOROOT/bin:$GOPATH/bin
```

### Window

配置环境变量 

我的电脑->属性->环境变量->添加

```
GOROOT: /当前go解压的目录

GOPATH: /当前go工作的目录

GOPROXY: https://goproxy.io

GO111MODULE: on

PATH: 最后添加;%GOROOT%/bin:%GOPATH%/bin
```


```Batch
go env
```
