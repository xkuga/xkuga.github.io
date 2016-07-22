---
layout: post
title: Docker Redis Cluster
date:  2016-07-22
author: kuga
header-img: img/docker.jpg
---

最近需要在 3 台机器上搭建一个 Redis 3.0 的集群，
因为在测试的时候并没有这么多机器，即使是虚拟机也要一台台部署，很不方便。
这个时候很自然就想到 Docker 了。

其实关于 Docker 部署 Redis 集群的文章，网上有很多，GitHub 也有源码，
但是这些例子大多是在一个容器中部署的，基本上不存在网络的问题。
但在真实的场景中，主从一般是不在同一台机器中的，
所以这次我们在 3 个容器上做，相当于 3 台机器，每个容器 1 主 2 从。

## Redis 集群结构
----------------

Reids 集群最少需要 3 个主结点才能正常工作，相当于一个环状结构。
这个环一共分了 16384 个 slot，每个结点负责一部分 slot。
当你需要操作一个 key 的时候，他会计算这个 key 的哈希，然后会重定向到对应结点进行操作。
如果这个环状结构有任何一个缺口，都会导致集群不可用。

我们这次测试的集群结构图如下，注意颜色代表主从关系。

![redis-cluster](/img/redis-cluster.png)

每一个主的另外两个从都在不同的机器，这样保证了任意一台机器都拥有整个集群的数据。

假如机器 A 挂了，那么机器 A 上的主会落到机器 B 或者机器 C 上。
这个时候集群一定是正常的。

现在假设机器 A 的主落到了机器 B，也就是说机器 B 有两个主了。
如果机器 Ｃ 现在挂掉了，那么集群还是正常的，
但如果机器 B 挂了，那么就真的完蛋了〜，
因为集群中两个主同时挂掉就无法进行选举，所以要是两台机器挂了，集群会有 50% 的概率撑住。

## 构建容器
----------

首先我们创建下面的目录结构。

```
redis-cluster
├── Dockerfile
├── redis-conf
│   ├── 7000
│   │   └── redis.conf
│   ├── 7001
│   │   └── redis.conf
│   └── 7002
│       └── redis.conf
└── start.sh
```

#### Dockerfile

```bash
FROM centos:6.8

RUN yum install -y \
    gcc \
    make \
    wget \
    ruby \
    ruby-devel \
    rubygems \
    rpm-build \
    vim \
    net-tools

# 安装 redis 3.0
RUN wget http://download.redis.io/releases/redis-3.0.7.tar.gz
RUN tar zxvf redis-3.0.7.tar.gz
RUN cd /redis-3.0.7 && make MALLOC=libc && make install

# redis-trib.rb 是 redis cluster 提供的命令行工具
RUN cp /redis-3.0.7/src/redis-trib.rb /usr/local/bin

RUN gem install redis

# 把当前目录挂到容器的 / 目录
ADD . /

RUN chmod 755 /start.sh

# 运行启运脚本
CMD ["/bin/bash", "/start.sh"]
```

#### start.sh

```bash
cd /redis-conf/7000 && redis-server redis.conf &
cd /redis-conf/7001 && redis-server redis.conf &
cd /redis-conf/7002 && redis-server redis.conf &
/bin/bash
```

#### redis.conf

```
# 找到默认的 redis.conf，然后修改以下地方，port 分别为 7000-7002
port 7000
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
appendonly yes
```

Dockerfile 比较清晰，我们就直接构建容器吧。

```bash
docker build -t my/redis-cluster:v1 .
```

运行的时候我们需要给容器分配 IP，而分配置 IP 需要先创建网络，参数要根据你的情况修改。

```bash
docker network create --subnet 10.10.10.0/24 onepiece
```

下面是一些网络相关的命令。

```bash
# 列出所有网络
docker network ls

# 查看某个网络
docker network inspect onepiece

# 删除某个网络
docker network rm onepiece
```

接着我们就可以运行 Docker 容器了。

```bash
docker run --net onepiece --ip 10.10.10.100 -it -p 7000-7002:7000-7002 my/redis-cluster:v1
```

上面的意思是使用 onepiece 这个网络，并给容器分配 10.10.10.100 这个 IP。
-p 是端口映射，冒号左边是主机的端口，右边是容器的端口，
我们容器里的 redis 就是使用了 7000-7002 这 3 个端口。

继续运行第二个容器，命令在 IP 和端口上有些修改，如下。

```bash
docker run --net onepiece --ip 10.10.10.101 -it -p 7003-7005:7000-7002 my/redis-cluster:v1
```

