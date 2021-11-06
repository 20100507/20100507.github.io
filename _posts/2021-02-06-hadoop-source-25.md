---
layout: post
title: "Hadoop源码分析-24-Trash"
date: 2021-02-06
tag: Hadoop源码分析
---

### 前言

       HDFS经常删除数据是非常正常的事情，但是偶尔的误删也是存在的，因此回收站的存在是非常有必要的，本文基于hadoop2.8.5深入到hadoop内部源代码进行走读，分析客户端和服务端是如何配合完成回收站这一个功能的，并且下文会提供回收站如何自定义修改.以避开hadoop回收站默认路径硬编码的问题.

### 设计总思路

> 1. 默认的3副本放置策略状态
> 2. 升级域3副本放置策略状态
> 3. 机架组3副本放置策略状态
> 4. 默认第一次放置策略状态

### 客户端流程

​    主要核心类如下

> 1.  BlockPlacementStatusDefault  默认存放策略状态
> 2.  BlockPlacementStatusWithUpgradeDomain 升级域存放策略状态
> 3.  BlockPlacementStatusWithNodeGroup  机架组存放策略状态
> 4. BlockPlacementStatusSatisfied  副本满意度存放策略

   结构图如下： 

<div>
<img src="/images/posts/hadoop-source-24/hadoop01.png" height="480" width="780" />
</div>


### 服务端流程

* BlockPlacementStatusDefault 

  判断副本放置是否满意

```java
public boolean isPlacementPolicySatisfied() {
  //当前要求机架数小于当前的机架数据 或者当前机架数大于总的机架数  
  return requiredRacks <= currentRacks || currentRacks >= totalRacks;
}
```

* BlockPlacementStatusDefault 

   获取需要额外的添加的副本数

```java
public int getAdditionalReplicasRequired() {
  //如果满足 需要额外的副本数据为0  
  if (isPlacementPolicySatisfied()) {
    return 0;
  } else {
    //否则要求的机架数据减去当前的机架数  
    return requiredRacks - currentRacks;
  }
}
```

* BlockPlacementStatusWithUpgradeDomain 

  判断升级域副本放置是否满意

```java
private boolean isUpgradeDomainPolicySatisfied() {
  //副本数是否小于升级域个数
  if (numberOfReplicas <= upgradeDomainFactor) {
    //副本数小于升级域的大小  
    return (numberOfReplicas <= upgradeDomains.size());
  } else {
    //升级域大小已经超过了升级域的个数
    return upgradeDomains.size() >= upgradeDomainFactor;
  }
}
```

* BlockPlacementStatusWithUpgradeDomain 

  额外的需要的添加的最少副本数

```java
 public int getAdditionalReplicasRequired() {
  if (isPlacementPolicySatisfied()) {
    return 0;
  } else {
    //获取没有升级域时需要添加的额外的副本  
    int parent = parentBlockPlacementStatus.getAdditionalReplicasRequired();
    int child;
     //如果副本数小于升级域大小 使用副本数减去正在升级域的数量
    if (numberOfReplicas <= upgradeDomainFactor) {
      child = numberOfReplicas - upgradeDomains.size();
    } else {
      //否则使用升级域减去升级域数量  
      child = upgradeDomainFactor - upgradeDomains.size();
    }
    //相比较获取最大的值
    return Math.max(parent, child);
  }
}
```

* BlockPlacementStatusWithUpgradeDomain 

  同时判断没有升级域和有升级域的是否副本满足

```java
public boolean isPlacementPolicySatisfied() {
  //同时满足没有升级域时的副本放置条件以及携带了升级域后的副本放置策略    
  return parentBlockPlacementStatus.isPlacementPolicySatisfied() &&
      isUpgradeDomainPolicySatisfied();
}
```

### 核心类跟踪源代码(2.8.5)

### 总结

    以上为部分副本放置状态类的源码解析,希望可以对读者起到帮助作用.

### 参考

* http://hadoop.apache.org