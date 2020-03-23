---
layout: post
title: "Docker常用操作"
date: 2018-08-02   
tag: 容器Docker
---

### 前言
    
	Docker作为容器技术,以其秒级启动,资源占用小等优点,在2014年开始在国内流行,逐渐被越来越多的公司认可。如下为工作中记录的常用命令,
	用作备忘。

### 常用的应用启动

**启动mongodb容器**


> 1. `docker run -di --name=mongo -p 27017:27017 docker.io/mongo`

**启动rabbitMQ容器**

> 1. `docker run -di --name=rabbitmq -p 5671:5671 -p 5672:5672 -p 4369:4369 -p 15671:15671 -p 15672:15672 -p 25672:25672 rabbitmq:management`

**启动Redis容器**

> 1. `docker run -di --name=redis -p 6379:6379 docker.io/redis`

**启动MySQL容器**

> 1. `docker run -di --name=mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=root docker.io/centos/mysql-57-centos7`

**启动ES容器**

***配置文件网盘地址***
****https://pan.baidu.com/s/1X8jXxt_8o09MnhV63wJLbQ v98c***

> 1. `docker run -di --name=es -p 9200:9200 -p 9300:9300 -v /usr/share/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml elasticsearch:5.6.8`
> 2. `docker cp es:/usr/share/elasticsearch/config/elasticsearch.yml /usr/share/elasticsearch.yml`
> 3. `docker run -di --name=header -p 9100:9100 mobz/elasticsearch-head:5`
> 4. `docker cp ik es:/usr/share/elasticsearch/plugins`

### 常用的批量操作命令


***删除所有容器***

> 1. `docker rm \`docker ps -a -q\``

***删除所有的镜像***

> 1. `docker rmi \`docker images -q\``

***删除没有tag的所有镜像***

> 1. `docker rmi -f $(docker images | grep '<none>' | tr -s ' ' | cut`

***删除镜像中包含关键字的镜像***

> 1. `docker rmi --force \`docker images | grep doss-api | awk '{print $3}'\``    //其中doss-api为关键字