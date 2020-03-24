---
layout: post
title: "Ranger集成Hive使用"
date: 2018-08-20   
tag: 大数据Ranger
---

### 前言
    
   使用Ranger&Hive集成中,出现一些设置上的错误,记录于此.

### 问题

   执行在hiveserver 命令行 set xxx报错如下 

>   org.apache.hive.service.cli.HiveSQLException: Error while processing statement: Cannot modify ..** at runtime.
>   It is not in list of params that are allowed to be modified at runtime
	 
### 解决方案
    
	在hiveserver2.xml文件中,添加如下配置,为以下命令添加白名单
	
    ```
	  <property>
          <name>hive.security.authorization.sqlstd.confwhitelist.append</name>
          <value>mapred.*|hive.*|mapreduce.*|spark.*</value>
      </property>
      <property>
          <name>hive.security.authorization.sqlstd.confwhitelist</name>
          <value>mapred.*|hive.*|mapreduce.*|spark.*</value>
      </property>
	```