第三个容器也相似。

```bash
docker run --net onepiece --ip 10.10.10.102 -it -p 7006-7008:7000-7002 my/redis-cluster:v1
```

我们可以在其中一个容器内部看下是否有 3 个 redis 实例在运行。

```bash
ps aux | grep redis
```

接下来就是使用这 9 个实例建立集群，在任意一个容器内部运行下面的命令。

```bash
redis-trib.rb create --replicas 2 10.10.10.100:7000 10.10.10.101:7000 10.10.10.102:7000 10.10.10.101:7001 10.10.10.102:7001 10.10.10.100:7001 10.10.10.100:7002 10.10.10.102:7002 10.10.10.101:7002
```

\-\-replicas 2 表示集群中 1 个主结点使用 2 个从结点，
他会自动根据这个规则分配主从结点，会尽量把主从结点分配到不同的机器上，
但还是有可能分配到同一台机器，这个我们先不管，先把集群跑起来。

运行了以上的命令会让你确认是否采用他的配置去建立集群，提示如下。

```
>>> Creating cluster
>>> Performing hash slots allocation on 9 nodes...
Using 3 masters:
10.10.10.102:7000
10.10.10.101:7000
10.10.10.100:7000
Adding replica 10.10.10.101:7001 to 10.10.10.102:7000
Adding replica 10.10.10.100:7001 to 10.10.10.102:7000
Adding replica 10.10.10.102:7001 to 10.10.10.101:7000
Adding replica 10.10.10.102:7002 to 10.10.10.101:7000
Adding replica 10.10.10.101:7002 to 10.10.10.100:7000
Adding replica 10.10.10.100:7002 to 10.10.10.100:7000
M: ba3bd6e0296b6cead5fd73e440808f4b4e4449d3 10.10.10.100:7000
   slots:10923-16383 (5461 slots) master
M: 5937611adbb704eba78908d9d2fba2bcd20ab035 10.10.10.101:7000
   slots:5461-10922 (5462 slots) master
M: 1381ea189a7b2653d87af6ef2e36a31f731ca411 10.10.10.102:7000
   slots:0-5460 (5461 slots) master
S: 00f5d58fd40d3f77bfa3b6a93b00b72d2d91ff88 10.10.10.101:7001
   replicates 1381ea189a7b2653d87af6ef2e36a31f731ca411
S: 4764fc41edbcc2f8e9f6758476ad0762b74d7450 10.10.10.102:7001
   replicates 5937611adbb704eba78908d9d2fba2bcd20ab035
S: 5b9ca6e79501b51c2ad907ec778b4f6234aaa2ae 10.10.10.100:7001
   replicates 1381ea189a7b2653d87af6ef2e36a31f731ca411
S: 1e8c3b25664489ca0d718049fe80bf30e75caec4 10.10.10.100:7002
   replicates ba3bd6e0296b6cead5fd73e440808f4b4e4449d3
S: 3975b12290072860a4df5431cea61f3496165b6b 10.10.10.102:7002
   replicates 5937611adbb704eba78908d9d2fba2bcd20ab035
S: 656fba0c0d8f911c54d54a548a242a37b7d65fc0 10.10.10.101:7002
   replicates ba3bd6e0296b6cead5fd73e440808f4b4e4449d3
Can I set the above configuration? (type 'yes' to accept):
```

这里输入 yes。
然后你会发现另外两个容器会输出相应的提示信息，
这也是为什么在 start.sh 里运行 redis 的时候没有关闭标准输出，
我们就是想看集群运行时的各种提示信息。

在容器里敲个回车就能回到正常的命令行，我们可以使用下面的命令检查一下容器的状态。

```
redis-trib.rb check 10.10.10.100:7000
```

输出如下。

