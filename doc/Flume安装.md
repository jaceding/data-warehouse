# Flume安装

Flume 版本 1.9.0

## 部署规划

|                   | hadoop12 | hadoop13 | hadoop14 |
| :---------------: | :------: | :------: | :------: |
| Flume（采集日志） |  Flume   |  Flume   |          |
| Flume（消费日志） |          |          |  Flume   |

## 安装Flume

```shell
# 将 apache-flume-1.9.0-bin.tar.gz 上传至 /opt/package 目录中
# 解压
tar -zxvf apache-flume-1.9.0-bin.tar.gz -C /opt

# 修改目录名
cd /opt && mv apache-flume-1.9.0-bin flume-1.9.0

# 将lib文件夹下的guava-11.0.2.jar删除以兼容Hadoop 3.1.3，否则会报java.lang.ClassNotFoundException异常
# 一定要配置hadoop环境变量，否则会报java.lang.ClassNotFoundException异常，或者将Hadoop中的guava复制到flume中
rm /opt/flume-1.9.0/lib/guava-11.0.2.jar

# 将flume/conf下的flume-env.sh.template文件修改为flume-env.sh，并配置flume-env.sh文件
mv /opt/flume-1.9.0/conf/flume-env.sh.template /opt/flume-1.9.0/conf/flume-env.sh

# 修改flume-env.sh
vim /opt/flume-1.9.0/conf/flume-env.sh
# 添加如下内容：
export JAVA_HOME=/usr/local/jdk1.8.0_231

# 复制给其他服务器
pdcp -a -r /opt/flume-1.9.0 /opt
```

## 采集日志Flume

Flume直接读log日志的数据，log日志的格式是app.yyyy-mm-dd.log。

采集日志时，需要自定义拦截器过滤掉格式不正确的数据。

### 配置

在hadoop12、hadoop13分别：

1. 添加并修改配置文件

   ```shell
   vim /opt/flume-1.9.0/conf/file-flume-kafka.conf
   ```

2. 在文件配置中添加如下内容：

   ```conf
   #为各组件命名
   a1.sources = r1
   a1.channels = c1
   
   #描述source
   a1.sources.r1.type = TAILDIR
   a1.sources.r1.filegroups = f1
   a1.sources.r1.filegroups.f1 = /opt/applog/log/app.*
   a1.sources.r1.positionFile = /opt/flume-1.9.0/taildir_position.json
   a1.sources.r1.interceptors =  i1
   a1.sources.r1.interceptors.i1.type = per.jaceding.flume.interceptor.EtlInterceptor$Builder
   
   #描述channel
   a1.channels.c1.type = org.apache.flume.channel.kafka.KafkaChannel
   a1.channels.c1.kafka.bootstrap.servers = hadoop12:9092,hadoop13:9092,hadoop14:9092
   a1.channels.c1.kafka.topic = topic_log
   a1.channels.c1.parseAsFlumeEvent = false
   
   #绑定source和channel以及sink和channel的关系
   a1.sources.r1.channels = c1
   ```

### 拦截器

1. 创建Maven工程flume-interceptor

2. 创建包名：per.jaceding.flume.interceptor

3. 在pom.xml文件中添加如下配置

   ```xml
   <dependencies>
       <dependency>
           <groupId>org.apache.flume</groupId>
           <artifactId>flume-ng-core</artifactId>
           <version>1.9.0</version>
           <scope>provided</scope>
       </dependency>
   
       <dependency>
           <groupId>cn.hutool</groupId>
           <artifactId>hutool-json</artifactId>
           <version>5.5.7</version>
       </dependency>
   </dependencies>
   
   <build>
       <plugins>
           <plugin>
               <artifactId>maven-compiler-plugin</artifactId>
               <version>2.3.2</version>
               <configuration>
                   <source>1.8</source>
                   <target>1.8</target>
               </configuration>
           </plugin>
           <plugin>
               <artifactId>maven-assembly-plugin</artifactId>
               <configuration>
                   <descriptorRefs>
                       <descriptorRef>jar-with-dependencies</descriptorRef>
                   </descriptorRefs>
               </configuration>
               <executions>
                 <execution>
                       <id>make-assembly</id>
                       <phase>package</phase>
                       <goals>
                           <goal>single</goal>
                       </goals>
                   </execution>
               </executions>
           </plugin>
       </plugins>
   </build>
   ```

4. 在 `per.jaceding.flume.interceptor` 包中创建 `EtlInterceptor`

   ```java
   import cn.hutool.json.JSONUtil;
   import org.apache.flume.Context;
   import org.apache.flume.Event;
   import org.apache.flume.interceptor.Interceptor;
   
   import java.nio.charset.StandardCharsets;
   import java.util.List;
   
   /**
    * 过滤不是json格式的数据
    *
    * @author jaceding
    * @date 2021/3/23
    */
   public class EtlInterceptor implements Interceptor {
   
       @Override
       public void initialize() {
   
       }
   
       @Override
       public Event intercept(Event event) {
           byte[] body = event.getBody();
           String log = new String(body, StandardCharsets.UTF_8);
   
           try {
               JSONUtil.parse(log);
               return event;
           } catch (Throwable e) {
               return null;
           }
       }
   
       @Override
       public List<Event> intercept(List<Event> events) {
   
           events.removeIf(next -> intercept(next) == null);
   
           return events;
       }
   
       @Override
       public void close() {
   
       }
   
       public static class Builder implements Interceptor.Builder {
   
           @Override
           public Interceptor build() {
               return new EtlInterceptor();
           }
   
           @Override
           public void configure(Context context) {
   
           }
       }
   }
   ```

