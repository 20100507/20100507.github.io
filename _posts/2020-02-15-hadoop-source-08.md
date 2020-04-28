---
layout: post
title: "Hadoop源码分析-08-租约"
date: 2020-02-15
tag: Hadoop源码分析
---

### 前言

    在写文件和读取文件的时候，为了保证读写的一致性，如果一个客户端向HDFS写文件，是不允许其他客户端写入的。HDFS使用了租约(Lease)的机制,
	一个客户端写入数据他会持有临时的身份，但是如果身份过期后，需要续约或者需要释放该租约。下面作者聊聊HDFS的租约。

### 租约概念相关类

> * LeaseManager
> * Lease
> * Monitor
> * LeaseRenewer
> * DFSClient


1. `LeaseManager` 该类为核心的租约管理类,其中包含了`Lease`租约和`Monitor`线程类,`LeaseRenewer`包含了`DFSClient`,持有多个`DFSClient`,定期更新租约.
   `FSDirWriteFileOp`为写入HDFS文件的操作类,`FSDirDeleteOp`为删除HDFS相关操作类.


### 结构梳理

* Lease

  `Lease`作为`LeaseManager`一个内部类,每一个租约都会持有租约的持有者,最后一次更新时间,已经该租约持有者对应的INode文件列表

  ```
  class Lease {
    //租约的持有者
    private final String holder;
    //最后一次的更新时间
    private long lastUpdate;
    //持有者对应的INode 文件列表
    private final HashSet<Long> files = new HashSet<>();
  }
  ```

  如下图为其结构:
  
<div>
<img src="/images/posts/hadoop-source-08/hadoop01.png" height="320" width="560" />
</div>

* Monitor

  `LeaseManager`内部类 `Monitor`该类是一个线程类，定期的检查租约是否过期，如果过期需要去处理这些过期的租约

```
public void run() {
 //线程是否运行中并且fsnamesystem是否运行中
  for(; shouldRunMonitor && fsnamesystem.isRunning(); ) {
    boolean needSync = false;
    try {
      //获取fsnamesystem写锁
      fsnamesystem.writeLockInterruptibly();
      try {
       //是否是处于安全模式
        if (!fsnamesystem.isInSafeMode()) {
          needSync = checkLeases();
        }
      } finally {
      //释放写锁
        fsnamesystem.writeUnlock("leaseManager");
        // lease reassignments should to be sync'ed.
		//是否需要异步同步editlogs
        if (needSync) {
          fsnamesystem.getEditLog().logSync();
        }
      }
        //每隔2秒 检查一次
      Thread.sleep(fsnamesystem.getLeaseRecheckIntervalMs());
    } catch(InterruptedException ie) {
      LOG.debug("{} is interrupted", name, ie);
    } catch(Throwable e) {
      LOG.warn("Unexpected throwable: ", e);
    }
  }
}
```

  看一下checkLeases方法

```
synchronized boolean checkLeases() {
  boolean needSync = false;
  assert fsnamesystem.hasWriteLock();

  long start = monotonicNow();
  //如果租约队列不为空并且超过了最老的租约超过了硬限制并且没有达到租约检查的持有锁释放租约的最长时间
  while(!sortedLeases.isEmpty() &&
      sortedLeases.first().expiredHardLimit()
      && !isMaxLockHoldToReleaseLease(start)) {
    Lease leaseToCheck = sortedLeases.first();
    LOG.info("{} has expired hard limit", leaseToCheck);

    final List<Long> removing = new ArrayList<>();
    // need to create a copy of the oldest lease files, because
    // internalReleaseLease() removes files corresponding to empty files,
    // i.e. it needs to modify the collection being iterated over
    // causing ConcurrentModificationException
    Collection<Long> files = leaseToCheck.getFiles();
    Long[] leaseINodeIds = files.toArray(new Long[files.size()]);
    FSDirectory fsd = fsnamesystem.getFSDirectory();
    String p = null;
    String newHolder = getInternalLeaseHolder();
     .... 
    }
```

  继续查看while里面结构
 
