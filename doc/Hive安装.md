# Hive安装

Hadoop 版本  3.1.2

## 上传压缩包并解压

1. 上传安装包 `apache-hive-3.1.2-bin.tar.gz` 到 hadoop14 的 `/opt/package` 路径中

2. 解压  hive 安装包到指定目录 

   ```shell
   tar -zxvf /opt/package/apache-hive-3.1.2-bin.tar.gz -C /opt
   ```

3. 重命名目录

   ```shell
   cd /opt && mv apache-hive-3.1.2-bin hive-3.1.2
   ```

## 配置环境变量

1. 修改 `/etc/profile.d/my_env.sh`

   ```shell
   vim /etc/profile.d/my_env.sh
   ```

2. 添加环境变量

   ```shell
   # hive enviroment
   export HIVE_HOME=/opt/hive-3.1.2
   export PATH=$PATH:$HIVE_HOME/bin
   ```

3. 使环境变量生效：`source /etc/profile`

## 解决jar包冲突

不解决也不影响正常使用

```shell
mv /opt/hive-3.1.2/lib/log4j-slf4j-impl-2.10.0.jar /opt/hive-3.1.2/lib/log4j-slf4j-impl-2.10.0.jar.bak
```

## Hive元数据配置到MySQL

### 拷贝驱动

将MySQL的JDBC驱动拷贝到Hive的lib目录下

```shell
cp /opt/package/mysql-connector-java-8.0.18.jar /opt/hive-3.1.2/lib/
```

### 配置Metastore

1. 在 `/opt/hive-3.1.2/conf` 目录下新建 `hive-site.xml` 文件

   ```shell
   cd /opt/hive-3.1.2/conf && vim hive-site.xml
   ```

2. 添加如下内容

   ```xml
   <?xml version="1.0"?>
   <?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
   <configuration>
       <property>
           <name>javax.jdo.option.ConnectionURL</name>
           <value>jdbc:mysql://hadoop14:3306/hive?useSSL=false</value>
       </property>
   
       <property>
           <name>javax.jdo.option.ConnectionDriverName</name>
           <value>com.mysql.cj.jdbc.Driver</value>
       </property>
   
       <property>
           <name>javax.jdo.option.ConnectionUserName</name>
           <value>root</value>
       </property>
   
       <property>
           <name>javax.jdo.option.ConnectionPassword</name>
           <value>jTc7Y9AFzMRp083Ubw5s</value>
       </property>
   
       <property>
           <name>hive.metastore.warehouse.dir</name>
           <value>/user/hive/warehouse</value>
       </property>
   
       <property>
           <name>hive.metastore.schema.verification</name>
           <value>false</value>
       </property>
   
       <property>
       <name>hive.server2.thrift.port</name>
       <value>10000</value>
       </property>
   
       <property>
           <name>hive.server2.thrift.bind.host</name>
           <value>hadoop14</value>
       </property>
   
       <property>
           <name>hive.metastore.event.db.notification.api.auth</name>
           <value>false</value>
       </property>
       
       <property>
           <name>hive.cli.print.header</name>
           <value>true</value>
       </property>
   
       <property>
           <name>hive.cli.print.current.db</name>
           <value>true</value>
       </property>
   </configuration>
   ```

### 初始化Hive

1. 给 mysql 新建数据库：hive，注意修改数据编码为 latin1

2. 初始化 Hive 元数据库

   ```shell
   cd /opt/hive-3.1.2/bin && schematool -initSchema -dbType mysql -verbose
   ```

## 启动Hive客户端

```shell
cd /opt/hive-3.1.2/ && bin/hive
# 查看一下数据库
show databases;
```

## Hive on Spark

将 Hive 默认的引擎替换成 Spark

### 兼容性说明

注意：官网下载的Hive3.1.2和Spark3.0.0默认是不兼容的。因为Hive3.1.2支持的Spark版本是2.4.5，所以重新编译Hive3.1.2版本。

编译步骤：官网下载Hive3.1.2源码，修改pom文件中引用的Spark版本为3.0.0，如果编译通过，直接打包获取jar包。如果报错，就根据提示，修改相关方法，直到不报错，打包获取jar包。

