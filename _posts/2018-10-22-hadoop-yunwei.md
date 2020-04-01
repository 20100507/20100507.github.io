---
layout: post
title: "Hadoop trouble-shooting"
date: 2018-10-21 
tag: 大数据技术
---

### 前言
    
	   在数据平台事业部运维Hadoop集群中,出现一些问题以及总结记录如下

### 问题&解决方案


1.HDFS扩容升级

> 1. 问题描述：新增加机器，扩容HDFS，已经扩容完毕，之后运维部重新挂载磁盘，HDFS出现逻辑卷错误
> 2. 问题分析：日志显示挂载磁盘错误，实际是写数据的目录权限不足
> 3. 问题解决方案：修改挂载的磁盘的所属用户为hadoop，重新启动dataNode即可

2.HDFS 修复

> 1. 问题描述：其他部门在yarn平台上跑spark 程序错误的生成了海量的不到100K的小文件，导致namenode压力过大，其中一个namenode宕机后，没有及时发现 使得edits文件大量积累，在namenode1   宕机后，namenode2 随后在凌晨1点也宕机。
> 2. 原因分析：NameNode 内存设置太低，之前内存设置在1G，后调高namenode 堆内存，调高到18G。编写程序的人员不应该生成海量的小文件落地HDFS，大量的小文件不适合存储在HDFS上。
> 3. 问题解决方案：提高namenode 内存，由于edits文件太多，其他人删除了edits文件，但是journalNode 上edits txid不一致，导致namenode 无法启动，后执行命令 初始化edits文件的txid ，修复完毕！丢失了少量的文件。

3.HDFS 异构存储 磁盘容量不一致

> 1. 问题描述：由于HDFS dataNode节点的磁盘大小不一致，导致小容量的HDFS先打满，yarn跑任务出现问题
> 2. 问题分析：尽量让数据一致
> 3. 问题解决方案：开启HDFS让数据优先写入容量大的磁盘，优先写入容量的dataNode节点，开启rebalance

4.HDFS 机架感知不可以同时存在有机架和没有机架的节点

> 1. 问题描述：机架感知不可以同时存在有机架和没有机架的节点
> 2. 问题分析：开始以为是HDFS的topology.sh文件中的default-rack导致的，但是修改为已有的机架依旧不可以，观察HDFS源码发现hdfs源码中也有default-rack，如果没有机架配置默认使用default-rack即没有机架
> 3. 问题解决方案：由于topology.data文件最后一行需要回车即可

5.Spark 客户端安装后Driver和Yarn无法通讯

> 1. 问题描述：项目经理在10.10.26.7 机器上安装spark 客户端，但是dirver端和yarn无法通讯.一直报通讯超时
> 2. 原因分析：由于客户端是centos6,服务端为centos7。底层通讯方式不相同
> 3. 问题解决方案：升级客户端机器到centos7问题解决。

6.Yarn NodeManager 宕机

> 1. 问题描述：新增的NodeManager 机器NodeManager 进程一直宕
> 2. 问题分析：观察日志问OOM，打开文件limit受限制
> 3. 问题解决方案：修改linux系统其它用户打开文件数量的限制即可解决

7. HDFS NameNode无法和JournalNode通讯，NameNode宕机

> 1. 问题描述： 由于运维不修改网络导致未通知数据平台部journalNode无法和nameNode 通讯,观察日志NameNode尝试与journalNode通讯10次失败后，NameNode直接宕机
> 2. 问题分析:   JournalNode网络通讯高可用需要保证,多数节点写入成功才可,超过半数的节点宕机,active namenode无法写入journalnode 为了数据一致性,namenode主动退出
> 3. 问题解决方案：重新启动nameNode问题解决

8. resourcemanager 和 nodemanager全部主动停止

> 1. 问题描述：运维部未通知数据平台部修改网络,导致zookeeper如果全部宕机，yarn也全部宕机
> 2. 问题分析：yarn需要靠zookeeper做强一致性选举，如果zookeeper全部宕机yarn也会宕机
> 3. 问题解决方案：yarn生产环境一定要配置zookeeper高可用

9. Hbase 表无法写入查询失败

