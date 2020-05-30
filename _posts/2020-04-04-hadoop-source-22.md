---
layout: post
title: "Hadoop源码分析-22-磁盘损坏DataNode进程退出"
date: 2020-04-04
tag: Hadoop源码分析
---

### 前言

    在运维大型HDFS中，磁盘损坏是很正常的一件事。DataNode默认若有一块磁盘损坏，       DataNode进程直接退出，如果   稍加对HDFS有了解的人都知道，DataNode进程退出后，该   节点的副本将被重新复制。倘若集群中单台节点的存储的数   据量是100T的话，意味着单台节   点100T数据需要移动。这对于集群的影响还是比较大的。
      因此我们在HDFS中可以修改当该节点的DataNode磁盘损坏个数为-1或者自定义磁盘个     数，该节点的DataNode进程退出.

### 配置部署

  * 配置修改[-1代表只要有一个磁盘就DataNode进程就不退出]
  ```
<property>
   <name>dfs.datanode.failed.volumes.tolerated</name>
    <value>-1</value>
</property>
  ```

* 重启DataNode
```
 hadoop-daemon.sh stop datanode
 hadoop-daemon.sh start datanode
```

### 检测磁盘损坏源代码


  * DataNode类 main方法入口

```java
public static void main(String args[]) {
  secureMain(args, null);
}

public static void secureMain(String args[], SecureResources resources) {
  try {
    DataNode datanode = createDataNode(args, null, resources);
     ...
  }
}

public static DataNode createDataNode(String args[], Configuration conf,
    SecureResources resources) throws IOException {
  if (dn != null) {
    dn.runDatanodeDaemon();
  }
  ...
}

public void runDatanodeDaemon() throws IOException {
  blockPoolManager.startAll();
  ...   
}
```

* BlockPoolManager类 BP池管理类

```java
 synchronized void startAll() throws IOException {
  try {
    UserGroupInformation.getLoginUser().doAs(
        new PrivilegedExceptionAction<Object>() {
          @Override
          public Object run() throws Exception {
            for (BPOfferService bpos : offerServices) {
              //启动BPOfferService  
              bpos.start();
            }
            return null;
          }
        });
  } catch (InterruptedException ex) {
     ...
  }
}
```

* BPOfferService 类 start方法

```java
 void start() {
  for (BPServiceActor actor : bpServices) {
    //启动所有的BPServiceActor   
    actor.start();
  }
}
```
* BPServiceActor类 start方法

```java
void start() {
  bpThread = new Thread(this);
  bpThread.setDaemon(true);
  //启动BPServiceActor线程  
  bpThread.start();
  ...
}

public void run() {
  try {
    while (true) {
      try {
        //和NN握手  
        connectToNNAndHandshake();
        break;
      } catch (IOException ioe) {
      }
     ...
}
```

  * BPServiceActor 类 握手方法

```java
private void connectToNNAndHandshake() throws IOException {
  ...
  //校验并且设置namespace
  bpos.verifyAndSetNamespaceInfo(this, nsInfo);
  ...
}
```
 * BPOfferService 类 中校验并且设置namespace信息

```java
void verifyAndSetNamespaceInfo(BPServiceActor actor, NamespaceInfo nsInfo)
  throws IOException {
  writeLock();
  try {
    if (setNamespaceInfo(nsInfo) == null) {
        ...
      try {
        //初始化块池  
        dn.initBlockPool(this);
        success = true;
      } finally {
         ...
      }
    }
  } finally {
    writeUnlock();
  }
}
```

 * DataNode 类中 初始化BP池

```java
void initBlockPool(BPOfferService bpos) throws IOException {
  NamespaceInfo nsInfo = bpos.getNamespaceInfo();
  //检查磁盘错误
  checkDiskError();
}
```
* DataNode 类中 检查磁盘错误方法

   这里有两个处理逻辑

  1. 检查所有的磁盘
  2. 处理不健康的磁盘

```java
public void checkDiskError() throws IOException {
  Set<FsVolumeSpi> unhealthyVolumes;
  try {
    //检查所有的磁盘  
    unhealthyVolumes = volumeChecker.checkAllVolumes(data);
    lastDiskErrorCheck = Time.monotonicNow();
  } catch (InterruptedException e) {
     ...
  }
  if (unhealthyVolumes.size() > 0) {
    //处理磁盘错误    
    handleVolumeFailures(unhealthyVolumes);
  } else {
    LOG.debug("checkDiskError encountered no failures");
  }
}
```

*  DatasetVolumeChecker 类 检查所有的磁盘方法

