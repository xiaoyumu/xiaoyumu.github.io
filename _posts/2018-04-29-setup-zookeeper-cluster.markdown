---
layout: post
title:  "部署zookeeper集群"
date:   2018-04-29 17:44:00 +0800
categories: zookeeper
---

本例创建含有6个节点的Zookeeper cluster (Replication Mode)

* infra-app-01  192.168.1.61
* infra-app-02  192.168.1.62
* infra-app-03  192.168.1.63
* infra-app-04  192.168.1.64
* infra-app-05  192.168.1.65
* infra-app-06  192.168.1.66

下载最新安装包 (先安装JDK/OpenJDK)
[https://www.apache.org/dyn/closer.cgi/zookeeper/](https://www.apache.org/dyn/closer.cgi/zookeeper/)

例如:
[zookeeper-3.4.11.tar.gz](http://www-eu.apache.org/dist/zookeeper/zookeeper-3.4.12/zookeeper-3.4.11.tar.gz)

参考官网文档
http://zookeeper.apache.org/doc/r3.4.11/zookeeperStarted.html

解压缩安装包

```tar zxvf zookeeper-3.4.11.tar.gz```

切换到root(```sudo su```)后执行下面的命令:

```bash
# 将解压后的文件移动到/usr/local目录下
mv zookeeper-3.4.11 /usr/local/
# 在/usr/local目录下创建名为zookeeper的符号链接指向/usr/local/zookeeper-3.4.11
ln -s /usr/local/zookeeper-3.4.11 /usr/local/zookeeper
# 切换到zookeeper目录，设置owner为root账户
cd /usr/local/zookeeper-3.4.11
chown -R root:root *

# 创建data目录
mkdir data

# 将sample配置文件复制为zoo.cfg
cp conf/zoo_sample.cfg conf/zoo.cfg

# 更新默认配置
sed -i 's~syncLimit=5~syncLimit=2~g' conf/zoo.cfg
# 更新默认data目录
sed -i 's~dataDir=/tmp/zookeeper~dataDir=/usr/local/zookeeper/data~g' conf/zoo.cfg
# 在配置文件中添加cluster中各节点信息
echo "server.1=infra-app-01:2888:3888" | tee -a conf/zoo.cfg
echo "server.2=infra-app-02:2888:3888" | tee -a conf/zoo.cfg
echo "server.3=infra-app-03:2888:3888" | tee -a conf/zoo.cfg
echo "server.4=infra-app-04:2888:3888" | tee -a conf/zoo.cfg
echo "server.5=infra-app-05:2888:3888" | tee -a conf/zoo.cfg
echo "server.6=infra-app-06:2888:3888" | tee -a conf/zoo.cfg
exit # 退出su模式
```

为每台机器生成一个包含唯一ID的myid文件到data目录中，例如下面的例子在节点infra-app-01上使用1作为myid的内容:

```bash
sudo touch /usr/local/zookeeper-3.4.11/data/myid
echo "1" | sudo tee -a /usr/local/zookeeper-3.4.11/data/myid
```

使用命令启动zookeeper

```bash
sudo bin/zkServer.sh start
```

将zookeeper配置为自动启动的后台服务:
创建一个service unit文件/usr/lib/systemd/system/zookeeper.service
```
sudo vi /usr/lib/systemd/system/zookeeper.service
```

在service unit文件中写入如下内容:
```
[Unit]
Description=Zookeeper
After=syslog.target

[Service]
SyslogIdentifier=zookeeper
TimeoutStartSec=10min
Type=forking
ExecStart=/usr/local/zookeeper/bin/zkServer.sh start
ExecStop=/usr/local/zookeeper/bin/zkServer.sh stop

[Install]
WantedBy=multi-user.target
```

然后执行:

```bash
sudo systemctl enable zookeeper
sudo systemctl start zookeeper
```

查看zookeeper日志 ```cat zookeeper.out```

使用下面命令连接zookeeper本地服务

```/usr/local/zookeeper/bin/zkCli.sh -server 127.0.0.1:2181```

也可以使用下面命令连接远程zookeeper服务

```/usr/local/zookeeper/bin/zkCli.sh -server 192.168.1.61:2181```

然后可以使用下面的命令测试在Cluster中一台机器创建数据项，群集中其它机器可以正常获取到数据

```bash
ls /
create /zk_test my_data
get /zk_test
set /zk_test junk
```

使用命令```quit```退出zk client
