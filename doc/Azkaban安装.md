# Azkaban安装

Azkaban 版本 3.84.4

## 上传并解压

1. 上传 azkaban-db-3.84.4.tar.gz，azkaban-exec-server-3.84.4.tar.gz，azkaban-web-server-3.84.4.tar.gz 到 hadoop13的 /opt/package 路径

2. 解压 azkaban 安装包到指定目录

   ```shell
   cd /opt/package && tar -zxvf azkaban-db-3.84.4.tar.gz -C /opt/azkaban-3.84.4/
   cd /opt/package && tar -zxvf azkaban-exec-server-3.84.4.tar.gz -C /opt/azkaban-3.84.4/
   cd /opt/package && tar -zxvf azkaban-web-server-3.84.4.tar.gz  -C /opt/azkaban-3.84.3.4/
   ```

3. 重命名目录

   ```shell
   cd /opt/azkaban-3.84.4/ && mv azkaban-db-3.84.4 azkaban-db
   cd /opt/azkaban-3.84.4/ && mv azkaban-exec-server-3.84.4 azkaban-exec
   cd /opt/azkaban-3.84.4/ && mv azkaban-web-server-3.84.4 azkaban-web
   ```

## 配置MySQL

1. 新建数据库 `azkaban`

2. 创建 azkaban 用户，任何主机都可以访问，密码是 nuAUSik2pdEW8GQ4

   ```shell
   CREATE USER 'azkaban'@'%' IDENTIFIED BY 'nuAUSik2pdEW8GQ4';
   ```

3. 赋予 azkaban 用户增删改查权限

   ```shell
   GRANT SELECT,INSERT,UPDATE,DELETE ON azkaban.* to 'azkaban'@'%' WITH GRANT OPTION;
   ```

4. 初始化表

   ```shell
   # mysql执行 /opt/azkaban-3.84.4/azkaban-db-3.84.4/create-all-sql-3.84.4.sql
   ```

5. 修改 MySQL 包大小：防止Azkaban连接MySQL阻塞

   ```shell
   # 在[mysqld]下面加一行
   [mysqld]
   max_allowed_packet=1024M
   ```

6. 重启MySQ

## 配置Executor Server

Azkaban Executor Server 处理工作流和作业的实际执行。

1. 编辑 azkaban.properties

   ```shell
   vim /opt/module/azkaban/azkaban-exec/conf/azkaban.properties
   # 修改或添加以下属性
   default.timezone.id=Asia/Shanghai
   
   azkaban.webserver.url=http://hadoop13:8081
   
   executor.port=12321
   
   database.type=mysql
   mysql.port=3306
   mysql.host=hadoop14
   mysql.database=azkaban
   mysql.user=azkaban
   mysql.password=nuAUSik2pdEW8GQ4
   mysql.numconnections=100
   
   executor.metric.reports=true
   executor.metric.milisecinterval.default=60000
   ```

2. 替换对应的 mysql jdbc jar 包版本

   ```shell
   rm -f /opt/azkaban-3.84.4/azkaban-exec/lib/mysql-connector-java-5.1.28.jar
   mv /opt/package/mysql-connector-java-8.0.18.jar /opt/azkaban-3.84.4/azkaban-exec/lib/
   ```

3. 同步 azkaban-exec 到所有节点

   ```shell
   # 需要先在其他两台节点上创建 azkaban-3.84.4 目录
   pdcp -a -r /opt/azkaban-3.84.4/azkaban-exec /opt/azkaban-3.84.4/
   ```

4. 必须进入到 /opt/azkaban-3.84.4/azkaban-exec 路径，分别在三台机器上，启动 executor server

   ```shell
   pdsh -a "cd /opt/azkaban-3.84.4/azkaban-exec && bin/start-exec.sh"
   ```

5. 下面激活executor

   ```shell
   curl -G "hadoop12:$(<./executor.port)/executor?action=activate" && echo
   curl -G "hadoop13:$(<./executor.port)/executor?action=activate" && echo
   curl -G "hadoop14:$(<./executor.port)/executor?action=activate" && echo
   ```

## 配置Web Server

1. 编辑azkaban.properties

   ```shell
   vim /opt/azkaban-3.84.4/azkaban-web/conf/azkaban.properties
   # 添加或修改如下属性
   default.timezone.id=Asia/Shanghai
   
   database.type=mysql
   mysql.port=3306
   mysql.host=hadoop14
   mysql.database=azkaban
   mysql.user=azkaban
   mysql.password=nuAUSik2pdEW8GQ4
   mysql.numconnections=100
   
   azkaban.executorselector.filters=StaticRemainingFlowSize,CpuStatus
   
   ```

2. 替换对应的 mysql jdbc jar 包版本

   ```shell
   rm -f /opt/azkaban-3.84.4/azkaban-web/lib/mysql-connector-java-5.1.28.jar
   cp /opt/azkaban-3.84.4/azkaban-exec/lib/mysql-connector-java-8.0.18.jar /opt/azkaban-3.84.4/azkaban-web/lib/
   ```

3. 修改azkaban-users.xml文件，添加root用户

   ```shell
   vim /opt/azkaban-3.84.4/azkaban-web/conf/azkaban-users.xml
   # 如下
   <azkaban-users>
     <user groups="azkaban" password="azkaban" roles="admin" username="azkaban"/>
     <user password="metrics" roles="metrics" username="metrics"/>
     <user password="azkaban" roles="metrics,admin" username="root"/>
   
     <role name="admin" permissions="ADMIN"/>
     <role name="metrics" permissions="METRICS"/>
   </azkaban-users>
   ```

4. 启动web server

   ```shell
   cd /opt/azkaban-3.84.4/azkaban-web/ && bin/start-web.sh
   ```

5. 访问http://hadoop13:8081,并用root用户登陆

## 启动/停止命令

### 启动 exec

```shell
pdsh -a "cd /opt/azkaban-3.84.4/azkaban-exec && bin/start-exec.sh"
```

### 停止 exec

```shell
pdsh -a "cd /opt/azkaban-3.84.4/azkaban-exec && bin/shutdown-exec.sh"
```

### 激活 executor

```shell
curl -G "hadoop12:$(<./executor.port)/executor?action=activate" && echo
curl -G "hadoop13:$(<./executor.port)/executor?action=activate" && echo
curl -G "hadoop14:$(<./executor.port)/executor?action=activate" && echo
```

### 启动 web

在 hadoop13 执行

```shell
cd /opt/azkaban-3.84.4/azkaban-web/ && bin/start-web.sh
```

### 关闭 web

在 hadoop13 执行

```shell
cd /opt/azkaban-3.84.4/azkaban-web/ && bin/shutdown-web.sh
```

