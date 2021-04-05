# Zookeeper和Kafka安装

Zookeeper 版本 3.5.7

Kafka 版本 2.11-2.4.1

## 集群规划

| hadoop12  | hadoop13  | hadoop14  |
| :-------: | :-------: | :-------: |
| ZK、Kafka | ZK、Kafka | ZK、Kafka |

## 安装包准备

```shell
# 将 apache-zookeeper-3.5.7-bin.tar.gz 上传至 /opt/package 目录中
# 将 kafka_2.11-2.4.1.tgz 上传至 /opt/package 目录中
```

## 解压并复制给其他服务器

```shell
tar -zxvf apache-zookeeper-3.5.7-bin.tar.gz -C /opt
tar -zxvf kafka_2.11-2.4.1.tgz -C /opt
# 修改zookeeper目录名
cd /opt && mv apache-zookeeper-3.5.7-bin zookeeper-3.5.7
pdcp -a -r /opt/zookeeper-3.5.7 /opt
pdcp -a -r /opt/kafka_2.11-2.4.1 /opt
```

## 安装Zookeeper

### 配置服务器编号

```shell
# 创建 zkData 目录
pdsh -a "cd /opt/zookeeper-3.5.7 && mkdir zkData"
# 创建文件 myid
vim myid
# 三台服务器分别输入 2，3，4
```

### 配置zoo.cfg文件

```shell
# 重命名conf这个目录下的zoo_sample.cfg为zoo.cfg
mv zoo_sample.cfg zoo.cfg
# 修改 zoo.cfg
vim zoo.cfg
# 修改 dataDir 
dataDir=/opt/zookeeper-3.5.7/zkData
# 添加如下内容
#######################cluster##########################
server.2=hadoop12:2888:3888
server.3=hadoop13:2888:3888
server.4=hadoop14:2888:3888
```

### 复制配置文件到其他服务器

```shell
pdcp -a /opt/zookeeper-3.5.7/conf/zoo.cfg /opt/zookeeper-3.5.7/conf/zoo.cfg
```

### 启动

```shell
pdsh -a "cd /opt/zookeeper-3.5.7 && bin/zkServer.sh start"
```

## 安装Kafka

### 配置server.properties文件

```shell
# 在 /opt/kafka_2.11-2.4.1 目录下创建logs文件夹
pdsh -a "cd /opt/kafka_2.11-2.4.1 && mkdir data"
# 修改配置文件
vim /opt/kafka_2.11-2.4.1/config/server.properties
# 修改以下内容
#broker的全局唯一编号，不能重复
broker.id=2
#删除topic功能使能
delete.topic.enable=true
#kafka运行日志存放的路径
log.dirs=/opt/kafka_2.11-2.4.1/data
#配置连接Zookeeper集群地址
zookeeper.connect=hadoop12:2181,hadoop13:2181,hadoop14:2181/kafka
```

### 配置环境变量

```shell
vim /etc/profile.d/my_env.sh
# 添加以下内容
# kafka enviroment
export KAFKA_HOME=/opt/kafka_2.11-2.4.1
export PATH=$PATH:$KAFKA_HOME/bin
```

### 复制配置文件到其他服务器

```shell
pdcp -a /opt/kafka_2.11-2.4.1/config/server.properties /opt/kafka_2.11-2.4.1/config/server.properties
# 修改hadoop3中的 broker.id=3 hadoop4中的 broker.id=4

pdcp -a /etc/profile.d/my_env.sh /etc/profile.d/my_env.sh
```

### 启动

```shell
pdsh -a "kafka-server-start.sh -daemon /opt/kafka_2.11-2.4.1/config/server.properties"
```

## Kafka压力测试

用Kafka官方自带的脚本，对Kafka进行压测。Kafka压测时，可以查看到哪个地方出现了瓶颈（CPU，内存，网络IO）。

kafka-consumer-perf-test.sh

kafka-producer-perf-test.sh

### Kafka Producer压力测试

```shell
kafka-producer-perf-test.sh  --topic test --record-size 100 --num-records 100000 --throughput -1 --producer-props bootstrap.servers=hadoop12:9092,hadoop13:9092,hadoop14:9092
```

### Kafka Consumer压力测试

```shell
kafka-consumer-perf-test.sh --broker-list hadoop12:9092,hadoop13:9092,hadoop14:9092 --topic test --fetch-size 10000 --messages 10000000 --threads 1
```

