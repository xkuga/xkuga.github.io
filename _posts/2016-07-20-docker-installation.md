---
layout: post
title: Docker 安装
date:  2016-07-20
author: kuga
---

一直以来，安装和配置环境都是很费时间的，虽然大多是重复的劳动，
但报错又总是层出不穷，不按套路出招，几个小时很快就过去了，
自己写的部署脚本也是不现实的，没有标准还不通用，往往就成了一次性脚本，
另外又不是很懂一些专业的运维工具，狗带的心都有了 :(

就在这时，Docker 这货跑过来说：哥可是一次编写，到处部署，要不要试试？
我靠，这不是 Java 的广告词吗，？？黑人问号？？

那好吧，总得先装一下吧...？

和官网重复的部分就不说了，下面主要说一下网络不太行的问题。

从 DaoCloud 安装
----------------

##### Mac

Mac 的话直接从官网下载 dmg 文件安装就好了，总感觉 Docker 很重视 Mac。

##### Ubuntu

安装 Docker 官方的最新发行版， 支持 Ubuntu 12.04 以上版本

    curl -sSL https://get.daocloud.io/docker | sh

安装过程结束后，可执行下面命令验证安装结果。如果看到输出 docker start/running 就表示安装成功。

    sudo service docker status

##### Centos

安装 Docker 官方的最新发行版

    curl -sSL https://get.daocloud.io/docker | sh
    sudo chkconfig docker on
    sudo systemctl start docker

安装过程结束后，可执行下面命令验证安装结果。如果看到输出 active (running) 就表示安装成功。

    sudo systemctl status docker

> 由于 CentOS 6 内核太旧，Docker 和 RedHat 都不再支持，请升级您的操作系统。

DaoCloud 加速器
---------------

这个才是重点啊，没有加速器，一般的网络，这镜像简直就拉不下来！！！

安装主机监控程序

    curl -sSL https://get.daocloud.io/daomonit/install.sh | sh -s 3006cb2348551801a8957d88d4ad2916f9015a70

> 主机监控程序可以帮助您将主机接入到 DaoCloud 智能分发网络中，通过调用 Docker API 管理您的容器。

安装完后会发现多了 DaoCloud 相关的镜像。

```
$ docker images
REPOSITORY                              TAG                 IMAGE ID            CREATED             SIZE
daocloud.io/daocloud/daocloud-toolset   latest              1ab33797d8a1        12 weeks ago        150.2 MB
daocloud.io/daocloud/daomonit           latest              afee7a12814f        10 months ago       149 MB
```

然后通过下面的命令就可以拉镜像了。

    dao pull ubuntu

最近搞 Docker 总是会想起 skey，一个 2014-2015 年和我支付宝来往最多的人(zha)。

引用
----

1. [https://dashboard.daocloud.io/nodes/new](https://dashboard.daocloud.io/nodes/new)
