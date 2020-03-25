---
layout: post
title: "Hadoop2.7.6滚动升级Hadoop2.8.5"
date: 2018-10-26
tag: 大数据Hadoop
---

### 前言
    
	Hadoop2.7.6已经落后Hadoop主流版本好几个版本,目前主流的Hadoop版本为2.8.x,2.9.x,2.10.x,3.x.决定滚动升级数据平台部的Hadoop版本为2.8.5,采用零停机方式升级.
	笔者的Hadoop版本为2.7.6,需要注意Hadoop版本滚动升级最低版本为2.4.x.并且在升级的过程中,不要动JournalNode服务,如果升级JournalNode服务可能导致集群宕机.

### 滚动升级NameNode

**备份元数据**

***在Active NameNode执行如下命令***

> `hdfs dfsadmin -rollingUpgrade prepare`

***等待命令行返回 Proceed with rolling upgrade***

> `hdfs dfsadmin -rollingUpgrade query`

***或者观察web-ui 如下，回滚的镜像已经创建***

<div align="left">
<img src="/images/posts/hadoop3/rollupdate1.png" height="280" width="1180" />  
</div>

<br/>

<div align="left">
<img src="/images/posts/hadoop3/rollupdate2.png" height="280" width="1180" />  
</div>

> 1. 停止active namenode nn1 替换相关jar包 /opt/hadoop/share/hadoop/下所有的类库
> 2. 执行如下命令启动nn1
> 3. `hadoop-daemon.sh start namenode -rollingUpgrade started`
> 4. 停止 standyby namenode nn2 替换相关jar包 /opt/hadoop/share/hadoop/下所有的类库
> 5. `hadoop-daemon.sh start namenode -rollingUpgrade started`

### 滚动升级DataNode

**关闭更新datanode**

  期间如果集群规模较大，并且使用人数很多，建议修改datanode和namenode检查心跳丢失的最大时间默认为10分钟，或者降低每次datanode滚动更新的数量
  防止丢失临时namenode检测到block丢失,开始大量复制block对集群产生比较大的波动.

> 1. `hdfs dfsadmin -shutdownDatanode datanode01:50020 upgrade`
> 2. 替换完毕jar包后重新启动
> 3. `hadoop-daemon.sh start datanode`
> 4. 需要滚动更新所有的datanode节点

### 确认升级完毕

  执行如下命令确认升级完毕
  
> 1. `hdfs dfsadmin -rollingUpgrade finalize`

<br/>
<div align="left">
<img src="/images/posts/hadoop3/rollupdate3.png" height="280" width="1180" />  
</div>

### 重启YARN

> 1. yarn-daemon.sh stop resourcemanager
> 2. yarn-daemon.sh start resourcemanager
> 3. yarn-daemon.sh stop nodemanager
> 4. yarn-daemon.sh start nodemanger
>
  如下图为yarn升级完毕
<br/>
<div align="left">
<img src="/images/posts/hadoop3/rollupdate4.png" height="180" width="1180" />  
</div>

### 总结
	Hadoop2.7.6滚动升级难度不是很高,后期需要考虑升级Hadoop3.x,滴滴在最近已经有实践.可以参考借鉴.

### 参考

*https://hadoop.apache.org/docs/r2.8.5/hadoop-project-dist/hadoop-hdfs/HdfsRollingUpgrade.html
