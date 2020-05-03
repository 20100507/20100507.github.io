---
layout: post
title: "Hadoop源码分析-11-Decommission"
date: 2020-02-28
tag: Hadoop源码分析
---

### 前言

     大规模的集群中,会经常出现磁盘损坏需要下线节点.集群下线过程需要遵循下线的规则,切勿直接停止DataNode下线节点.

### 执行步骤

   1. 修改配置文件
     
	 找到dfs.hosts.exclude文件,在该文件中添加要下线的节点.
   
   2. 执行命令
     
	 hdfs dfsadmin -refreshNodes
  
### 源码追踪
    
   相关类涉及到`FSNamesystem`,`DatanodeManager`,`DatanodeAdminManager`,`HeartbeatManager`,DatanodeManager涉及大量的
DataNode相关操作,比如删除DataNode,获取存活的DataNode,注册到NameNode等等.`DatanodeAdminManager`涉及到下线节点,使得节点
进入维护状态,`HeartbeatManager`涉及心跳类,比如获取使用容量，获取Xceiver线程数等等.
   
   
### FSNamesystem
 
 从执行的流程上,输入了执行命令,会进入FSNamesystem的如下方法
 
```
  void refreshNodes() throws IOException {
    String operationName = "refreshNodes";
    checkOperation(OperationCategory.UNCHECKED);
	//检查权限
    checkSuperuserPrivilege(operationName);
    //刷新节点根据给定的配置文件
	getBlockManager().getDatanodeManager().refreshNodes(new HdfsConfiguration());
    //记录审计日志
	logAuditEvent(true, operationName, null);
  }
  
``` 

### DatanodeManager

 接下来进入如下方法,首先刷新配置文件,然后获取写锁,然后刷新DataNode节点

```
  public void refreshNodes(final Configuration conf) throws IOException {
    //刷新配置文件
    refreshHostsReader(conf);
    namesystem.writeLock();
    try {
	  //刷新DataNode节点
      refreshDatanodes();
	  //有可能版本不一致，统计版本
      countSoftwareVersions();
    } finally {
      namesystem.writeUnlock();
    }
  } 
``` 
 
 刷新配置文件

```
  private void refreshHostsReader(Configuration conf) throws IOException {
    if (conf == null) {
      conf = new HdfsConfiguration();
      this.hostConfigManager.setConf(conf);
    }
    this.hostConfigManager.refresh();
  }
  
```  

 刷新DataNode节点方法
 
```
  private void refreshDatanodes() {
    final Map<String, DatanodeDescriptor> copy;
    synchronized (this) {
      copy = new HashMap<>(datanodeMap);
    }
    for (DatanodeDescriptor node : copy.values()) {
      // 如果不在主机列表中设置为disable
      if (!hostConfigManager.isIncluded(node)) {
        node.setDisallowed(true);
      } else {
	    //获取正在处于维护状态的过期时常
        long maintenanceExpireTimeInMS =
            hostConfigManager.getMaintenanceExpirationTimeInMS(node);
		//如果没有过期，则开始进入维护状态
        if (node.maintenanceNotExpired(maintenanceExpireTimeInMS)) {
          datanodeAdminManager.startMaintenance(
              node, maintenanceExpireTimeInMS);
        } else if (hostConfigManager.isExcluded(node)) {
		  //如果在配置文件中添加了排除的节点则排除
          datanodeAdminManager.startDecommission(node);
        } else {
		  //以上都不是则立即停止维护状态和下线状态
          datanodeAdminManager.stopMaintenance(node);
          datanodeAdminManager.stopDecommission(node);
        }
      }
	  //设置升级域
      node.setUpgradeDomain(hostConfigManager.getUpgradeDomain(node));
    }
  }
```
  
### DatanodeAdminManager

  开始下线①
  
