# Hadoop安装

Hadoop 版本 3.1.3

## 集群规划

|      |     hadoop12      |           hadoop13           |          hadoop14          |
| :--: | :---------------: | :--------------------------: | :------------------------: |
| HDFS | NameNode DataNode |           DataNode           | SecondaryNameNode DataNode |
| YARN |    NodeManager    | ResourceManager  NodeManager |        NodeManager         |

注意：

- NameNode、SecondaryNameNode 不要安装在同一台服务器
- ResourceManager 也很消耗内存，不要和 NameNode、SecondaryNameNode 配置在同一台机器上

## 解压压缩包

```shell
# 先将 hadoop-3.1.3.tar.gz 上传至 /opt/package 目录中
# pdsh 解压至相应目录
pdsh -a "cd /opt/package && tar -zxvf hadoop-3.1.3.tar.gz -C /opt"
```

## 配置环境变量

```shell
vim /etc/profile.d/my_env.sh
# 添加如下内容
# hadoop environment
export HADOOP_HOME=/opt/hadoop-3.1.3
export PATH=$PATH:$HADOOP_HOME/bin
export PATH=$PATH:$HADOOP_HOME/sbin
```

## 配置集群

### 核心配置文件

配置core-site.xml文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>

<configuration>
	<!-- 指定NameNode的地址 -->
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://hadoop12:9820</value>
    </property>
    <!-- 指定hadoop数据的存储目录 -->
    <property>
        <name>hadoop.tmp.dir</name>
        <value>/opt/hadoop-3.1.3/data</value>
    </property>
    <!-- 配置HDFS网页登录使用的静态用户为root -->
    <property>
        <name>hadoop.http.staticuser.user</name>
        <value>root</value>
    </property>
    <!-- 配置该root(superUser)允许通过代理访问的主机节点 -->
    <property>
        <name>hadoop.proxyuser.root.hosts</name>
        <value>*</value>
    </property>
    <!-- 配置该root(superUser)允许通过代理用户所属组 -->
    <property>
        <name>hadoop.proxyuser.root.groups</name>
        <value>*</value>
    </property>
    <!-- 配置该root(superUser)允许通过代理的用户-->
    <property>
        <name>hadoop.proxyuser.root.users</name>
        <value>*</value>
    </property>
</configuration>
```

### HDFS配置文件

配置hadoop-env.sh，末尾添加

```
export JAVA_HOME=/usr/local/jdk1.8.0_231
export HDFS_NAMENODE_USER="root"
export HDFS_DATANODE_USER="root"
export HDFS_SECONDARYNAMENODE_USER="root"
export YARN_RESOURCEMANAGER_USER="root"
export YARN_NODEMANAGER_USER="root"
```

配置hdfs-site.xml文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>

<configuration>
	<!-- nn web端访问地址-->
	<property>
        <name>dfs.namenode.http-address</name>
        <value>hadoop12:9870</value>
    </property>
    
	<!-- 2nn web端访问地址-->
    <property>
        <name>dfs.namenode.secondary.http-address</name>
        <value>hadoop14:9868</value>
    </property>
    
    <!-- 指定HDFS副本的数量1 -->
    <property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>
</configuration>
```

### YARN配置文件

配置yarn-env.sh，末尾加上

```
export JAVA_HOME=/usr/local/jdk1.8.0_231
```

配置yarn-site.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>

<configuration>
	<!-- 指定MR走shuffle -->
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
    
    <!-- 指定ResourceManager的地址-->
    <property>
        <name>yarn.resourcemanager.hostname</name>
        <value>hadoop13</value>
    </property>
    
    <!-- 环境变量的继承 -->
    <property>
        <name>yarn.nodemanager.env-whitelist</name>
           <value>JAVA_HOME,HADOOP_COMMON_HOME,HADOOP_HDFS_HOME,HADOOP_CONF_DIR,CLASSPATH_PREPEND_DISTCACHE,HADOOP_YARN_HOME,HADOOP_MAPRED_HOME</value>
    </property>
    
    <!-- yarn容器允许分配的最大最小内存 -->
    <property>
        <name>yarn.scheduler.minimum-allocation-mb</name>
        <value>512</value>
    </property>
    <property>
        <name>yarn.scheduler.maximum-allocation-mb</name>
        <value>4096</value>
    </property>
    
    <!-- yarn容器允许管理的物理内存大小 -->
    <property>
        <name>yarn.nodemanager.resource.memory-mb</name>
        <value>4096</value>
    </property>
    
    <!-- 关闭yarn对虚拟内存的限制检查 -->
    <property>
        <name>yarn.nodemanager.pmem-check-enabled</name>
        <value>false</value>
    </property>
    <property>
        <name>yarn.nodemanager.vmem-check-enabled</name>
        <value>false</value>
    </property>
</configuration>
```

### MapReduce配置文件

配置mapred-env.sh，末尾加上

```
export JAVA_HOME=/usr/local/jdk1.8.0_231
```

配置mapred-site.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>

<configuration>
	<!-- 指定MapReduce程序运行在Yarn上 -->
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
</configuration>
```

### 配置workers

```shell
vim /opt/hadoop-3.1.3/etc/hadoop/workers
# 添加如下内容
hadoop12
hadoop13
hadoop14
```

### 配置历史服务器

为了查看程序的历史运行情况，需要配置一下历史服务器。

配置mapred-site.xml，添加如下内容：

