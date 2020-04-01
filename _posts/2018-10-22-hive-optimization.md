---
layout: post
title: "Hive 配置与调优"
date: 2018-10-28 
tag: 大数据技术
---

### 前言
    
	   笔者在HQL开发中,难免需要优化一些配置,后期不定期更新,记录如下

### 优化和解决方案

1.Hive乱码

> 1. 在元数据库中执行如下命令

<br/>

```
alter table COLUMNS_V2 modify column COMMENT varchar(256) character set utf8;
alter table TABLE_PARAMS modify column PARAM_VALUE varchar(4000) character set utf8;
alter table PARTITION_PARAMS  modify column PARAM_VALUE varchar(4000) character set utf8;
alter table PARTITION_KEYS  modify column PKEY_COMMENT varchar(4000) character set utf8;
alter table  INDEX_PARAMS  modify column PARAM_VALUE  varchar(4000) character set utf8;

```

2.Hive优化

Hive 名称太长报错

> 1. Hive 名称太长报错
> 2. `FAILED: Execution Error, return code 1 from org.apache.hadoop.hive.ql.exec.mr.MapRedTask. Could not find status of job:job_1565750929426_0404`
> 3. hive的运行原理：当任务执行结束后，对于每一个jobid，会根据一定规则，生成两个文件，一个是*.jhist,另一个是*.conf.这两个文件记录了这个job的所有执行信息，这两个文件是要写入到jobhistory所监控的路径的。
> 4. set  hive.jobname.length=10;

Hive 普通的查询不走mr

> 1. `set hive.fetch.task.conversion=more`

Hive 开启中间压缩

> 1. `set hive.exec.compress.intermediate=true`

JVM 重用 mapred-site.xml文件中修改

> 1. `mapreduce.job.jvm.numtasks=10`

## 总结

	在开发中难免会遇到Hive调优,记录于此.