```
   final List<Long> removing = new ArrayList<>();
Collection<Long> files = leaseToCheck.getFiles();
Long[] leaseINodeIds = files.toArray(new Long[files.size()]);
FSDirectory fsd = fsnamesystem.getFSDirectory();
String p = null;
String newHolder = getInternalLeaseHolder();
for(Long id : leaseINodeIds) {
  try {
    INodesInPath iip = INodesInPath.fromINode(fsd.getInode(id));
    p = iip.getPath();
    // Sanity check to make sure the path is correct
    if (!p.startsWith("/")) {
      throw new IOException("Invalid path in the lease " + p);
    }
	//获取最远的INode 判断是否该文件已经被删除，如果被删除了直接从租约的Map中移除
    final INodeFile lastINode = iip.getLastINode().asFile();
    if (fsnamesystem.isFileDeleted(lastINode)) {
      // INode referred by the lease could have been deleted.
      removeLease(lastINode.getId());
      continue;
    }
    boolean completed = false;
    try {
	  //内部释放租约,这一块逻辑相对来说更加复杂,后面来讨论
      completed = fsnamesystem.internalReleaseLease(
          leaseToCheck, p, iip, newHolder);
    } catch (IOException e) {
      LOG.warn("Cannot release the path {} in the lease {}. It will be "
          + "retried.", p, leaseToCheck, e);
      continue;
    }
    if (LOG.isDebugEnabled()) {
      if (completed) {
        LOG.debug("Lease recovery for inode {} is complete. File closed"
            + ".", id);
      } else {
        LOG.debug("Started block recovery {} lease {}", p, leaseToCheck);
      }
    }
    // If a lease recovery happened, we need to sync later.
    if (!needSync && !completed) {
      needSync = true;
    }
  } catch (IOException e) {
    LOG.warn("Removing lease with an invalid path: {},{}", p,
        leaseToCheck, e);
    //出现无效的路径就加入到removing List中
    removing.add(id);
  }
    //获取最旧的队列中的值
  if (isMaxLockHoldToReleaseLease(start)) {
    LOG.debug("Breaking out of checkLeases after {} ms.",
        fsnamesystem.getMaxLockHoldToReleaseLeaseMs());
    break;
  }
}
   //循环移除这些无效的租约
  for(Long id : removing) {
     removeLease(leaseToCheck, id);
   }
 }
```

* LeaseManager

 涉及到如下三个成员变量,租约持有者到Lease的映射,租约TreeSet列表,以及INodeID到租约的映射
 
```
 //租约持有者到Lease的映射
 private final SortedMap<String, Lease> leases = new TreeMap<>();
 //租约TreeSet列表
 private final NavigableSet<Lease> sortedLeases = new TreeSet<>(
    new Comparator<Lease>() {
      @Override
      public int compare(Lease o1, Lease o2) {
        if (o1.getLastUpdate() != o2.getLastUpdate()) {
          return Long.signum(o1.getLastUpdate() - o2.getLastUpdate());
        } else {
          return o1.holder.compareTo(o2.holder);
        }
      }
});
 //INodeID到租约的映射
private final TreeMap<Long, Lease> leasesById = new TreeMap<>();
``` 

至此: 关系梳理如下:

<div>
<img src="/images/posts/hadoop-source-08/hadoop02.png" height="320" width="860" />
</div>

至此: 检测租约的流程如下:


<div>
<img src="/images/posts/hadoop-source-08/hadoop03.png" height="320" width="860" />
</div>

* LeaseRenewer

  `LeaseRenewer` 定期更新DFSClient用户持有的租约.每个用户有一个`LeaseRenewer`对象,该对象中维护了一个DFSClient List,定期给客户端续约.如果DFSClient
  都操作文件流都关闭了,DFSClient将被`LeaseRenewer` 从其列表中移除.

  
### 租约操作类中使用

> * FSDirWriteFileOp
> * FSDirDeleteOp

  `FSDirWriteFileOp` 添加文件&覆盖重写方法中涉及到租约续约和添加租约以及移除租约如下
 