```xml
<!-- 历史服务器端地址 -->
<property>
    <name>mapreduce.jobhistory.address</name>
    <value>hadoop12:10020</value>
</property>

<!-- 历史服务器web端地址 -->
<property>
    <name>mapreduce.jobhistory.webapp.address</name>
    <value>hadoop12:19888</value>
</property>
```

### 配置日志的聚集

日志聚集概念：应用运行完成以后，将程序运行日志信息上传到HDFS系统上。

日志聚集功能好处：可以方便的查看到程序运行详情，方便开发调试。

注意：开启日志聚集功能，需要重新启动NodeManager 、ResourceManager和HistoryManager。

配置yarn-site.xml，添加如下内容：

```xml
<!-- 开启日志聚集功能 -->
<property>
    <name>yarn.log-aggregation-enable</name>
    <value>true</value>
</property>

<!-- 设置日志聚集服务器地址 -->
<property>  
    <name>yarn.log.server.url</name>  
    <value>http://hadoop12:19888/jobhistory/logs</value>
</property>

<!-- 设置日志保留时间为7天 -->
<property>
    <name>yarn.log-aggregation.retain-seconds</name>
    <value>604800</value>
</property>
```

### 复制配置文件到其他服务器

```shell
pdcp -a /opt/hadoop-3.1.3/etc/hadoop/core-site.xml /opt/hadoop-3.1.3/etc/hadoop/core-site.xml
pdcp -a /opt/hadoop-3.1.3/etc/hadoop/hadoop-env.sh /opt/hadoop-3.1.3/etc/hadoop/hadoop-env.sh
pdcp -a /opt/hadoop-3.1.3/etc/hadoop/hdfs-site.xml /opt/hadoop-3.1.3/etc/hadoop/hdfs-site.xml
pdcp -a /opt/hadoop-3.1.3/etc/hadoop/yarn-env.sh /opt/hadoop-3.1.3/etc/hadoop/yarn-env.sh
pdcp -a /opt/hadoop-3.1.3/etc/hadoop/yarn-site.xml /opt/hadoop-3.1.3/etc/hadoop/yarn-site.xml
pdcp -a /opt/hadoop-3.1.3/etc/hadoop/mapred-env.sh /opt/hadoop-3.1.3/etc/hadoop/mapred-env.sh
pdcp -a /opt/hadoop-3.1.3/etc/hadoop/mapred-site.xml /opt/hadoop-3.1.3/etc/hadoop/mapred-site.xml
pdcp -a /opt/hadoop-3.1.3/etc/hadoop/workers /opt/hadoop-3.1.3/etc/hadoop/workers
```

## 启动集群

### 格式化NameNode

如果集群是第一次启动，需要格式化NameNode。

注意：格式化之前，一定要先停止上次启动的所有namenode和datanode进程，然后再删除data和log数据

```shell
# 格式化namenode(只需在hadoop12上执行)
hadoop namenode -format
```

### 启动DFS集群

```shell
# 启动DFS集群(只需在hadoop12上执行)
start-dfs.sh
```

## LZO压缩

```shell
# 将 hadoop-lzo-0.4.20.jar 上传至 /opt/package
# 复制hadoop-lzo-0.4.20.jar到其他服务器
pdcp -a /opt/package/hadoop-lzo-0.4.20.jar /opt/hadoop-3.1.3/share/hadoop/common
```

### 配置参数

core-site.xml增加配置支持LZO压缩

```shell
<configuration>
    <property>
        <name>io.compression.codecs</name>
        <value>
            org.apache.hadoop.io.compress.GzipCodec,
            org.apache.hadoop.io.compress.DefaultCodec,
            org.apache.hadoop.io.compress.BZip2Codec,
            org.apache.hadoop.io.compress.SnappyCodec,
            com.hadoop.compression.lzo.LzoCodec,
            com.hadoop.compression.lzo.LzopCodec
        </value>
    </property>

    <property>
        <name>io.compression.codec.lzo.class</name>
        <value>com.hadoop.compression.lzo.LzoCodec</value>
    </property>
</configuration>
```

### 复制配置文件到其他服务器

```shell
pdcp -a /opt/hadoop-3.1.3/etc/hadoop/core-site.xml /opt/hadoop-3.1.3/etc/hadoop/core-site.xml
```

## 基准测试

### 测试写入性能

测试向HDFS集群写10个128M的文件

```shell
hadoop jar /opt/hadoop-3.1.3/share/hadoop/mapreduce/hadoop-mapreduce-client-jobclient-3.1.3-tests.jar TestDFSIO -write -nrFiles 10 -fileSize 128MB
```

### 测试HDFS读性能

测试读取HDFS集群10个128M的文件

```shell
hadoop jar /opt/hadoop-3.1.3/share/hadoop/mapreduce/hadoop-mapreduce-client-jobclient-3.1.3-tests.jar TestDFSIO -read -nrFiles 10 -fileSize 128MB
```

### 删除测试生成数据

```shell
hadoop jar /opt/hadoop-3.1.3/share/hadoop/mapreduce/hadoop-mapreduce-client-jobclient-3.1.3-tests.jar TestDFSIO -clean
```

## 相关命令

### 启动/关闭HDFS

```shell
start-dfs.sh
stop-dfs.sh
```

### 启动/关闭YARN

```shell
start-yarn.sh
stop-yarn.sh
```