5. 打包：`mvn clean install` ，将 `flume-interceptor-1.0-jar-with-dependencies.jar` 上传至 hadoop12、hadoop13 的 `/opt/flume-1.9.0/lib` 目录中

### 启动

分别在hadoop12、hadoop13上启动Flume

```shell
/opt/flume-1.9.0/bin/flume-ng agent --name a1 --conf-file /opt/flume-1.9.0/conf/file-flume-kafka.conf &
```

## 消费日志Flume

读取kafka数据，写入HDFS。

自定义拦截器实现根据日志里面的实际时间，发往HDFS的不同路径。

### 配置

在 hadoop14 分别

1. 添加并修改配置文件

   ```shell
   vim /opt/flume-1.9.0/conf/file-flume-kafka.conf
   ```

2. 在文件配置中添加如下内容：

   ```conf
   ## 组件
   a1.sources=r1
   a1.channels=c1
   a1.sinks=k1
   
   ## source1
   a1.sources.r1.type = org.apache.flume.source.kafka.KafkaSource
   a1.sources.r1.batchSize = 5000
   a1.sources.r1.batchDurationMillis = 2000
   a1.sources.r1.kafka.bootstrap.servers = hadoop12:9092,hadoop13:9092,hadoop14:9092
   a1.sources.r1.kafka.topics=topic_log
   a1.sources.r1.interceptors = i1
   a1.sources.r1.interceptors.i1.type = per.jaceding.flume.interceptor.TsInterceptor$Builder
   
   ## channel1
   a1.channels.c1.type = file
   a1.channels.c1.checkpointDir = /opt/flume-1.9.0/checkpoint/behavior1
   a1.channels.c1.dataDirs = /opt/flume-1.9.0/data/behavior1/
   a1.channels.c1.maxFileSize = 2146435071
   a1.channels.c1.capacity = 1000000
   a1.channels.c1.keep-alive = 6
   
   
   ## sink1
   a1.sinks.k1.type = hdfs
   a1.sinks.k1.hdfs.path = /origin_data/gmall/log/topic_log/%Y-%m-%d
   a1.sinks.k1.hdfs.filePrefix = log-
   a1.sinks.k1.hdfs.round = false
   
   
   a1.sinks.k1.hdfs.rollInterval = 10
   a1.sinks.k1.hdfs.rollSize = 134217728
   a1.sinks.k1.hdfs.rollCount = 0
   
   ## 控制输出文件是原生文件。
   a1.sinks.k1.hdfs.fileType = CompressedStream
   a1.sinks.k1.hdfs.codeC = lzop
   
   ## 拼装
   a1.sources.r1.channels = c1
   a1.sinks.k1.channel= c1
   ```

### 拦截器

在 `per.jaceding.flume.interceptor` 包中创建 `TslInterceptor`

```java
import cn.hutool.core.util.StrUtil;
import cn.hutool.json.JSONObject;
import cn.hutool.json.JSONUtil;
import org.apache.flume.Context;
import org.apache.flume.Event;
import org.apache.flume.interceptor.Interceptor;

import java.nio.charset.StandardCharsets;
import java.util.List;
import java.util.Map;
import java.util.Objects;
import java.util.stream.Collectors;

/**
 * 给 Event 添加 timestamp 字段
 *
 * @author jaceding
 * @date 2021/3/23
 */
public class TsInterceptor implements Interceptor {

    @Override
    public void initialize() {

    }

    @Override
    public Event intercept(Event event) {
        Map<String, String> headers = event.getHeaders();
        String log = new String(event.getBody(), StandardCharsets.UTF_8);

        try {
            JSONObject jsonObject = JSONUtil.parseObj(log);
            String ts = jsonObject.getStr("ts");

            if (!StrUtil.isBlank(ts)) {
                headers.put("timestamp", ts);
                return event;
            }
        } catch (Exception ignored) {

        }
        return null;
    }

    @Override
    public List<Event> intercept(List<Event> events) {
        return events.stream().map(this::intercept).filter(Objects::nonNull).collect(Collectors.toList());
    }

    @Override
    public void close() {

    }

    public static class Builder implements Interceptor.Builder {

        @Override
        public Interceptor build() {
            return new TsInterceptor();
        }

        @Override
        public void configure(Context context) {
        }
    }
}
```

打包：`mvn clean install` ，将 `flume-interceptor-1.0-jar-with-dependencies.jar` 上传至 hadoop14 的 `/opt/flume-1.9.0/lib` 目录中

### 启动

分别在 hadoop14 上启动Flume

```shell
/opt/flume-1.9.0/bin/flume-ng agent --name a1 --conf-file /opt/flume-1.9.0/conf/kafka-flume-hdfs.conf &
```