### 部署Spark

在Hive所在节点部署Spark

步骤如下：

1. 上传 `spark-3.0.0-bin-hadoop3.2.tgz` 压缩包至 `/opt/package` 目录，并解压，修改 spark 目录

   ```shell
   tar -zxvf /opt/package/spark-3.0.0-bin-hadoop3.2.tgz -C /opt
   mv /opt/spark-3.0.0-bin-hadoop3.2 /opt/spark-3.0.0
   ```

2. 配置 SPARK_HOME 环境变量

   ```shell
   vim /etc/profile.d/my_env.sh
   ```

   添加如下内容：

   ```shell
   # spark enviroment
   export SPARK_HOME=/opt/spark-3.0.0
   export PATH=$PATH:$SPARK_HOME/bin
   ```

3. 使环境变量生效：`source /etc/profile`

### 配置

1. 在hive中创建spark配置文件

   ```shell
   vim /opt/hive-3.1.2/conf/spark-defaults.conf
   ```

2. 添加如下内容（在执行任务时，会根据如下参数执行）

   ```shell
   spark.master                             yarn
   spark.eventLog.enabled                   true
   spark.eventLog.dir                       hdfs://hadoop12:9820/spark-history
   spark.executor.memory                    1g
   spark.driver.memory					     1g
   ```

3. 在HDFS创建如下路径，用于存储历史日志

   ```shell
   hadoop fs -mkdir /spark-history
   ```

### 向HDFS上传Spark纯净版jar包

说明1：由于Spark3.0.0非纯净版默认支持的是hive2.3.7版本，直接使用会和安装的Hive3.1.2出现兼容性问题。所以采用Spark纯净版jar包，不包含hadoop和hive相关依赖，避免冲突。

说明2：Hive任务最终由Spark来执行，Spark任务资源分配由Yarn来调度，该任务有可能被分配到集群的任何一个节点。所以需要将Spark的依赖上传到HDFS集群路径，这样集群中任何一个节点都能获取到。

1. 上传并解压 `spark-3.0.0-bin-without-hadoop.tgz`

   ```shell
   tar -zxvf /opt/package/spark-3.0.0-bin-without-hadoop.tgz
   ```

2. 上传Spark纯净版jar包到HDFS

   ```shell
   hadoop fs -mkdir /spark-jars
   hadoop fs -put /opt/package/spark-3.0.0-bin-without-hadoop/jars/* /spark-jars
   ```

### 修改hive-site.xml文件

1. 修改hive-site.xml文件

   ```shell
   vim /opt/hive-3.1.2/conf/hive-site.xml
   ```

2. 添加如下内容：

   ```xml
   <!--Spark依赖位置（注意：端口号9820必须和namenode的端口号一致）-->
   <property>
       <name>spark.yarn.jars</name>
       <value>hdfs://hadoop12:9820/spark-jars/*</value>
   </property>
     
   <!--Hive执行引擎-->
   <property>
       <name>hive.execution.engine</name>
       <value>spark</value>
   </property>
   
   <!--Hive和Spark连接超时时间-->
   <property>
       <name>hive.spark.client.connect.timeout</name>
       <value>10000ms</value>
   </property>
   ```

### Hive on Spark测试

1. 启动hive客户端

   ```shell
   cd /opt/hive-3.1.2/ && bin/hive
   ```

2. 创建一张测试表

   ```shell
   create table student(id int, name string);
   ```

3. 通过insert测试效果

   ```shell
   insert into table student values(1,'abc');
   ```

### 增加ApplicationMaster资源比例

Yarn默认调度器为Capacity Scheduler（容量调度器），且默认只有一个队列——default。如果队列中执行第一个任务资源不够，就不会再执行第二个任务，一直等到第一个任务执行完毕。

针对容量调度器并发度低的问题，考虑调整 `yarn.scheduler.capacity.maximum-am-resource-percent` 该参数。默认值是0.1，表示集群上AM最多可使用的资源比例，目的为限制过多的app数量。