``` 
  static HdfsFileStatus startFile(
      FSNamesystem fsn, INodesInPath iip,
      PermissionStatus permissions, String holder, String clientMachine,
      EnumSet<CreateFlag> flag, boolean createParent,
      short replication, long blockSize,
      FileEncryptionInfo feInfo, INode.BlocksMapUpdateInfo toRemoveBlocks,
      boolean shouldReplicate, String ecPolicyName, String storagePolicy,
      boolean logRetryEntry)
      throws IOException {
    assert fsn.hasWriteLock();
    boolean overwrite = flag.contains(CreateFlag.OVERWRITE);
    boolean isLazyPersist = flag.contains(CreateFlag.LAZY_PERSIST);

    final String src = iip.getPath();
    FSDirectory fsd = fsn.getFSDirectory();

    if (iip.getLastINode() != null) {
      if (overwrite) {
        List<INode> toRemoveINodes = new ChunkedArrayList<>();
        List<Long> toRemoveUCFiles = new ChunkedArrayList<>();
        long ret = FSDirDeleteOp.delete(fsd, iip, toRemoveBlocks,
                                        toRemoveINodes, toRemoveUCFiles, now());
        if (ret >= 0) {
          iip = INodesInPath.replace(iip, iip.length() - 1, null);
          FSDirDeleteOp.incrDeletedFileCount(ret);
		  //删除旧的文件移除租约
          fsn.removeLeasesAndINodes(toRemoveUCFiles, toRemoveINodes, true);
        }
      } else {
        //如果超出租约的软限制/续约
        fsn.recoverLeaseInternal(FSNamesystem.RecoverLeaseOp.CREATE_FILE, iip,
                                 src, holder, clientMachine, false);
        throw new FileAlreadyExistsException(src + " for client " +
            clientMachine + " already exists");
      }
    }
     ....
	//新建文件的时候添加租约该文件是UC状态
    fsn.leaseManager.addLease(
        newNode.getFileUnderConstructionFeature().getClientName(),
        newNode.getId());
    if (feInfo != null) {
      FSDirEncryptionZoneOp.setFileEncryptionInfo(fsd, iip, feInfo,
          XAttrSetFlag.CREATE);
    }
     ...
    return FSDirStatAndListingOp.getFileInfo(fsd, iip, false, false);
  }
```

* FSDirDeleteOp
 
  `FSDirDeleteOp` 删除文件操作,删除文件后需要释放租约.
  
```
 static BlocksMapUpdateInfo deleteInternal(
      FSNamesystem fsn, INodesInPath iip, boolean logRetryCache)
      throws IOException {
    assert fsn.hasWriteLock();
    if (NameNode.stateChangeLog.isDebugEnabled()) {
      NameNode.stateChangeLog.debug("DIR* NameSystem.delete: " + iip.getPath());
    }

    FSDirectory fsd = fsn.getFSDirectory();
    BlocksMapUpdateInfo collectedBlocks = new BlocksMapUpdateInfo();
    List<INode> removedINodes = new ChunkedArrayList<>();
    List<Long> removedUCFiles = new ChunkedArrayList<>();

    long mtime = now();
    long filesRemoved = delete(
        fsd, iip, collectedBlocks, removedINodes, removedUCFiles, mtime);
    if (filesRemoved < 0) {
      return null;
    }
    fsd.getEditLog().logDelete(iip.getPath(), mtime, logRetryCache);
    incrDeletedFileCount(filesRemoved);
    //释放租约
    fsn.removeLeasesAndINodes(removedUCFiles, removedINodes, true);
    if (NameNode.stateChangeLog.isDebugEnabled()) {
      NameNode.stateChangeLog.debug(
          "DIR* Namesystem.delete: " + iip.getPath() +" is removed");
    }
    return collectedBlocks;
  }
```

### 租约在Block级别控制


> * FSNamesystem

 `FSNamesystem`类中持久化文件的元数据`fsync`方法,添加新的Block`getAdditionalDatanode`,`checkLease`检查当前操作Block的
 用户是不是当前租约的持有者.

 ```
  INodeFile checkLease(INodesInPath iip, String holder, long fileId)
      throws LeaseExpiredException, FileNotFoundException {
    final String owner = file.getFileUnderConstructionFeature().getClientName();
	//检查当前操作Block的用户是不是当前租约的持有者.
    if (holder != null && !owner.equals(holder)) {
      throw new LeaseExpiredException("Client (=" + holder
          + ") is not the lease owner (=" + owner + ": "
          + leaseExceptionString(src, fileId, holder));
    }
    return file;
  }
 ```
 
### 总结

   以上为笔者总结部分租约的源代码,感兴趣的读者可以继续深入学习,希望本文对读者起到一定的帮助.

### 参考

* https://github.com/apache/hadoop
* http://hadoop.apache.org/
* http://blog.csdn.net/androidlushangderen/article/details/52850349