```java
public Set<FsVolumeSpi> checkAllVolumes(
    final FsDatasetSpi<? extends FsVolumeSpi> dataset)
    throws InterruptedException {
  final long gap = timer.monotonicNow() - lastAllVolumesCheck;
  if (gap < minDiskCheckGapMs) {
    numSkippedChecks.incrementAndGet();
    LOG.trace(
        "Skipped checking all volumes, time since last check {} is less " +
        "than the minimum gap between checks ({} ms).",
        gap, minDiskCheckGapMs);
    return Collections.emptySet();
  }
  //获取所有的磁盘引用
  final FsDatasetSpi.FsVolumeReferences references =
      dataset.getFsVolumeReferences();

  if (references.size() == 0) {
    LOG.warn("checkAllVolumesAsync - no volumes can be referenced");
    return Collections.emptySet();
  }

  lastAllVolumesCheck = timer.monotonicNow();
  final Set<FsVolumeSpi> healthyVolumes = new HashSet<>();
  final Set<FsVolumeSpi> failedVolumes = new HashSet<>();
  final Set<FsVolumeSpi> allVolumes = new HashSet<>();
  //所有待检查的磁盘
  final AtomicLong numVolumes = new AtomicLong(references.size());
  //采用闭锁 等待所有的检查都完毕然后返回结果  
  final CountDownLatch latch = new CountDownLatch(1);

  for (int i = 0; i < references.size(); ++i) {
    final FsVolumeReference reference = references.getReference(i);
    Optional<ListenableFuture<VolumeCheckResult>> olf =
        delegateChecker.schedule(reference.getVolume(), IGNORED_CONTEXT);
    LOG.info("Scheduled health check for volume {}", reference.getVolume());
    if (olf.isPresent()) {
      allVolumes.add(reference.getVolume());
      //开启异步回调 检查 DataNode启动的时候开始检查 周期性检查 
      Futures.addCallback(olf.get(),
          new ResultHandler(reference, healthyVolumes, failedVolumes,
              numVolumes, new Callback() {
                @Override
                public void call(Set<FsVolumeSpi> ignored1,
                                 Set<FsVolumeSpi> ignored2) {
                  latch.countDown();
                }
              }), MoreExecutors.directExecutor());
    } else {
      IOUtils.cleanup(null, reference);
      if (numVolumes.decrementAndGet() == 0) {
        latch.countDown();
      }
    }
  }

  // Wait until our timeout elapses, after which we give up on
  // the remaining volumes.
  if (!latch.await(maxAllowedTimeForCheckMs, TimeUnit.MILLISECONDS)) {
    LOG.warn("checkAllVolumes timed out after {} ms",
        maxAllowedTimeForCheckMs);
  }

  numSyncDatasetChecks.incrementAndGet();
  synchronized (this) {
    // All volumes that have not been detected as healthy should be
    // considered failed. This is a superset of 'failedVolumes'.
    //
    // Make a copy under the mutex as Sets.difference() returns a view
    // of a potentially changing set.
    //返回所有不健康的磁盘
    return new HashSet<>(Sets.difference(allVolumes, healthyVolumes));
  }
}
```

 *  DataNode 类 处理磁盘错误

 ```java
  private void handleDiskError(String failedVolumes, int failedNumber) {
    //判断是否有足够的资源  
    final boolean hasEnoughResources = data.hasEnoughResource();
    LOG.warn("DataNode.handleDiskError on: " +
        "[{}] Keep Running: {}", failedVolumes, hasEnoughResources);
    
    // If we have enough active valid volumes then we do not want to 
    // shutdown the DN completely.
    int dpError = hasEnoughResources ? DatanodeProtocol.DISK_ERROR  
                                     : DatanodeProtocol.FATAL_DISK_ERROR; 
    //更新指标数据  
    metrics.incrVolumeFailures(failedNumber);

    // 通知NN
    for(BPOfferService bpos: blockPoolManager.getAllNamenodeThreads()) {
      bpos.trySendErrorReport(dpError, failedVolumes);
    }
    
    if(hasEnoughResources) {
      //如果认为有足够的资源调度所有的块 不退出DataNode
      scheduleAllBlockReport(0);
      return;
    }
    //否则退出DataNode
    LOG.warn("DataNode is shutting down due to failed volumes: ["
        + failedVolumes + "]");
    //修改标志位DataNode 则退出  
    shouldRun = false;
  }
 ```

 * FSDatasetImpl 类中 hasEnoughResource方法

 ```java
  public boolean hasEnoughResource() {
    //如果配置  dfs.datanode.failed.volumes.tolerated 为-1的话 则认为只要有一个磁盘正常就正常
    if (volFailuresTolerated == DataNode.MAX_VOLUME_FAILURE_TOLERATED_LIMIT) {
      return volumes.getVolumes().size() >= 1;
    } else {
      //否则 获取失败的磁盘和给定的允许失败的磁盘数进行比较  
      return getNumFailedVolumes() <= volFailuresTolerated;
    }
  }
 ```
### 总结

       本博客分析了磁盘损坏DataNode详细逻辑，希望对读者可以起到一定的帮助作用.在生产      环境笔者建议读者在配置容忍最大磁盘损坏数时，按照自己的每个节点磁盘个数合理配置容忍数。    对于大型集群还是有一定的帮助作用的。


### 参考

* http://hadoop.apache.org
* http://www.github.com/apache/hadoop/hadoop-hdfs