1. 修改配置文件

   ```shell
   vim /opt/hadoop-3.1.3/etc/hadoop/capacity-scheduler.xml
   ```

2. 修改如下参数

   ```xml
     <property>
       <name>yarn.scheduler.capacity.maximum-am-resource-percent</name>
       <value>0.5</value>
       <description>
         Maximum percent of resources in the cluster which can be used to run 
         application masters i.e. controls number of concurrent running
         applications.
       </description>
     </property>
   ```

3. 复制配置文件到其他服务器

   ```shell
   pdcp -a /opt/hadoop-3.1.3/etc/hadoop/capacity-scheduler.xml /opt/hadoop-3.1.3/etc/hadoop/capacity-scheduler.xml
   ```

### 配置Yarn容量调度器多队列

1. 修改容量调度器配置文件

   ```shell
   vim /opt/hadoop-3.1.3/etc/hadoop/capacity-scheduler.xml
   ```

2. 添加如下内容

   ```xml
   <property>
       <name>yarn.scheduler.capacity.root.queues</name>
       <value>default,hive</value>
       <description>
           再增加一个hive队列
       </description>
   </property>
   
   <property>
       <name>yarn.scheduler.capacity.root.default.capacity</name>
       <value>50</value>
       <description>
           default队列的容量为50%
       </description>
   </property>
   
   <property>
       <name>yarn.scheduler.capacity.root.hive.capacity</name>
       <value>50</value>
       <description>
           hive队列的容量为50%
       </description>
   </property>
   
   <property>
       <name>yarn.scheduler.capacity.root.hive.user-limit-factor</name>
       <value>1</value>
       <description>
           一个用户最多能够获取该队列资源容量的比例，取值0-1
       </description>
   </property>
   
   <property>
       <name>yarn.scheduler.capacity.root.hive.maximum-capacity</name>
       <value>80</value>
       <description>
           hive队列的最大容量（自己队列资源不够，可以使用其他队列资源上限）
       </description>
   </property>
   
   <property>
       <name>yarn.scheduler.capacity.root.hive.state</name>
       <value>RUNNING</value>
       <description>
           开启hive队列运行，不设置队列不能使用
       </description>
   </property>
   
   <property>
       <name>yarn.scheduler.capacity.root.hive.acl_submit_applications</name>
       <value>*</value>
       <description>
           访问控制，控制谁可以将任务提交到该队列,*表示任何人
       </description>
   </property>
   
   <property>
       <name>yarn.scheduler.capacity.root.hive.acl_administer_queue</name>
       <value>*</value>
       <description>
           访问控制，控制谁可以管理(包括提交和取消)该队列的任务，*表示任何人
       </description>
   </property>
   
   <property>
       <name>yarn.scheduler.capacity.root.hive.acl_application_max_priority</name>
       <value>*</value>
       <description>
           指定哪个用户可以提交配置任务优先级
       </description>
   </property>
   
   <property>
       <name>yarn.scheduler.capacity.root.hive.maximum-application-lifetime</name>
       <value>-1</value>
       <description>
           hive队列中任务的最大生命时长，以秒为单位。任何小于或等于零的值将被视为禁用。
       </description>
   </property>
   
   <property>
       <name>yarn.scheduler.capacity.root.hive.default-application-lifetime</name>
       <value>-1</value>
       <description>
           hive队列中任务的默认生命时长，以秒为单位。任何小于或等于零的值将被视为禁用。
       </description>
   </property>
   ```

3. 复制配置文件到其他服务器

   ```shell
   pdcp -a /opt/hadoop-3.1.3/etc/hadoop/capacity-scheduler.xml /opt/hadoop-3.1.3/etc/hadoop/capacity-scheduler.xml
   ```

4. 重启 Hadoop 集群

5. 测试：提交一个MR任务，并指定队列为hive

   ```shell
   hadoop jar /opt/hadoop-3.1.3/share/hadoop/mapreduce/hadoop-mapreduce-examples-3.1.3.jar pi -Dmapreduce.job.queuename=hive 1 1
   ```

   