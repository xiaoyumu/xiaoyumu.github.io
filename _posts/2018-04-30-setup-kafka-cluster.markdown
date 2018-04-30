---
layout: post
title:  "基于现有的zk群集部署kafka cluster"
date:   2018-04-30 17:44:00 +0800
categories: kafka
---

下载最新安装包

* [https://kafka.apache.org/downloads](https://kafka.apache.org/downloads)
* [https://www.apache.org/dyn/closer.cgi?path=/kafka/1.0.1/kafka_2.11-1.0.1.tgz](https://www.apache.org/dyn/closer.cgi?path=/kafka/1.0.1/kafka_2.11-1.0.1.tgz)

依赖环境准备:

* 安装JDK
* 安装Zookeeper Cluster

关于如何安装一个zk cluster 可以参考([这里]({% post_url 2018-04-29-setup-zookeeper-cluster %})).

解压缩:

```tar zxvf kafka_2.11-1.0.1.tgz```

切换到root执行下面的命令```sudo su```，将安装文件移动到/usr/local下，并创建符号链接，然后设置权限。

```bash
mv kafka_2.11-1.0.1 /usr/local/
ln -s /usr/local/kafka_2.11-1.0.1 /usr/local/kafka
cd /usr/local/kafka
useradd kafka
chown -R kafka. /usr/local/kafka_2.11-1.0.1
chown -h kafka. /usr/local/kafka
```


修改server.properties配置文件:
注意下面的broker.id=后面的值，根据cluster中的broker节点机器来确定，只要保证cluster中每个broker节点唯一即可。

```bash
sed -i 's~broker.id=0~broker.id=6~g' /usr/local/kafka/config/server.properties
```

修改zookeeper地址，下面的命令将配置文件```/usr/local/kafka/config/server.properties```中的```zookeeper.connect=localhost:2181```改为我们的zookeeper cluster节点列表：

```bash
sed -i 's~zookeeper.connect=localhost:2181~zookeeper.connect=infra-app-01:2181,infra-app-02:2181,infra-app-03:2181,infra-app-04:2181,infra-app-05:2181,infra-app-06:2181~g' /usr/local/kafka/config/server.properties
```


修改默认的log存放目录：

```bash
sed -i 's~log.dirs=/tmp/kafka-logs~log.dirs=/usr/local/kafka/kafka-logs~g' /usr/local/kafka/config/server.properties
```

完成后验证配置是否正确:

```bash
cat /usr/local/kafka/config/server.properties | grep broker.id=
cat /usr/local/kafka/config/server.properties | grep zookeeper.connect=
cat /usr/local/kafka/config/server.properties | grep log.dirs= 
```

有两种方式可以启动kafka server，其中用命令方式启动kafka server：

```
sudo bin/kafka-server-start.sh config/server.properties
```

推荐以服务方式启动kafka server，创建一个kafka service unit文件

```bash
sudo vi /etc/systemd/system/kafka.service
```

在```kafka.service```文件中添加以下内容:

```
[Unit]
Description=Apache Kafka server (broker)
Documentation=http://kafka.apache.org/documentation.html
Requires=network.target remote-fs.target
After=network.target remote-fs.target zookeeper.service

[Service]
Type=simple
User=kafka
Group=kafka
#Environment=JAVA_HOME=/usr/java/jdk1.8.0_152
ExecStart=/usr/local/kafka/bin/kafka-server-start.sh /usr/local/kafka/config/server.properties
ExecStop=/usr/local/kafka/bin/kafka-server-stop.sh

[Install]
WantedBy=multi-user.target
```

然后执行下面的命令启动和检查kafka服务的状态

```bash
sudo systemctl enable kafka
sudo systemctl start kafka
sudo systemctl status kafka
```

使用zookeeper client连接到zk cluster中验证brokers的id列表是否正确:

```bash
$ /usr/local/zookeeper/bin/zkCli.sh -server 192.168.1.61:2181
ls /brokers/ids
```

如下内容所示，zk的/brokers/ids已经存储了部署的kafka broker id:

```
[zk: 192.168.1.61:2181(CONNECTED) 1] ls /brokers
[ids, topics, seqid]
[zk: 192.168.1.61:2181(CONNECTED) 2] ls /brokers/ids
[1, 2, 3, 4, 5, 6]
[zk: 192.168.1.61:2181(CONNECTED) 3]
```

如果重启节点后, 由于zookeeper cluster暂时不能链接，可能会导致部分kafka service启动失败，
所以需要手动重启一下服务，建议kafka cluster与zk cluster不要共用节点机器。

```bash
sudo systemctl restart kafka
sudo systemctl status kafka
```

如下面的内容所示，节点2,3在一段时间后恢复。
```
[zk: 192.168.1.61:2181(CONNECTED) 0] ls /brokers/ids
[1, 4, 5, 6]
[zk: 192.168.1.61:2181(CONNECTED) 1] ls /brokers/ids
[1, 2, 3, 4, 5, 6]
```
