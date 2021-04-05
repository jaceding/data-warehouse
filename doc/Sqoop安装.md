# Sqoop安装

Sqoop 版本 1.4.6

## 下载并解压

1. 上传安装包 `sqoop-1.4.6.bin__hadoop-2.0.4-alpha.tar.gz` 到 hadoop12 的 `/opt/package` 路径中

2. 解压 sqoop 安装包到指定目录 

   ```shell
   tar -zxvf sqoop-1.4.6.bin__hadoop-2.0.4-alpha.tar.gz -C /opt
   ```

3. 重命名目录

   ```shell
   cd /opt && mv sqoop-1.4.6.bin__hadoop-2.0.4-alpha/ sqoop-1.4.6
   ```

## 修改配置文件

1. 进入到 `/op/sqoop-1.4.6/conf` 目录，重命名配置文件

   ```shell
   cd /op/sqoop-1.4.6/conf
   mv sqoop-env-template.sh sqoop-env.sh
   ```

2. 修改配置文件，`vim sqoop-env.sh`，添加如下内容：

   ```shell
   export HADOOP_COMMON_HOME=/opt/hadoop-3.1.3
   export HADOOP_MAPRED_HOME=/opt/hadoop-3.1.3
   export HIVE_HOME=/opt/hive-3.1.2
   export ZOOKEEPER_HOME=/opt/zookeeper-3.5.7
   export ZOOCFGDIR=/opt/zookeeper-3.5.7/conf
   ```

## 拷贝JDBC驱动

1. 将 `mysql-connector-java-8.0.18.jar` 上传到 `/opt/package` 路径

2. 进入到`/opt/package`路径，拷贝jdbc驱动到sqoop的lib目录下

   ```shell
   cp /opt/package/mysql-connector-java-8.0.18.jar /opt/sqoop-1.4.6/lib/
   ```

## 验证Sqoop

```shell
bin/sqoop help
```

## 测试连接数据库

```shell
bin/sqoop list-databases --connect jdbc:mysql://hadoop14:3306/ --username root --password jTc7Y9AFzMRp083Ubw5s
```

## 测试导入数据

```shell
bin/sqoop import \
--connect jdbc:mysql://hadoop14:3306/gmall \
--username root \
--password jTc7Y9AFzMRp083Ubw5s \
--table user_info \
--columns id,login_name \
--where "id >= 10" \
--target-dir /test \
--delete-target-dir \
--num-mappers 1 \
--fields-terminated-by '\t' \
--split-by id
```

## 业务数据导入HDFS

编写 `mysql_to_hdfs.sh` 脚本

- 初次导入

  ```shell
  mysql_to_hdfs.sh first 2020-05-15
  ```

- 每日导入

  ```shell
  mysql_to_hdfs.sh all 2020-05-16
  ```

  