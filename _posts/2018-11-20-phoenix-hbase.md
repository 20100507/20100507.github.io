---
layout: post
title: "Phoenix &Hbase"
date: 2018-11-20  
tag: 大数据Phoenix
---

### 前言
    
	Hbase作为列式存储,严重依赖rowkey查询,一定程度限制了开发人员使用,Phoenix采用空间换时间的方式存放数据,采用SQL查询数据降低了开发人员
	开发的门槛.

### 安装Phoenix

**拷贝相关jar包**

* 备注:注意Hbase和Phoenix对应的关系

> 1. `cp phoenix-4.14.3-HBase-1.3-server.jar /opt/hbase/lib`
> 2. `cp phoenix-core-4.14.3-HBase-1.3.jar /opt/hbase/lib`

**修改hbase配置**

* 编辑hbase-site.xml

```
   <property>
      <name>hbase.table.sanity.checks</name>
      <value>false</value>
   </property>
   <property>
     <name>phoenix.schema.mapSystemTablesToNamespace</name>
     <value>true</value>
   </property>
   <property>
     <name>phoenix.schema.isNamespaceMappingEnabled</name>
     <value>true</value>
   </property>
   <property>
     <name>hbase.regionserver.wal.codec</name>
     <value>org.apache.hadoop.hbase.regionserver.wal.IndexedWALEditCodec</value>
   </property>
```

**滚动重启Hbase**

> 1. `rolling-restart.sh`

**创建Phoenix客户端&初始化Phoenix**

> 1. `tar -zxvf apache-phoenix-4.14.3-HBase-1.3-bin.tar.gz /opt/phoenix`
> 2. `cp /opt/hbase/conf/hbase-site.xml /opt/phoenix/bin/`
> 3. `python2.7 /opt/phoenix/bin/sqlline.py`

### 功能测试

* 创建Hbase表

> 1. `create 'test:FACT_FORWARD_TEST',{NAME=>'LOG_DETAIL',COMPRESSION=>'lzo'},SPLITS=>['1','2','3','4','5','6']`

* 常见错误

<div align="left">
<img src="/images/posts/phoenix-hbase/phoenix.png" height="540" width="1140" />  
</div>


* 创建视图和Hbase映射

> 2. `CREATE VIEW test.FACT_FORWARD_TEST ( ROWKEY VARCHAR PRIMARY KEY, "LOG_DETAIL"."DELAY20S" VARCHAR, "LOG_DETAIL"."DELAY30S" VARCHAR, "LOG_DETAIL"."DELAY30SGT" VARCHAR, "LOG_DETAIL"."DEPLAYPERCENT" VARCHAR, "LOG_DETAIL"."MONITORPLATFORM" VARCHAR,"LOG_DETAIL"."SENDTIME" VARCHAR ,"LOG_DETAIL"."TIME" VARCHAR ,"LOG_DETAIL"."TOTALDELAY" VARCHAR ,"LOG_DETAIL"."TOTAL" VARCHAR,"LOG_DETAIL"."VID" VARCHAR,"LOG_DETAIL"."VTYPE" VARCHAR ) COLUMN_ENCODED_BYTES = 0;`


* 创建ACT_FORWARD_TEST索引

> 3. `CREATE INDEX TEST_INDEX ON test.FACT_FORWARD_TEST(VID) INCLUDE(VTYPE,SENDTIME,MONITORPLATFORM, DEPLAYPERCENT);`

* Hbase插入数据

> 1. `put 'test:FACT_FORWARD_TEST','0_LS4AAB3C1GG700213_1558668658000','LOG_DETAIL:DELAY20S','0'`
> 2. `put 'test:FACT_FORWARD_TEST','0_LS4AAB3C1GG700213_1558668658000','LOG_DETAIL:DELAY30S','2'`
> 3. `put 'test:FACT_FORWARD_TEST','0_LS4AAB3C1GG700213_1558668658000','LOG_DETAIL:DELAY30SGT','0'`
> 4. `put 'test:FACT_FORWARD_TEST','0_LS4AAB3C1GG700213_1558668658000','LOG_DETAIL:DEPLAYPERCENT','100.0'`
> 5. `put 'test:FACT_FORWARD_TEST','0_LS4AAB3C1GG700213_1558668658000','LOG_DETAIL:MONITORPLATFORM',''`
> 6. `put 'test:FACT_FORWARD_TEST','0_LS4AAB3C1GG700213_1558668658000','LOG_DETAIL:SENDTIME','1558668680000'`
> 7. `put 'test:FACT_FORWARD_TEST','0_LS4AAB3C1GG700213_1558668658000','LOG_DETAIL:TIME','1558668658000'`

* phoenix查询数据,一定不要认为进入phoenix执行!table可以查看到表说明安装就没有问题

> 1. `SELECT * FROM test.FACT_FORWARD_TEST LIMIT 2;`


### 优化配置

* 一次性创建大量数据的索引超时错误,修改hbase-site.xml文件

```
  <property>
  <name>phoenix.query.timeoutMs</name>
  <value>1800000</value>
  <source>hbase-site.xml</source>
  </property>
  
```

### 总结

	网上对Phoenix资料零散,本文对Phoenix使用安装做一个简单的总结.希望对读者可以起到一定的帮助作用.

