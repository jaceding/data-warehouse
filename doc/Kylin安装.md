# Kylin安装

Kylin 版本 3.0.2

安装 Kylin 前需要先部署好 Hadoop、Hive、Zk、HBase

## 上传压缩包并解压

1. 将 apache-kylin-3.0.2-bin.tar.gz 上传至 hadoop14 的 /opt/package 目录中

2. 解压到  /opt 目录下

   ```shell
   tar -zxvf /opt/package/apache-kylin-3.0.2-bin.tar.gz -C /opt
   ```

3. 重命名目录

   ```shell
   cd /opt && mv apache-kylin-3.0.2-bin kylin-3.0.2
   ```

## 环境变量准备

```shell
# java environment
JAVA_HOME=/usr/local/jdk1.8.0_231
JRE_HOME=$JAVA_HOME/jre
PATH=$PATH:$JAVA_HOME/bin:$JRE_home/bin:$PATH
CLASSPATH=.:$JAVA_HOME/lib/tools.jar:$JAVA_HOME/lib/dt.jar:$CLASSPATH
export JAVA_HOME
export JRE_HOME
export PATH
export CLASSPATH

# hadoop environment
export HADOOP_HOME=/opt/hadoop-3.1.3
export PATH=$PATH:$HADOOP_HOME/bin
export PATH=$PATH:$HADOOP_HOME/sbin

# kafka enviroment
export KAFKA_HOME=/opt/kafka_2.11-2.4.1
export PATH=$PATH:$KAFKA_HOME/bin

# hive enviroment
export HIVE_HOME=/opt/hive-3.1.2
export PATH=$PATH:$HIVE_HOME/bin

# spark enviroment
export SPARK_HOME=/opt/spark-3.0.0
export PATH=$PATH:$SPARK_HOME/bin

# hbase enviroment
export HBASE_HOME=/opt/hbase-2.0.5
export PATH=$PATH:$HBASE_HOME/bin
```

## 兼容性问题

修改 /opt/kylin-3.0.2/bin/find-spark-dependency.sh，排除冲突的jar包

```shell
vim  /opt/kylin-3.0.2/bin/find-spark-dependency.sh
# 修改为如下内容
spark_dependency=`find -L $spark_home/jars -name '*.jar' ! -name '*slf4j*' ! -name '*jackson*' ! -name '*metastore*' ! -name '*calcite*' ! -name '*doc*' ! -name '*test*' ! -name '*sources*' ''-printf '%p:' | sed 's/:$//'`
```

## 启动/关闭

```shell
cd /opt/kylin-3.0.2 && bin/kylin.sh start
cd /opt/kylin-3.0.2 && bin/kylin.sh stop
```

## 测试

访问：http://hadoop14:7070/

账号：ADMIN

密码：KYLIN