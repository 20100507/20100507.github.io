---
layout: post
title: "Hadoop源码分析-14-AvailableSpaceBlockPlacementPolicy"
date: 2020-03-04
tag: Hadoop源码分析
---

### 前言

       当集群规模庞大的时候,难免会存在异构的节点,比如磁盘的大小差异巨大,如果依然采用HDFS默认的副本的放置策略(第一块副本为本节点,第二块为另外
	一个机架,最后一个副本为同一个机架的随机的一个节点).这就存在一个问题,如果随机存放第三个副本的话,会导致节点基本上是数据均衡存放.如果存在异构
	节点存放数据不是按照节点属于空间的百分比存放数据.同样的数据磁盘小的节点很快被打满.
	   这是我们需要一个新的策略第三个副本不可以随机的选择副本节点存放数据,而是考虑节点使用百分比.这是就会使用到`AvailableSpaceBlockPlacementPolicy`

### 配置使用AvailableSpaceBlockPlacementPolicy

  1. 修改配置hdfs-site.xml文件
    
	```
	dfs.block.replicator.classname: org.apache.hadoop.hdfs.server.blockmanagement.AvailableSpaceBlockPlacementPolicy
    dfs.namenode.available-space-block-placement-policy.balanced-space-preference-fraction: 1 
    ```

  2. 滚动重启HDFS

### 详解类AvailableSpaceBlockPlacementPolicy
  
   该类继承自`BlockPlacementPolicyDefault`
   
<div>
<img src="/images/posts/hadoop-source-14/hadoop01.png" height="880" width="1180" />
</div>   
  
  核心方法对比节点

```
  protected int compareDataNode(final DatanodeDescriptor a,
      final DatanodeDescriptor b, boolean isBalanceLocal) {
    if (
	  //以下三个条件任意满足一个认为两个节点是相同
	  //1. 两个节点相同
	  //2. 两个节点使用比率差距小于5%
	  //3. 是否是本地均衡并且当前节点使用率低于50%
	  a.equals(b)
        || Math.abs(a.getDfsUsedPercent() - b.getDfsUsedPercent()) < 5 || ((
        isBalanceLocal && a.getDfsUsedPercent() < 50))) {
      return 0;
    }
	// 如果两个节点使用比率A<B 那么返回A否则返回B
    return a.getDfsUsedPercent() < b.getDfsUsedPercent() ? -1 : 1;
  }
}  
``` 

  负载因子 `private int balancedPreference = (int) (100 * DFS_NAMENODE_AVAILABLE_SPACE_BLOCK_PLACEMENT_POLICY_BALANCED_SPACE_PREFERENCE_FRACTION_DEFAULT);`
  
```
  private DatanodeDescriptor select(DatanodeDescriptor a, DatanodeDescriptor b,
      boolean isBalanceLocal) {
    if (a != null && b != null){
      int ret = compareDataNode(a, b, isBalanceLocal);
      if (ret == 0) {
        return a;
      } else if (ret < 0) {
	   //负载因子balancedPreference 如果负载因子调为1那么基本上数据和大概率落入到比例小的空间磁盘上
        return (RAND.nextInt(100) < balancedPreference) ? a : b;
      } else {
        return (RAND.nextInt(100) < balancedPreference) ? b : a;
      }
    } else {
      return a == null ? b : a;
    }
  }
```  

### 总结

   以上为笔者对`AvailableSpaceBlockPlacementPolicy`的概述,下文继续解析核心可用磁盘逻辑卷存放策略`AvailableSpaceVolumeChoosingPolicy`,希望本文对读者起到一定的帮助.

### 参考

* http://hadoop.apache.org/
* https://issues.apache.org/jira/browse/HDFS-8131