> 1. 问题描述：由于zookeeper宕机，观察hbase UI 如下错误
> 2. 问题观察分析：由于region分裂和zookeeper通讯失败，hregionserver 宕机，该表的region开始迁移，多次迁移后，依然region存在宕机，达到重试最高次数，表直接损坏无法写入
> 3. 重新修改zookeeper地址,启动zk
<br/>
```
sun.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method)
sun.reflect.NativeConstructorAccessorImpl.newInstance(NativeConstructorAccessorImpl.java:62)
sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45)
java.lang.reflect.Constructor.newInstance(Constructor.java:423)
org.apache.hadoop.hbase.ipc.RemoteWithExtrasException.instantiateException(RemoteWithExtrasException.java:95)
org.apache.hadoop.hbase.ipc.RemoteWithExtrasException.unwrapRemoteException(RemoteWithExtrasException.java:85)
org.apache.hadoop.hbase.protobuf.ProtobufUtil.makeIOExceptionOfException(ProtobufUtil.java:368)
org.apache.hadoop.hbase.protobuf.ProtobufUtil.getRemoteException(ProtobufUtil.java:330)
org.apache.hadoop.hbase.client.HBaseAdmin.getCompactionState(HBaseAdmin.java:3454)
org.apache.hadoop.hbase.generated.master.table_jsp._jspService(table_jsp.java:283)
org.apache.jasper.runtime.HttpJspBase.service(HttpJspBase.java:98)
javax.servlet.http.HttpServlet.service(HttpServlet.java:820)
org.mortbay.jetty.servlet.ServletHolder.handle(ServletHolder.java:511)
org.mortbay.jetty.servlet.ServletHandler$CachedChain.doFilter(ServletHandler.java:1221)
org.apache.hadoop.hbase.http.lib.StaticUserWebFilter$StaticUserFilter.doFilter(StaticUserWebFilter.java:113)
org.mortbay.jetty.servlet.ServletHandler$CachedChain.doFilter(ServletHandler.java:1212)
org.apache.hadoop.hbase.http.ClickjackingPreventionFilter.doFilter(ClickjackingPreventionFilter.java:48)
org.mortbay.jetty.servlet.ServletHandler$CachedChain.doFilter(ServletHandler.java:1212)
org.apache.hadoop.hbase.http.HttpServer$QuotingInputFilter.doFilter(HttpServer.java:1435)
org.mortbay.jetty.servlet.ServletHandler$CachedChain.doFilter(ServletHandler.java:1212)
org.apache.hadoop.hbase.http.NoCacheFilter.doFilter(NoCacheFilter.java:49)
org.mortbay.jetty.servlet.ServletHandler$CachedChain.doFilter(ServletHandler.java:1212)
org.apache.hadoop.hbase.http.NoCacheFilter.doFilter(NoCacheFilter.java:49)
org.mortbay.jetty.servlet.ServletHandler$CachedChain.doFilter(ServletHandler.java:1212)
org.mortbay.jetty.servlet.ServletHandler.handle(ServletHandler.java:399)
org.mortbay.jetty.security.SecurityHandler.handle(SecurityHandler.java:216)
org.mortbay.jetty.servlet.SessionHandler.handle(SessionHandler.java:182)
org.mortbay.jetty.handler.ContextHandler.handle(ContextHandler.java:766)
org.mortbay.jetty.webapp.WebAppContext.handle(WebAppContext.java:450)
org.mortbay.jetty.handler.ContextHandlerCollection.handle(ContextHandlerCollection.java:230)
org.mortbay.jetty.handler.HandlerWrapper.handle(HandlerWrapper.java:152)
org.mortbay.jetty.Server.handle(Server.java:326)org.mortbay.jetty.HttpConnection.handleRequest(HttpConnection.java:542)
org.mortbay.jetty.HttpConnection$RequestHandler.headerComplete(HttpConnection.java:928)
org.mortbay.jetty.HttpParser.parseNext(HttpParser.java:549)
org.mortbay.jetty.HttpParser.parseAvailable(HttpParser.java:212)
org.mortbay.jetty.HttpConnection.handle(HttpConnection.java:404)
org.mortbay.io.nio.SelectChannelEndPoint.run(SelectChannelEndPoint.java:410)
org.mortbay.thread.QueuedThreadPool$PoolThread.run(QueuedThreadPool.java:582) Unknown
```

## 总结

	集群规模庞大后,会出现一系列的问题,发现问题第一时间需要观察日志,定位问题所在,基本上可以解决.