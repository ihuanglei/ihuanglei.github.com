---
title: Centos配置
date: 2020-05-08 13:05:37
tags:
---

## Centos 安装好后初始化

- 安装第三方源

```
    yum install epel-release -y

```

- 安装时间同步工具

```
    yum install ntpdate -y
    ntpdate cn.ntp.org.cn

```
