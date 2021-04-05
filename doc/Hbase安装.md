# Hbase安装

Hbase 版本 2.0.5

安装 Hbase 前需要先部署好 Zk、Hadoop

## 上传压缩包并解压

1. 将 hbase-2.0.5-bin.tar.gz 上传至 hadoop12的 /opt/package 目录中

2. 解压到  /opt 目录下

   ```shell
   tar -zxvf /opt/package/hbase-2.0.5-bin.tar.gz -C /opt
   ```

## 配置

1. 修改 hbase-env.sh

   ```shell
   vim /opt/hbase-2.0.5/conf/hbase-env.sh
   export HBASE_MANAGES_ZK=false
   ```

2. 修改 hbase-site.xml

   ```shell
   vim /opt/hbase-2.0.5/conf/hbase-site.xml
   ```

   修改为：

   ```xml
   <configuration>
     <property>
       <name>hbase.rootdir</name>
       <value>hdfs://hadoop12:9820/hbase</value>
     </property>
     <property>
       <name>hbase.cluster.distributed</name>
       <value>true</value>
     </property>
     <property>
       <name>hbase.zookeeper.quorum</name>
       <value>hadoop12,hadoop13,hadoop14</value>
     </property>
   </configuration>
   ```

3. 修改 regionservers

   ```shell
   vim /opt/hbase-2.0.5/conf/regionservers
   # 添加如下内容
   hadoop12
   hadoop13
   hadoop14
   ```

4. 软连接 hadoop 配置文件到 HBase

   ```shell
   ln -s /opt/hadoop-3.1.3/etc/hadoop/core-site.xml /opt/hbase-2.0.5/conf/core-site.xml
   ln -s /opt/hadoop-3.1.3/etc/hadoop/hdfs-site.xml /opt/hbase-2.0.5/conf/hdfs-site.xml
   ```

## 复制给其他服务器

```shell
pdcp -a -r /opt/hbase-2.0.5 /opt
```

## 启动/关闭

### 方式一

```shell
# 启动
cd /opt/hbase-2.0.5 && bin/hbase-daemon.sh start master
cd /opt/hbase-2.0.5 && bin/hbase-daemon.sh start regionserver
# 关闭
cd /opt/hbase-2.0.5 && bin/hbase-daemon.sh stop master
cd /opt/hbase-2.0.5 && bin/hbase-daemon.sh stop regionserver
```

### 方式二

```shell
# 启动
cd /opt/hbase-2.0.5 && bin/start-hbase.sh
# 关闭
cd /opt/hbase-2.0.5 && bin/stop-hbase.sh
```

## 查看

http://hadoop12:16010