```
  public void startDecommission(DatanodeDescriptor node) {
    //如果既不是正在下线也不是处于已经下线
    if (!node.isDecommissionInProgress() && !node.isDecommissioned()) {
	  //开始下线
      hbManager.startDecommission(node);
      //如果正在下线
      if (node.isDecommissionInProgress()) {
        for (DatanodeStorageInfo storage : node.getStorageInfos()) {
          LOG.info("Starting decommission of {} {} with {} blocks",
              node, storage, storage.numBlocks());
        }
		//获取下线开始的时间
        node.getLeavingServiceStatus().setStartTime(monotonicNow());
		//添加到待处理的队列中
        monitor.startTrackingNode(node);
      }
    } else {
      LOG.trace("startDecommission: Node {} in {}, nothing to do.",
          node, node.getAdminState());
    }
  }
```  
  
  定期检测线程
  
```
  void activate(Configuration conf) {
    final int intervalSecs = (int) conf.getTimeDuration(
        DFSConfigKeys.DFS_NAMENODE_DECOMMISSION_INTERVAL_KEY,
        DFSConfigKeys.DFS_NAMENODE_DECOMMISSION_INTERVAL_DEFAULT,
        TimeUnit.SECONDS);
    checkArgument(intervalSecs >= 0, "Cannot set a negative " +
        "value for " + DFSConfigKeys.DFS_NAMENODE_DECOMMISSION_INTERVAL_KEY);

    int blocksPerInterval = conf.getInt(
        DFSConfigKeys.DFS_NAMENODE_DECOMMISSION_BLOCKS_PER_INTERVAL_KEY,
        DFSConfigKeys.DFS_NAMENODE_DECOMMISSION_BLOCKS_PER_INTERVAL_DEFAULT);

    final String deprecatedKey =
        "dfs.namenode.decommission.nodes.per.interval";
    final String strNodes = conf.get(deprecatedKey);
    if (strNodes != null) {
      LOG.warn("Deprecated configuration key {} will be ignored.",
          deprecatedKey);
      LOG.warn("Please update your configuration to use {} instead.",
          DFSConfigKeys.DFS_NAMENODE_DECOMMISSION_BLOCKS_PER_INTERVAL_KEY);
    }

    checkArgument(blocksPerInterval > 0,
        "Must set a positive value for "
        + DFSConfigKeys.DFS_NAMENODE_DECOMMISSION_BLOCKS_PER_INTERVAL_KEY);

    final int maxConcurrentTrackedNodes = conf.getInt(
        DFSConfigKeys.DFS_NAMENODE_DECOMMISSION_MAX_CONCURRENT_TRACKED_NODES,
        DFSConfigKeys
            .DFS_NAMENODE_DECOMMISSION_MAX_CONCURRENT_TRACKED_NODES_DEFAULT);
    checkArgument(maxConcurrentTrackedNodes >= 0, "Cannot set a negative " +
        "value for "
        + DFSConfigKeys.DFS_NAMENODE_DECOMMISSION_MAX_CONCURRENT_TRACKED_NODES);

    Class cls = null;
    try {
      cls = conf.getClass(
          DFSConfigKeys.DFS_NAMENODE_DECOMMISSION_MONITOR_CLASS,
          DatanodeAdminDefaultMonitor.class);
      monitor =
          (DatanodeAdminMonitorInterface)ReflectionUtils.newInstance(cls, conf);
      monitor.setBlockManager(blockManager);
      monitor.setNameSystem(namesystem);
      monitor.setDatanodeAdminManager(this);
    } catch (Exception e) {
      throw new RuntimeException("Unable to create the Decommission monitor " +
          "from "+cls, e);
    }
    executor.scheduleAtFixedRate(monitor, intervalSecs, intervalSecs,
        TimeUnit.SECONDS);
  }  
```

### HeartbeatManager

     
```
  synchronized void startDecommission(final DatanodeDescriptor node) {
    
    if (!node.isAlive()) {
      LOG.info("Dead node {} is decommissioned immediately.", node);
      node.setDecommissioned();
    } else {
      stats.subtract(node);
      node.startDecommission();
      stats.add(node);
    }
  }
```   
 
### 总结

   以上为笔者Decommission总结,感兴趣的读者可以继续深入理解,希望本文对读者起到一定的帮助.

### 参考

* https://github.com/apache/hadoop
* http://hadoop.apache.org/