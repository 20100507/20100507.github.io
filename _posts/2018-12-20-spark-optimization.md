---
layout: post
title: "Spark优化"
date: 2018-12-20  
tag: 大数据Spark
---

### 前言
    
	数据平台部使用的是Spark2.3.1 on Yarn,以下总结以下开发中遇到的Spark调优问题.希望对读者可以起到一定的帮助作用

### 优化

**并行度&资源调整**

* 调整分区数量              spark.sql.shuffle.partitions                      默认200 按文件大小和文件个数调整
* 调整并行度                --num-executors                                   一般数量为 partitions/2 比较合适
* 调整每个executor内存      --executor-memory                                 一般取分区数据最大值的2倍
* 调整每个executor core     --executor-cores                                  默认为1,一般不需要修改
* 调整driver内存            --driver-memory                                   不建议在driver存放过多的数据
* 调整堆外内存              --conf spark.yarn.executor.memoryOverhead         默认为300M,一般设置为1G或者更多


**数据倾斜处理**

* 重新分区或者增加减少分区数,给指定的key增加前缀,把reduce转换为map操作

**其他参数调整**

* 调整拉取数据最长等待时间          --conf spark.core.connection.ack.wait.timeout     默认为60S
* 调整map端中间结果写入到buffer中   --conf spark.shuffle.file.buffer                  默认为32K
* 调整作业拉取buffer内存大小        --conf spark.reducer.maxSizeInFlight              默认48M
* 调整shuffle read内存比例          --conf spark.shuffle.memoryFraction               默认0.2
* 调整网络超时时长                  --conf spark.network.timeout
* 调整序列化设置                    --conf spark.network.timeout

**SparkStreaming调优**

* 使用直连方式 kafka partition数量决定了task并行度,适当增加partition
* 适当调整每次拉取数据量和拉取的间隔
* --conf spark.dynamicAllocation.enabled=false 关闭动态资源分配
* Offset 可以采用redis管理

**代码调优**

* 尽量使用连接池
* 尽量采用RDD(Cache)
* checkpoint(适可而止使用)
* 不同的算子shuffle过程不同尽量结合适合的业务场景使用优化过的算子

**Hive&SparkSQL&BUG提交社区**

> * Hive结合SparkSQL 解析复杂JSON出现异常,通过测试Spark使用一个core不会报错,超过一个core会报错,可以初步判断是线程安全问题
> * Bug在hive-hcatalog-core-1.1.0.jar包中 ,在多线程情况使用了非线程安全的HashMap,切换为ConcurrentHashMap即可.
> * 更多资料请参见社区 SPARK-17398 HIVE-21752


### 总结

	Spark开发中优化可以对节省大量集群的时间和资源开销.

### 参考

* https://issues.apache.org/jira/browse/HIVE-21752
* https://issues.apache.org/jira/browse/SPARK-17398