```
>>> Performing Cluster Check (using node 10.10.10.100:7000)
M: ba3bd6e0296b6cead5fd73e440808f4b4e4449d3 10.10.10.100:7000
   slots:10923-16383 (5461 slots) master
   2 additional replica(s)
S: 656fba0c0d8f911c54d54a548a242a37b7d65fc0 10.10.10.101:7002
   slots: (0 slots) slave
   replicates ba3bd6e0296b6cead5fd73e440808f4b4e4449d3
S: 3975b12290072860a4df5431cea61f3496165b6b 10.10.10.102:7002
   slots: (0 slots) slave
   replicates 5937611adbb704eba78908d9d2fba2bcd20ab035
M: 1381ea189a7b2653d87af6ef2e36a31f731ca411 10.10.10.102:7000
   slots:0-5460 (5461 slots) master
   2 additional replica(s)
S: 00f5d58fd40d3f77bfa3b6a93b00b72d2d91ff88 10.10.10.101:7001
   slots: (0 slots) slave
   replicates 1381ea189a7b2653d87af6ef2e36a31f731ca411
S: 4764fc41edbcc2f8e9f6758476ad0762b74d7450 10.10.10.102:7001
   slots: (0 slots) slave
   replicates 5937611adbb704eba78908d9d2fba2bcd20ab035
M: 5937611adbb704eba78908d9d2fba2bcd20ab035 10.10.10.101:7000
   slots:5461-10922 (5462 slots) master
   2 additional replica(s)
S: 1e8c3b25664489ca0d718049fe80bf30e75caec4 10.10.10.100:7002
   slots: (0 slots) slave
   replicates ba3bd6e0296b6cead5fd73e440808f4b4e4449d3
S: 5b9ca6e79501b51c2ad907ec778b4f6234aaa2ae 10.10.10.100:7001
   slots: (0 slots) slave
   replicates 1381ea189a7b2653d87af6ef2e36a31f731ca411
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```

这里有主从关系，但是这个不直观，所以我写了一个脚本。

```python
import sys


class Node:
    def __init__(self, id, host, pid=0):
        self.id = id
        self.host = host
        self.pid = pid


def parse(text):
    rows = text.split('\n')
    nodes = []

    for i in range(len(rows)):
        foo = rows[i].split(' ')
        if (foo[0] == 'M:' or foo[0] == 'S:') and len(foo[1]) == 40:
            id = foo[1]
            host = foo[2]
            bar = rows[i + 2].strip().split(' ')
            pid = bar[1] if len(bar[1]) == 40 else 0
            nodes.append(Node(id, host, pid))

    return nodes


def pretty_print(nodes):
    master_nodes = []
    slave_nodes = []

    for node in nodes:
        if node.pid:
            slave_nodes.append(node)
        else:
            master_nodes.append(node)

    for master in master_nodes:
        print('\n%s %s' % (master.host, master.id))
        for slave in slave_nodes:
            if slave.pid == master.id:
                print('    %s %s' % (slave.host, slave.id))

    print('')


if __name__ == '__main__':
    pretty_print(parse(sys.stdin.read()))
```

把上面的代码保存为 magic.py，然后查看集群状态。

```bash
redis-trib.rb check 10.10.10.100:7000 | python magic.py
```

输出如下，有缩进的是从结点。

```
10.10.10.100:7000 ba3bd6e0296b6cead5fd73e440808f4b4e4449d3
    10.10.10.101:7002 656fba0c0d8f911c54d54a548a242a37b7d65fc0
    10.10.10.100:7002 1e8c3b25664489ca0d718049fe80bf30e75caec4

10.10.10.102:7000 1381ea189a7b2653d87af6ef2e36a31f731ca411
    10.10.10.101:7001 00f5d58fd40d3f77bfa3b6a93b00b72d2d91ff88
    10.10.10.100:7001 5b9ca6e79501b51c2ad907ec778b4f6234aaa2ae

10.10.10.101:7000 5937611adbb704eba78908d9d2fba2bcd20ab035
    10.10.10.102:7002 3975b12290072860a4df5431cea61f3496165b6b
    10.10.10.102:7001 4764fc41edbcc2f8e9f6758476ad0762b74d7450
```

问题很明显，就是主从被分配到了同一台容器了。
关于这个问题，GitHub 上有相关的 <a href="https://github.com/antirez/redis/issues/2204" target="_blank">issue</a>，
不过有点旧，我还没细看，现在除了手动配置，我还没找到其它比较方便的方法，留个坑。

我们先简单测试一下集群，一定要加 -c，这个是集群的选项，不然无法进行重定向。

```bash
$ redis-cli -c -h 10.10.10.100 -p 7000
10.10.10.100:7000> set name Mayday
-> Redirected to slot [5798] located at 10.10.10.101:7000
OK
```

就像前面说的一样，哈希了 key 之后做了重定向。
其它测试我就不做了，留给大家。
另外，如果实例挂掉了，容器里是会有相关的提示信息的，可以参考一下。

## 手动配置
----------

最后就简单说一下怎么手动配置，这有点麻烦，如果你有更好的方法留个言吧。

首先我们重新运行容器，回到初始状态，然后使用以下命令构建集群。

