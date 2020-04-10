---
layout: post
title: "Hadoop源码分析-01-HDFS启动过程"
date: 2020-01-24
tag: Hadoop源码分析
---

### 前言
    
	   在HDFS启动的过程中,我们可以在webUI观察到,Startup Progress.笔者以最新的源代码对这一部分源码做一个简要的分析,加深对HDFS启动
	加载的过程做一个更加深入的理解.
	   
		  
### 启动过程剖析

**启动阶段(Phase)**

> 1. LOADING_FSIMAGE
> 2. LOADING_EDITS
> 3. SAVING_CHECKPOINT
> 4. SAFEMODE

<br/>

```
public enum Phase {
  LOADING_FSIMAGE("LoadingFsImage", "Loading fsimage"),
  LOADING_EDITS("LoadingEdits", "Loading edits"),
  SAVING_CHECKPOINT("SavingCheckpoint", "Saving checkpoint"),
  SAFEMODE("SafeMode", "Safe mode");
}
```
如上所述: 启动阶段分为把fsimage文件加载到内存,把edits加载到内存并且把edits文件中对应的操作放入内存中
保存新的检查点,最后namenode进入安全模式节点,直到等待DataNode把数据汇报完毕,退出安全模式.

**每个阶段的状态(Status)**

> 1. PENDING
> 2. RUNNING
> 3. COMPLETE

<br/>

```
public enum Status {
  PENDING,
  RUNNING,
  COMPLETE
}
```

每个阶段分为`等待中`,`运行中`,`已经完成`

**每个阶段的下一步(StepType)**

> 1. AWAITING_REPORTED_BLOCKS
> 2. DELEGATION_KEYS
> 3. DELEGATION_TOKENS
> 4. INODES
> 5. CACHE_POOLS
> 6. CACHE_ENTRIES
> 7. ERASURE_CODING_POLICIES

<br/>

```
public enum StepType {

  AWAITING_REPORTED_BLOCKS("AwaitingReportedBlocks", "awaiting reported blocks"),
  DELEGATION_KEYS("DelegationKeys", "delegation keys"),
  DELEGATION_TOKENS("DelegationTokens", "delegation tokens"),
  INODES("Inodes", "inodes"),
  CACHE_POOLS("CachePools", "cache pools"),
  CACHE_ENTRIES("CacheEntries", "cache entries"),
  ERASURE_CODING_POLICIES("ErasureCodingPolicies", "erasure coding policies");
}
```

<br/>

每个阶段有下一步不同的类型,`等待块汇报` `INODES加载` `NameNode和键和令牌的操作`,`加载缓冲池`,`缓存实体`和`纠删码加载`


**PhaseTracking&StepTracking 阶段和下一步跟踪**

```
final class PhaseTracking extends AbstractTracking{}
final class StepTracking extends AbstractTracking{}
```

PhaseTracking,StepTracking全部继承自AbstractTracking用来统计每一步执行的时间.

```
@InterfaceAudience.Private
final class PhaseTracking extends AbstractTracking {
  String file;
  long size = Long.MIN_VALUE;
  final ConcurrentMap<Step, StepTracking> steps =
    new ConcurrentHashMap<Step, StepTracking>();
}
```

从`ConcurrentMap<Step, StepTracking>`可以看出每一个Phase包含多个Step


**StartupProgress 启动程序类**

```
  final Map<Phase, PhaseTracking> phases =
    new ConcurrentHashMap<Phase, PhaseTracking>();
```

如上代码片段说明每一个`StartupProgress`包含多个`Phase`.

```
  private StepTracking lazyInitStep(Phase phase, Step step) {
    ConcurrentMap<Step, StepTracking> steps = phases.get(phase).steps;
    if (!steps.containsKey(step)) {
      steps.putIfAbsent(step, new StepTracking());
    }
    return steps.get(step);
  }
```

如上代码片段,在一个phases的Map中存放了多个Step的Map值,到这里我们基本上就可以梳理出来
HDFS启动过程的结构了.

### 启动系统结构梳理图

**启动过程代码结构如下**

<div align="left">
<img src="/images/posts/hadoop-source-01/HDFS-source-01.jpg" height="920" width="1180" />
</div>

启动程序开始后,先执行每一个Phase,每一个`Phase`中包含多个`Step`.

### 总结

	以上为笔者启动阶段源代码结构.启动还涉及到各个阶段的时间统计等等,笔者在此没有过多的阐述,希望读者可以通过阅读源码加以连接.
	希望本文对读者起到帮助作用.
