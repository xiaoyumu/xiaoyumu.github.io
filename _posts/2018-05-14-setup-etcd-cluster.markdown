---
layout: post
title:  "在 CentOS 7 中安装 etcd cluster"
date:   2018-05-14 11:11:00 +0800
categories: etcd
---

概述
--

etcd 是一个高性能且安全的分布式键值数据库，广泛应用于分布式系统环境下的共享配置存储和服务发现，例如 k8s 。更详细的介绍可以参考 CoreOS 的官方文档，接下来我们就在6台装有 CentOS7 的 VM 中从头开始搭建一个etcd cluster。

这个[链接](http://www.infoq.com/cn/articles/etcd-interpretation-application-scenario-implement-principle)也介绍了etcd的一些常见应用场景。

下面是我们的基本机器信息。
```
etcd01 192.168.1.61 infra-app-01
etcd02 192.168.1.62 infra-app-02
etcd03 192.168.1.63 infra-app-03
etcd04 192.168.1.64 infra-app-04
etcd05 192.168.1.65 infra-app-05
```

在 cluster 中每台机器上的 host 文件中配置所有节点机器的名称和 IP 映射:
```bash
echo "192.168.1.61  infra-app-01" | sudo tee -a /etc/hosts
echo "192.168.1.62  infra-app-02" | sudo tee -a /etc/hosts
echo "192.168.1.63  infra-app-03" | sudo tee -a /etc/hosts
echo "192.168.1.64  infra-app-04" | sudo tee -a /etc/hosts
echo "192.168.1.65  infra-app-05" | sudo tee -a /etc/hosts
```

接下来首先在每个节点上安装 etcd 服务
```bash
$ sudo yum install -y etcd
```

配置 Cluster 时，可以先配置单节点 Cluster，这里我们可以选择 192.168.1.61 作为第一个节点。


配置 Cluster 中的第一个节点
--

按下面的配置进行设置，然后通过 Add member 方式将其他机器添加到 cluster 中,这样节点的配置会相对简单一些。

```bash
# 编辑配置文件 
$ sudo vi /etc/etcd/etcd.conf
```
对比修改下面的内容
```ini
# [member]
ETCD_NAME=etcd01
ETCD_LISTEN_PEER_URLS="http://192.168.1.61:2380"
ETCD_LISTEN_CLIENT_URLS="http://192.168.1.61:2379,http://127.0.0.1:2379"
#[cluster]
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.1.61:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.1.61:2379"
ETCD_INITIAL_CLUSTER="etcd07=http://192.168.1.61:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
```

修改后完整的配置:
```ini
#[Member]
#ETCD_CORS=""
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
#ETCD_WAL_DIR=""

ETCD_LISTEN_PEER_URLS="http://192.168.1.61:2380"
ETCD_LISTEN_CLIENT_URLS="http://192.168.1.61:2379,http://127.0.0.1:2379"

#ETCD_MAX_SNAPSHOTS="5"
#ETCD_MAX_WALS="5"
ETCD_NAME="etcd01"
#ETCD_SNAPSHOT_COUNT="100000"
#ETCD_HEARTBEAT_INTERVAL="100"
#ETCD_ELECTION_TIMEOUT="1000"
#ETCD_QUOTA_BACKEND_BYTES="0"
#ETCD_MAX_REQUEST_BYTES="1572864"
#ETCD_GRPC_KEEPALIVE_MIN_TIME="5s"
#ETCD_GRPC_KEEPALIVE_INTERVAL="2h0m0s"
#ETCD_GRPC_KEEPALIVE_TIMEOUT="20s"
#
#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.1.61:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.1.61:2379"
#ETCD_DISCOVERY=""
#ETCD_DISCOVERY_FALLBACK="proxy"
#ETCD_DISCOVERY_PROXY=""
#ETCD_DISCOVERY_SRV=""
ETCD_INITIAL_CLUSTER="etcd07=http://192.168.1.61:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
#ETCD_STRICT_RECONFIG_CHECK="true"
#ETCD_ENABLE_V2="true"
#
# 剩余的配置没有调整，所以这里省略
```

配置完成后即可启动服务

```bash
sudo systemctl start etcd
sudo systemctl enable etcd
sudo systemctl status etcd
```

向 cluster 中添加成员节点
--

首先在第一个节点 etcd01 执行下面的命令添加 etcd02 节点到 cluster 中
```bash
etcdctl member add etcd02 http://192.168.1.62:2380
```
命令输出
```
Added member named etcd02 with ID xxxxxxxxxxxxxxxx to cluster

ETCD_NAME="etcd02"
ETCD_INITIAL_CLUSTER="etcd02=http://192.168.1.62:2380,etcd01=http://192.168.1.61:2380"
ETCD_INITIAL_CLUSTER_STATE="existing"
```

然后修改 etcd02 (infra-app-02) 的 etcd 配置文件
```ini
#[Member]
#ETCD_CORS=""
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
#ETCD_WAL_DIR=""
ETCD_LISTEN_PEER_URLS="http://192.168.1.62:2380"
ETCD_LISTEN_CLIENT_URLS="http://192.168.1.62:2379,http://127.0.0.1:2379"
#ETCD_MAX_SNAPSHOTS="5"
#ETCD_MAX_WALS="5"
ETCD_NAME="etcd02"
#ETCD_SNAPSHOT_COUNT="100000"
#ETCD_HEARTBEAT_INTERVAL="100"
#ETCD_ELECTION_TIMEOUT="1000"
#ETCD_QUOTA_BACKEND_BYTES="0"
#ETCD_MAX_REQUEST_BYTES="1572864"
#ETCD_GRPC_KEEPALIVE_MIN_TIME="5s"
#ETCD_GRPC_KEEPALIVE_INTERVAL="2h0m0s"
#ETCD_GRPC_KEEPALIVE_TIMEOUT="20s"
#
#[Clustering]
#ETCD_INITIAL_ADVERTISE_PEER_URLS="http://localhost:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.1.62:2379"
#ETCD_DISCOVERY=""
#ETCD_DISCOVERY_FALLBACK="proxy"
#ETCD_DISCOVERY_PROXY=""
#ETCD_DISCOVERY_SRV=""
ETCD_INITIAL_CLUSTER="etcd01=http://192.168.1.61:2380,etcd02=http://192.168.1.62:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="existing"
#ETCD_STRICT_RECONFIG_CHECK="true"
#ETCD_ENABLE_V2="true"
#
# 剩余的配置没有调整，所以这里省略
```

配置完成后即可启动服务

```bash
sudo systemctl start etcd
sudo systemctl enable etcd
sudo systemctl status etcd
```

在第一个节点 etcd01 执行下面的命令添加 etcd03 节点到 cluster 中
```bash
etcdctl member add etcd03 http://192.168.1.63:2380
```
命令输出
```
Added member named etcd03 with ID xxxxxxxxxxxxxxxx to cluster

ETCD_NAME="etcd03"
ETCD_INITIAL_CLUSTER="etcd03=http://192.168.1.63:2380,etcd02=http://192.168.1.62:2380,etcd01=http://192.168.1.61:2380"
ETCD_INITIAL_CLUSTER_STATE="existing"
```

参考上面命令输出中的内容和上面 etcd02 节点的配置修改 etcd03 的 ```/etc/etcd/etcd.conf```。

修改完成后启动服务
```bash
sudo systemctl start etcd
sudo systemctl enable etcd
sudo systemctl status etcd
```

如果没有什么意外和错误信息输出，则可以在 cluster 中的任意节点中执行下面的命令检查 cluster 中的节点状态。

```bash
etcdctl member list
```

命令输出
```
xxxxxxxxxxxxxxxx: name=etcd02 peerURLs=http://192.168.1.62:2380 clientURLs=http://192.168.1.62:2379 isLeader=false
xxxxxxxxxxxxxxxx: name=etcd03 peerURLs=http://192.168.1.63:2380 clientURLs=http://192.168.1.63:2379 isLeader=false
xxxxxxxxxxxxxxxx: name=etcd01 peerURLs=http://192.168.1.61:2380 clientURLs=http://192.168.1.61:2379 isLeader=true
...
```

测试
--

在 cluster 中的其中一个节点中执行下面的命令添加一个测试配置项并设置一个值。如下面的命令所示，我们在 /test 下面创建一个名为 config的配置，并设置其值为 value1 。

```bash
etcdctl set /test/config value1
```

然后在另外一个节点上执行 get 命令，可以看到能够获得在刚才节点上设置的配置内容。

```bash
etcdctl get /test/config
```
到这里我们这个简单的 etcd cluster 就算配置完成了。不过值得注意的是上面的配置并不适用于生产环境，建议只用于测试 etcd cluster 的使用。