```bash
redis-trib.rb create --replicas 1 10.10.10.100:7000 10.10.10.101:7000 10.10.10.102:7000 10.10.10.101:7001 10.10.10.102:7001 10.10.10.100:7001
```

这里采用 1 主 1 从，然后我们配合之前的 magic.py 脚本看一下集群状态。

```
10.10.10.100:7000 954767ebf73d251e089e8744fdc372741e9692a3
    10.10.10.100:7001 571b2561e53b3160abda1f23a5a6da15a65efa2e

10.10.10.102:7000 f28185a65d377e096ffbdfe7b6b58c70ceeb17f8
    10.10.10.101:7001 27df91e5e16006e8be3f6130257f04881d3cd48b

10.10.10.101:7000 f648ca54cc768e7b8ed6468b6395b93d3f5cd843
    10.10.10.102:7001 33d9040b65e6c5cd9678b8e5fc3dfcf0de0a2028
```

看了上面的结果，这货还说会尽量分配到不同的机器，简直坑爹啊！
现在要做的是把有问题的从结点从集群中删掉，然后再重新加进去，操作如下。

```bash
redis-trib.rb del-node 10.10.10.100:7001 571b2561e53b3160abda1f23a5a6da15a65efa2e
```

因为删除的是从结点，所以会比较顺利，提示如下。

```
14:S 22 Jul 07:47:49.725 # User requested shutdown...
14:S 22 Jul 07:47:49.725 * Calling fsync() on the AOF file.
14:S 22 Jul 07:47:49.726 * Saving the final RDB snapshot before exiting.
14:S 22 Jul 07:47:49.730 * DB saved on disk
14:S 22 Jul 07:47:49.732 # Redis is now ready to exit, bye bye...
10:M 22 Jul 07:47:49.734 # Connection with slave 10.10.10.100:7001 lost.
```

我们再看一下集群状态。

```
10.10.10.100:7000 954767ebf73d251e089e8744fdc372741e9692a3

10.10.10.102:7000 f28185a65d377e096ffbdfe7b6b58c70ceeb17f8
    10.10.10.101:7001 27df91e5e16006e8be3f6130257f04881d3cd48b

10.10.10.101:7000 f648ca54cc768e7b8ed6468b6395b93d3f5cd843
    10.10.10.102:7001 33d9040b65e6c5cd9678b8e5fc3dfcf0de0a2028
```

好，一切正常。现在再删除 10.10.10.102:7000 的从结点，我就不重复了。

删除之后我们看一下 10.10.10.100 中的 redis 进程的状态。

```
$ ps aux | grep redis
root        10  0.2  0.1  28496  3796 ?        Sl   07:41   0:01 redis-server *:7000 [cluster]
root        13  0.2  0.1  27352  3556 ?        Sl   07:41   0:01 redis-server *:7002 [cluster]
```

你会发现删除结点后进程也停了，
如果现在把进程跑起来，再加入到集群是不行的，因为他还保留有之前的集群信息。
我们还需要在他的配置目录中删除那些信息。

```bash
cd /redis-conf/7001 && ls | grep -v redis.conf | xargs rm
```

好了，现在可以把 10.10.10.100:7001 跑起来了。

```bash
cd /redis-conf/7001 && redis-server redis.conf &
```

把实例加入到集群，这里指定了 master 是 10.10.10.102:7000，slave 是 10.10.10.100:7001。

```bash
redis-trib.rb add-node --slave --master-id f28185a65d377e096ffbdfe7b6b58c70ceeb17f8 10.10.10.100:7001 10.10.10.102:7000
```

加入后的状态如下。

```
10.10.10.100:7000 954767ebf73d251e089e8744fdc372741e9692a3

10.10.10.102:7000 f28185a65d377e096ffbdfe7b6b58c70ceeb17f8
    10.10.10.100:7001 9dd8620a80588a76a9840bfdebb8b0ffc353b198

10.10.10.101:7000 f648ca54cc768e7b8ed6468b6395b93d3f5cd843
    10.10.10.102:7001 33d9040b65e6c5cd9678b8e5fc3dfcf0de0a2028
```

重复的不说了，这个就是一个 ugly 的方法哈哈哈。

## 未解决的问题
-------------

虽然容器内的网络是通的，但如果要把集群提供给外部使用，一旦发生重定向，就会失败，
原因是外部主机和容器的网络不通。这也有相关的 <a href="https://github.com/antirez/redis/issues/2527" target="_blank">issue</a>。
今天先写到这，以后再填，好累 XD。
