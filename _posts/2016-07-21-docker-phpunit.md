---
layout: post
title: Docker 与 PHPUnit 单元测试
date:  2016-07-21
author: kuga
header-img: img/docker.jpg
---

如果你有写单元测试，你一定会发现随着测试的增加，测试所需要的环境会越来越复杂。
自己本地配置一份，测试服务器配置一份，一旦不止你一个人开发，其它人也要配置一份，想想都累。
但如果把测试集成到 Docker 的容器中，大家只要拉取镜像就能跑了，是不是很方便？
而且共同维护一份 Dockerfile 就相当于文档，既标准又清晰，想想都有点小激动！

那我们就从一个具体的例子说明吧，主要分两个部分：

1. 写一个简单的 PHPUnit 测试。
2. 创建 Dockerfile，并运行测试。

## PHPUnit
----------

这里举一个测试 Redis 的例子，保存为 RedisTest.php。

```php
<?php

class RedisTest extends PHPUnit_Framework_TestCase
{
    public function testBar()
    {
        $redis = new Redis();
        $redis->connect('127.0.0.1', 6379);
        $redis->set('name', '周杰伦');
        $this->assertEquals('周杰伦', $redis->get('name'));
    }
}
```

因为这个比较简单，就不解释了，你没看错，世界上最好的语言也要写测试啊哈哈哈。

## Dockerfile
-------------

这个才是重点啊，Dockerfile 其实就是一份文档。

```bash
FROM ubuntu:14.04

WORKDIR /

# ubuntu14.04 国内源
RUN printf 'deb http://cn.archive.ubuntu.com/ubuntu/ trusty main restricted universe multiverse\ndeb http://cn.archive.ubuntu.com/ubuntu/ trusty-security main restricted universe multiverse\ndeb http://cn.archive.ubuntu.com/ubuntu/ trusty-updates main restricted universe multiverse\ndeb http://cn.archive.ubuntu.com/ubuntu/ trusty-backports main restricted universe multiverse\ndeb http://cn.archive.ubuntu.com/ubuntu/ trusty-proposed main restricted universe multiverse\ndeb-src http://cn.archive.ubuntu.com/ubuntu/ trusty main restricted universe multiverse\ndeb-src http://cn.archive.ubuntu.com/ubuntu/ trusty-security main restricted universe multiverse\ndeb-src http://cn.archive.ubuntu.com/ubuntu/ trusty-updates main restricted universe multiverse\ndeb-src http://cn.archive.ubuntu.com/ubuntu/ trusty-backports main restricted universe multiverse\ndeb-src http://cn.archive.ubuntu.com/ubuntu/ trusty-proposed main restricted universe multiverse\ndeb http://archive.canonical.com/ubuntu/ trusty partner\n' > /etc/apt/sources.list

RUN apt-get update && apt-get install -y \
    make \
    curl \
    wget \
    php5-cli \
    php5-dev

# 安装 redis
RUN wget http://download.redis.io/releases/redis-2.8.6.tar.gz
RUN tar zxvf redis-2.8.6.tar.gz
RUN cd /redis-2.8.6/deps && make
RUN cd /redis-2.8.6 && make && make install

# 安装 phpredis 扩展
RUN wget https://github.com/phpredis/phpredis/archive/2.2.8.tar.gz
RUN tar zxvf 2.2.8.tar.gz
RUN cd /phpredis-2.2.8 && phpize
RUN cd /phpredis-2.2.8 && ./configure
RUN cd /phpredis-2.2.8 && make && make install
RUN echo extension=redis.so > /etc/php5/cli/conf.d/redis.ini

# 安装 composer
RUN curl -sS https://getcomposer.org/installer | php
RUN mv composer.phar /usr/local/bin/composer

# 配置 composer 国内镜像
RUN composer config -g repo.packagist composer https://packagist.phpcomposer.com

# 安装 phpunit
RUN echo '{"require-dev": {"phpunit/phpunit": "4.8.*"}}' > composer.json
RUN composer install

# 把当前目录挂到容器的根目录
ADD . /

RUN chmod 755 /start.sh

# 运行启运脚本
CMD ["/bin/bash", "/start.sh"]
```

该说的上面已经有注释了，
这里要注意的是 Ubuntu 和 Composer 换成了国内的源，不然基本上跑不动。
然后把 Dockerfile 和 RedisTest.php 放到同一个目录，在该目录下运行以下命令创建镜像。

```bash
docker build -t my/phpunit:v1 .
```

-t 用于指定镜像的名字和标签，这里名字是 my/phpunit，标签是 v1。

创建过程其实可以随时中断的，因为有缓存，一般如果网络慢，可以重跑一次试试。
这个时候你可以去吃点狗粮或者喝杯可乐，因为的确有点久。

如果在构建的时候有报错，可以进入前一步成功的容器进行调试，
在出错的输出里有前一步成功的容器 hash，当然，docker images 也能看到，
那些标记为 none 的容器就是中间状态的容器，运行下面的命令就能进入容器调试。

```bash
docker run -it hash
```

构建完成后，终于可以运行测试了。

```bash
docker run my/phpunit:v1
```

这突然让我想起了当初教我写单元测试的那个人 :)

好吧，我们下期 Redis 3.0 集群再见〜
