---
layout: post
title:  安装Azkaban
date: 2014-08-25T08:00:00+08:00
keywords: azkaban 
categories: [  devops ]
tags: [azkaban]
---

Azkaban 是由 Linkedin 开源的一个批量工作流任务调度器。用于在一个工作流内以一个特定的顺序运行一组工作和流程。Azkaban 定义了一种 KV 文件格式来建立任务之间的依赖关系，并提供一个易于使用的 web 用户界面维护和跟踪你的工作流。

- Azkaban 官网地址：<http://azkaban.github.io/>
- Azkaban 的下载地址：<http://azkaban.github.io/downloads.html>

Azkaban 包括三个关键组件：

- `关系数据库`：使用 Mysql数据库，主要用于保存流程、权限、任务状态、任务计划等信息。
- `AzkabanWebServer`：为用户提供管理留存、任务计划、权限等功能。
- `AzkabanExecutorServer`：执行任务，并把任务执行的输出日志保存到 Mysql；可以同时启动多个 AzkabanExecutorServer ，他们通过 mysql 获取流程状态来协调工作。

![](http://azkaban.github.io/azkaban/docs/2.5/images/azkaban2overviewdesign.png)

在 2.5 版本之后，Azkaban 提供了两种模式来安装：

- 一种是 standalone 的 “solo-server” 模式；
- 另一种是两个 server 的模式，分别为 AzkabanWebServer 和 AzkabanExecutorServer。

这里主要介绍第二种模式的安装方法，安装的版本为3.80.1。

# 编译源码

```bash
wget https://github.com/azkaban/azkaban/archive/3.80.1.zip

./gradlew build
./gradlew installDist
```

编译后的文件：

```bash
azkaban-exec-server/build/distributions/azkaban-exec-server-0.1.0-SNAPSHOT.zip
azkaban-web-server/build/distributions/azkaban-web-server-0.1.0-SNAPSHOT.zip
```

# 安装 MySql

目前 Azkaban 只支持 MySql ，故需安装 MySql 服务器，安装 MySql 的过程这里不作介绍。

安装之后，创建 azkaban 数据库，并创建 azkaban 用户，密码为 azkaban，并设置权限。

~~~sql
mysql> CREATE DATABASE azkaban;
mysql> CREATE USER 'azkaban'@'%' IDENTIFIED BY 'azkaban';
mysql> GRANT all ON azkaban.* to 'azkaban'@'%' WITH GRANT OPTION;
mysql> flush privileges;
~~~

修改 `/etc/my.cnf` 文件，设置 `max_allowed_packet` 值：

~~~
[mysqld]
...
max_allowed_packet=1024M
~~~

然后重启 MySql。

# 安装 azkaban-web-server

解压缩 azkaban-web-server-0.1.0-SNAPSHOT.zip。

```bash
unzip azkaban-web-server-0.1.0-SNAPSHOT.zip
mv azkaban-web-server-0.1.0-SNAPSHOT /opt/azkaban-web-server
```

创建 SSL 配置，命令：

```bash
keytool -keystore keystore -alias jetty -genkey -keyalg RSA
```

密码均输入azkaban，最后在当前目录生成 keystore 证书文件。



将 keystore 考贝到 azkaban web 目录中

```bash
mv keystore /opt/azkaban-web-server
```

修改 azkaban web 服务器配置conf/azkaban.properties，主要包括：

a. 修改时区和首页名称:

~~~properties
# Azkaban Personalization Settings
azkaban.name=ETL Task
azkaban.label=By BI
azkaban.color=#FF3601
azkaban.default.servlet.path=/index
web.resource.dir=web/
default.timezone.id=Asia/Shanghai
~~~

b. 修改 MySql 数据库配置

~~~properties
database.type=mysql
mysql.port=3306
mysql.host=localhost
mysql.database=azkaban
mysql.user=azkaban
mysql.password=azkaban
mysql.numconnections=100
~~~

c. 修改 Jetty 服务器属性，包括 keystore 的相关配置

~~~properties
# Azkaban Jetty server properties.
jetty.hostname=0.0.0.0
jetty.maxThreads=25
jetty.port=8081
jetty.keystore=keystore
jetty.password=azkaban
jetty.keypassword=azkaban
jetty.truststore=keystore
jetty.trustpassword=azkaban
~~~

d. 开启SSL

```bash
jetty.use.ssl=true
jetty.ssl.port=8443
```

e.修改邮件设置（可选）

~~~properties
# mail settings
mail.sender=admin@javachen.space
mail.host=javachen.space
mail.user=admin
mail.password=admin
~~~

f. 解压缩azkaban-db-0.1.0-SNAPSHOT.zip，然后进入 mysql 命令行模式：

~~~bash
unzip azkaban-db-0.1.0-SNAPSHOT.zip

$ mysql -uazkaban -pazkaban
mysql> use azkaban
mysql> source azkaban-db-0.1.0-SNAPSHOT/create-all-sql-0.1.0-SNAPSHOT.sql
~~~

完整conf/azkaban.properties文件：

```bash
# Azkaban Personalization Settings
azkaban.name=ETL Task
azkaban.label=
azkaban.color=#FF3601
azkaban.default.servlet.path=/index
web.resource.dir=web/
default.timezone.id=Asia/Shanghai
# Azkaban UserManager class
user.manager.class=azkaban.user.XmlUserManager
user.manager.xml.file=conf/azkaban-users.xml
# Loader for projects
executor.global.properties=conf/global.properties
azkaban.project.dir=projects
# Velocity dev mode
velocity.dev.mode=false
# Azkaban Jetty server properties.
jetty.hostname=0.0.0.0
jetty.use.ssl=true
jetty.ssl.port=8443
jetty.maxThreads=25
jetty.port=8081
jetty.keystore=conf/keystore
jetty.password=azkaban
jetty.keypassword=azkaban
jetty.truststore=conf/keystore
jetty.trustpassword=azkaban
# Azkaban Executor settings
# mail settings
#mail.sender=
#mail.host=
#mail.user=
#mail.password=
# User facing web server configurations used to construct the user facing server URLs. They are useful when there is a reverse proxy between Azkaban web servers and users.
# enduser -> myazkabanhost:443 -> proxy -> localhost:8081
# when this parameters set then these parameters are used to generate email links.
# if these parameters are not set then jetty.hostname, and jetty.port(if ssl configured jetty.ssl.port) are used.
# azkaban.webserver.external_hostname=myazkabanhost.com
# azkaban.webserver.external_ssl_port=443
# azkaban.webserver.external_port=8081
job.failure.email=
job.success.email=
lockdown.create.projects=false
cache.directory=cache
# JMX stats
jetty.connector.stats=true
executor.connector.stats=true
# Azkaban mysql settings by default. Users should configure their own username and password.
database.type=mysql
mysql.port=3306
mysql.host=localhost
mysql.database=azkaban
mysql.user=azkaban
mysql.password=azkaban
mysql.numconnections=100
#Multiple Executor
azkaban.use.multiple.executors=true
#azkaban.executorselector.filters=StaticRemainingFlowSize,MinimumFreeMemory,CpuStatus
azkaban.executorselector.filters=StaticRemainingFlowSize,CpuStatus
azkaban.executorselector.comparator.NumberOfAssignedFlowComparator=1
azkaban.executorselector.comparator.Memory=1
azkaban.executorselector.comparator.LastDispatched=1
azkaban.executorselector.comparator.CpuUsage=1
executor.port=12321
```



# 安装 azkaban-exec-server

解压缩 azkaban-exec-server-0.1.0-SNAPSHOT.zip

```bash
unzip azkaban-exec-server-0.1.0-SNAPSHOT.zip
mv azkaban-exec-server-0.1.0-SNAPSHOT /opt/azkaban-exec-server
```



然后修改配置文件conf/azkaban.properties，包括：

a. 修改时区为：`default.timezone.id=Asia/Shanghai`

b. 修改 MySql 数据库配置

~~~properties
database.type=mysql
mysql.port=3306
mysql.host=localhost
mysql.database=azkaban
mysql.user=azkaban
mysql.password=azkaban
mysql.numconnections=100
~~~

c. 设置executor端口为12321

```bash
executor.maxThreads=50
executor.port=12321
executor.flow.threads=30
```

完整conf/azkaban.properties文件为：

```bash
default.timezone.id=Asia/Shanghai

azkaban.jobtype.plugin.dir=plugins/jobtypes

executor.global.properties=conf/global.properties
azkaban.project.dir=projects

database.type=mysql
mysql.port=3306
mysql.host=localhost
mysql.database=azkaban
mysql.user=azkaban
mysql.password=azkaban
mysql.numconnections=100

executor.maxThreads=50
executor.port=12321
executor.flow.threads=30
```



# 用户设置

进入 azkaban web 服务器 conf 目录，修改 azkaban-users.xml ，增加管理员用户：

~~~xml
<azkaban-users>
  <user groups="azkaban" password="admin027" roles="admin" username="azkaban"/>
  <user password="metrics" roles="metrics" username="metrics"/>

  <user groups="test" username="test"  password="123456" roles="read"/>

  <role name="admin" permissions="ADMIN"/>
  <role name="metrics" permissions="METRICS"/>
  <role name="read" permissions="READ"/>
  <role name="write" permissions="WRITE"/>
  <role name="execute" permissions="EXECUTE"/>
  <role name="schedule" permissions="SCHEDULE"/>
  <role name="createprojects" permissions="CREATEPROJECTS"/>
</azkaban-users>
~~~

# 启动服务

azkaban-exec-server，需要在 azkaban-exec-server 目录下执行下面命令：

~~~bash
sh bin/start-exec.sh
~~~

激活当前executor：

```bash
$ curl -G "localhost:$(<./executor.port)/executor?action=activate" && echo
{"status":"success"}
```

azkaban-web-server，需要在 azkaban-web-server 目录下执行下面命令：

~~~bash
sh bin/start-web.sh
~~~

# 配置插件

下载 [HDFS Browser](https://s3.amazonaws.com/azkaban2/azkaban-plugins/2.5.0/azkaban-hdfs-viewer-2.5.0.tar.gz) 插件，解压然后重命名为 hdfs，然后将其拷贝到 azkaban-web-server/plugins/viewer 目录下。

下载 [Job Types Plugins](https://s3.amazonaws.com/azkaban2/azkaban-plugins/2.5.0/azkaban-jobtype-2.5.0.tar.gz) 插件，解压然后重命名为 jobtype，然后将其拷贝到 azkaban-executor-server/plugins/ 目录下，然后修改以下文件，设置 `hive.home`：

- plugins/jobtypes/commonprivate.properties
- plugins/jobtypes/common.properties
- plugins/jobtypes/hive/plugin.properties

>说明：
>
>在实际使用中，这些插件都没有配置成功，故最后的生产环境没有使用这些插件，而是基于最基本的 command 或 script 方式来编写作业。

# Docker安装

参考：

- https://gitee.com/javachen/docker-azkaban

- https://gitee.com/datatech/docker-azkaban
- https://github.com/jfim/data-platform-demo/blob/master/dockerfiles/Dockerfile-azkaban
- https://github.com/wilson-lauw/azkaban