---
layout: post
title: 从零搭建容器化的服务环境（dev）笔记
categories: [docker]
tags: [docker, java]
description: 容器化
---


```
小公司没有运维岗位，所以一直只有两套环境（测试、生产），开发人员调试很难受，这里开始尝试使用容器化搭建一套调试环境，方便开发人员进行调试，也为后面其它环境服务的容器化铺路，这里由于资源问题，没有选择过于复杂的方案。
```
#### 硬件环境
- 阿里ecs 4 vCPU 32 GB CentOS 6.8 64位 x2

---

#### 准备工作
docker 1.13.1需要centos 7+版本，所以需要先升级系统，[参考链接](http://www.oracleonlinux.cn/2017/05/how-to-upgrade-centos6-5-to-centos7/)

升级重启，之后发现一些问题：

- 部分命令无法工作了
```
#grep无法使用
ln -s /lib64/libpcre.so.1.2.0 /lib64/libpcre.so.0
#yum无法使用
ln -s /lib64/libsasl2.so.3.0.0 /lib64/libsasl2.so.2
```

- sshd安装 [#1](https://my.oschina.net/glenxu/blog/1611859) [#2](https://blog.csdn.net/wang704987562/article/details/72722263/)
- 如果无法访问，重新安装firewalld并关闭
- [firewalld无法启动](https://blog.csdn.net/crynono/article/details/76132611)

```
#查看开放的端口
firewall-cmd --zone=public --list-ports
```

- [DOCKER、SWARM、PORTAINER](http://wiselyman.iteye.com/blog/2373562)
- 先建立一个Portainer Agent，之后在end point中加入这个agent

```
#swarm挂载本地时间
--mount type=bind,src=/etc/localtime,dst=/etc/localtime
#普通容器挂载本地时间
-v /etc/localtime:/etc/localtime:ro
```

- 如果本机时区有问题，需要同步时区，[参考](https://www.cnblogs.com/fanlinglong/p/6363031.html)
- 如果同步不成功，看一下ntp需要的123端口是否已打开，也可以使用rdate工具同步

---

#### 准备工作

> 这一步准备服务需要运行的基础环境，每个公司的基础环境都会有所不同，有些直接使用云服务，这里都是自己搭建了，整理了一下需要搭建的环境有：Mysql（主从，因为配置了读写分离），rabbitMQ，redis，es，consul，nginx。最终形成一些本地的镜像，方便使用。

##### Mysql主从

- 需要挂载配置文件，这里准备两份简单的配置文件

master.cnf
```
[mysqld]  
server-id=101
log-bin=mysql-bin
character-set-server=utf8
init_connect='SET NAMES utf8'
sql-mode="STRICT_TRANS_TABLES,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION"

[mysql]
default-character-set = utf8

[mysql.server]
default-character-set = utf8

[mysqld_safe]
default-character-set = utf8

[client]
default-character-set = utf8
```
slave.cnf

```
[mysqld]
server-id=102
log-bin=mysql-bin
character-set-server=utf8
init_connect='SET NAMES utf8'
sql-mode="STRICT_TRANS_TABLES,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION"

[mysql]
default-character-set = utf8

[mysql.server]
default-character-set = utf8

[mysqld_safe]
default-character-set = utf8

[client]
default-character-set = utf8
```

服务器1、主：

```
docker create --name dev-mysql-1 -v /usr/local/tmmt/mysql/master/data:/var/lib/mysql -v /usr/local/tmmt/mysql/master/cnf:/etc/mysql/ -e MYSQL_ROOT_PASSWORD=tmmtmysql -p 12001:3306 mysql:5.7
```

服务器2、从：

```
docker create --name dev-mysql-2 -v /usr/local/tmmt/mysql/slave/data:/var/lib/mysql -v /usr/local/tmmt/mysql/slave/cnf:/etc/mysql/ -e MYSQL_ROOT_PASSWORD=tmmtmysql -p 12001:3306 mysql:5.7
```

- start镜像后可以连接数据库了，之后配置一下主从复制

主

```
SET sql_mode=(SELECT REPLACE(@@sql_mode,'ONLY_FULL_GROUP_BY',''));
GRANT REPLICATION SLAVE ON *.* to 'backup'@'%' identified by 'tmmtmysql';
```

从

```
SET sql_mode=(SELECT REPLACE(@@sql_mode,'ONLY_FULL_GROUP_BY',''));
change master to change master to master_host='172.31.178.93',master_port=12001,master_user='backup',master_password='tmmtmysql',master_log_file='mysql-bin.000003',master_log_pos=0;
start slave
show slave status;
```
- 之后就是给对应的服务加账户、权限等等


##### CONSUL

[参考](https://blog.csdn.net/fenglailea/article/details/79098246)


```
docker run -d --name dev-consul-s-1 -e 'CONSUL_LOCAL_CONFIG={"skip_leave_on_interrupt": true}' consul:0.7.5 agent -server  -node=node1 -bootstrap-expect=2
JOIN_IP="$(docker inspect -f '{{.NetworkSettings.IPAddress}}' dev-consul-s-1)"
docker run -d --name dev-consul-s-2 -e 'CONSUL_LOCAL_CONFIG={"skip_leave_on_interrupt": true}' consul:0.7.5 agent -server  -node=node2 -join $JOIN_IP
docker run -d --name dev-consul-s-3 -e 'CONSUL_LOCAL_CONFIG={"skip_leave_on_interrupt": true}' consul:0.7.5 agent -server  -node=node3 -join $JOIN_IP

docker run -d --name  dev-consul-c-1 -p 12010:8400 -p 12011:8500 -p 12012:53/udp -e 'CONSUL_LOCAL_CONFIG={"skip_leave_on_interrupt": true}' consul:0.7.5 agent -ui -node=node11 -client=0.0.0.0 -join $JOIN_IP
```


##### redis

[参考](https://www.cnblogs.com/vipzhou/p/8580495.html)

集群搭建还没有研究，这里先用单点满足需求

```
docker run --name dev-redis-1 -p 12020:6379 -d redis

config set requirepass tmmtredis
cd / && touch sentinel.conf
echo 'sentinel monitor mymaster 172.17.0.7 6379 1'>>/sentinel.conf
echo 'sentinel auth-pass mymaster tmmtredis'>>/sentinel.conf
redis-sentinel /sentinel.conf &
```


##### rabbitMQ

集群搭建还没有研究，这里先用单点满足需求

```
docker run -d \
--name=dev-mq-1 \
-p 12030:5672 \
-p 12031:15672 \
-e RABBITMQ_NODENAME=rabbitmq1 \
-e RABBITMQ_ERLANG_COOKIE='YZSDHWMFSMKEMBDHSGGZ' \
-h rabbitmq1 \
--net=rabbitmqnet \
rabbitmq
```

##### es


```
# es1
docker run -d --name dev-es1 -p 12040:9200 -p 12042:9300 -v /usr/local/tmmt/es/config/es1/es1.yml:/usr/share/elasticsearch/config/elasticsearch.yml -v /usr/local/tmmt/es/data/es1:/usr/share/elasticsearch/data -v /usr/local/tmmt/es/config/analysis-ik:/usr/share/elasticsearch/plugins/analysis-ik/config docker.elastic.co/elasticsearch/elasticsearch:5.6.4

docker run -d --name dev-es1 -p 9200:9200 -p 9300:9300 -v /usr/local/tmmt/es/config/es1/es1.yml:/usr/share/elasticsearch/config/elasticsearch.yml -v /usr/local/tmmt/es/data/es1:/usr/share/elasticsearch/data -v /usr/local/tmmt/es/config/analysis-ik:/usr/share/elasticsearch/plugins/analysis-ik/config elasticsearch:5.6.4

# es2
docker run -d --name es2 --link es1:es1 -p 12043:9200 -p 12044:9300 -v /usr/local/tmmt/es/config/es2/es2.yml:/usr/share/elasticsearch/config/elasticsearch.yml -v /usr/local/tmmt/es/data/es2:/usr/share/elasticsearch/data docker.elastic.co/elasticsearch/elasticsearch:5.6.4

docker run -d --name dev-es2 -p 9201:9200 -p 9301:9300 -v /usr/local/tmmt/es/config/es2/es2.yml:/usr/share/elasticsearch/config/elasticsearch.yml -v /usr/local/tmmt/es/data/es2:/usr/share/elasticsearch/data elasticsearch:5.6.4

# head
docker run -d --name dev-head -p 12040:9100 -v /usr/local/tmmt/es/head/Gruntfile.js:/usr/src/app/Gruntfile.js -v /usr/local/tmmt/es/head/app.js:/usr/src/app/_site/app.js mobz/elasticsearch-head:5

docker run -d --name head -p 12041:9100 -v /usr/local/tmmt/es/head/app.js:/usr/src/app/_site/app.js mobz/elasticsearch-head:5

# ik 分词插件

elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v5.6.4/elasticsearch-analysis-ik-5.6.4.zip
```


##### registry私有仓库


```
cat config.yml
 
version: 0.1
log:
  fields:
    service: registry
storage:
  delete:
    enabled: true
  cache:
    blobdescriptor: inmemory
  filesystem:
    rootdirectory: /var/lib/registry
http:
  addr: :5000
  headers:
    X-Content-Type-Options: [nosniff]
health:
  storagedriver:
    enabled: true
    interval: 10s
    threshold: 3
```


```
docker run -d -p 5000:5000 -v /opt/data/registry:/var/lib/registry  -v /opt/data/config.yml:/etc/docker/registry/config.yml  registry
```

到这里，基本环境都准备的差不多了

---


```
下面开始服务的容器化工作，服务端使用java，所以首先，我们需要一个带jdk的基础镜像
```


- 制作一个jdk基础镜像，下载jdk-8u181-linux-x64.tar.gz包


```
vim Dockerfile

dockerfile：
#生成的新镜像以centos镜像为基础
FROM docker.io/centos
MAINTAINER by crazy0x
#升级系统
RUN yum -y update
#添加jdk安装包
RUN mkdir /usr/java
ADD jdk-8u181-linux-x64.tar.gz /usr/java/
#安装jdk
ENV JAVA_HOME /usr/java/jdk1.8.0_181
ENV CLASSPATH ./:$JAVA_HOME/lib:$JAVA_HOME/jre/lib
ENV PATH $JAVA_HOME/bin:$JAVA_HOME/jre/bin:$PATH

docker build -t centos_jdk .
```
- 之后将镜像push到本地仓库中，因为后面每次发布容器都要基于该镜像来做


- 写一个部署的shell脚本，将打包好的应用构建成镜像，然后发布。

- 脚本还在优化中，这里不同服务间的命令有一些区分，因为要做的事情不一样，比如支付服务需要挂载证书

```
#!/bin/bash

conf_dir=/usr/local/tmmt/war

service=$1

if [ -z "$1" ]; then
    echo -e "please enter service name!\n"
    exit
fi

package=$2

if [ -z "$2" ]; then
    echo -e "please enter package name!\n"
    exit
fi

port=$3

svr_runing=$(docker ps | grep ${service}_dev)
if [[ ${svr_runing} != "" ]];
then
    echo -e "service has been running.\n"
    echo -e "removing service ${service}_dev...\n"
    docker rm -f dev-${service}
fi

svr_image=$(docker images | grep ${service}_dev)
if [[ ${svr_image} != "" ]];
then
    echo -e "removing image ${service}_dev...\n"
    docker rmi 172.31.178.93:5000/${service}_dev
fi

mkdir -p /data/log/app/${service}

dockerfile="FROM 127.0.0.1:5000/centos_jdk:1.1\n
MAINTAINER by liubaixin\n
RUN mkdir -p /usr/local/tmmt/${service}\n
ADD ${package} /usr/local/tmmt/${service}\n
WORKDIR /usr/local/tmmt/${service}\n
ENTRYPOINT [\"java\",\"-Duser.timezone=GMT+08\",\"-jar\",\"${package}\"]"

echo $dockerfile
rm -rf ${conf_dir}/Dockerfile
echo -e ${dockerfile} > ${conf_dir}/Dockerfile
echo -e ${dockerfile}

docker build -t 172.31.178.93:5000/${service}_dev .

#docker push 172.31.178.93:5000/${service}_dev

if [[ ${port} != "" ]];
then
    #docker service create --name ${service}_dev -p ${port}:${port} --mount type=bind,src=/data/log/app/${service},dst=/data/log/app/${service} --replicas 1 172.31.178.93:5000/${service}_dev
    docker run -d --name dev-${service} -p ${port}:${port}  --net=host  -v /etc/localtime:/etc/localtime:ro -v /data/log/app/${service}:/data/log/app/${service} 172.31.178.93:5000/${service}_dev
    exit
fi

if [[ ${service} == "pay" ]];
then
    #docker service create --name ${service}_dev -p ${port}:${port} --mount type=bind,src=/data/log/app/${service},dst=/data/log/app/${service} --replicas 1 172.31.178.93:5000/${service}_dev
    docker run -d --name dev-${service} --net=host  -v /etc/localtime:/etc/localtime:ro -v /usr/local/tmmt/cert:/usr/local/tmmt/cert -v /data/log/app/${service}:/data/log/app/${service} 172.31.178.93:5000/${service}_dev
    exit
fi

docker run -d --name dev-${service} --net=host  -v /etc/localtime:/etc/localtime -v /data/log/app/${service}:/data/log/app/${service} 172.31.178.93:5000/${service}_dev
#docker service create --name ${service}_dev --mount type=bind,src=/data/log/app/${service},dst=/data/log/app/${service} --replicas 2 172.31.178.93:5000/${service}_dev

```

- 之后使用jenkins打包构建，然后将包传到服务器上面执行脚本，实现自动发布。

--- 

#### 其它

- 服务的发布应该使用swarm发布来保证可用性，这里因为是调试环境没有做深入研究。
- motan注册默认拿host ip，为了方便开发进行本地调试，可以将host ip设置为公网ip


