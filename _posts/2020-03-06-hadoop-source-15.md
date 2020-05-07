---
layout: post
title: "Hadoop源码分析-15-AvailableSpaceVolumeChoosingPolicy"
date: 2020-03-06
tag: Hadoop源码分析
---

### 前言

       当集群规模庞大的时候,难免会存在异构的节点,比如磁盘的大小差异巨大.上文我们提到了副本选择策略,本文我们谈谈磁盘选择策略.默认情况下
  HDFS磁盘选择策略为轮询的策略.也就是随机轮询选择.
       为防止异构的磁盘被提前写满,采用`AvailableSpaceVolumeChoosingPolicy`优先写入磁盘空间大的磁盘中.

### 配置使用AvailableSpaceVolumeChoosingPolicy

  1. 修改配置hdfs-site.xml文件
    
	```
    dfs.datanode.fsdataset.volume.choosing.policy: org.apache.hadoop.hdfs.server.datanode.fsdataset.AvailableSpaceVolumeChoosingPolicy
    dfs.datanode.available-space-volume-choosing-policy.balanced-space-preference-fraction: 0.99f
    dfs.datanode.available-space-volume-choosing-policy.balanced-space-threshold: 1073741824
    ```

  2. 滚动重启HDFS

### AvailableSpaceVolumeChoosingPolicy内部类
  
   该类实现接口`VolumeChoosingPolicy`
   
<div>
<img src="/images/posts/hadoop-source-15/hadoop01.png" height="780" width="980" />
</div>   
  
  该类的内部类`AvailableSpaceVolumePair`
 
```
  private List<V> extractVolumesFromPairs(List<AvailableSpaceVolumePair> volumes) {
    List<V> ret = new ArrayList<V>();
    for (AvailableSpaceVolumePair volume : volumes) {
      ret.add(volume.getVolume());
    }
    return ret;
  }
```
  
  
  该类的内部类`AvailableSpaceVolumeList`
  
  1. 判断是否所有磁盘都在阈值内

```
    public boolean areAllVolumesWithinFreeSpaceThreshold() {
    long leastAvailable = Long.MAX_VALUE;
    long mostAvailable = 0;
    for (AvailableSpaceVolumePair volume : volumes) {
      leastAvailable = Math.min(leastAvailable, volume.getAvailable());
      mostAvailable = Math.max(mostAvailable, volume.getAvailable());
    }
    return (mostAvailable - leastAvailable) < balancedSpaceThreshold;
    }
```

  2. 获取最低磁盘使用量
  
```
    private long getLeastAvailableSpace() {
      long leastAvailable = Long.MAX_VALUE;
      for (AvailableSpaceVolumePair volume : volumes) {
        leastAvailable = Math.min(leastAvailable, volume.getAvailable());
      }
      return leastAvailable;
    }
```  
  
  3. 获取给定阈值高容量可用磁盘
  
```
   public List<AvailableSpaceVolumePair> getVolumesWithHighAvailableSpace() {
      long leastAvailable = getLeastAvailableSpace();
      List<AvailableSpaceVolumePair> ret = new ArrayList<AvailableSpaceVolumePair>();
      for (AvailableSpaceVolumePair volume : volumes) {
        if (volume.getAvailable() > leastAvailable + balancedSpaceThreshold) {
          ret.add(volume);
        }
      }
      return ret;
    }  
```

  整体结构:

<div>
<img src="/images/posts/hadoop-source-15/hadoop02.png" height="780" width="980" />
</div>

### AvailableSpaceVolumeChoosingPolicy详解

  1. 核心选择磁盘逻辑
  
```
  private V doChooseVolume(final List<V> volumes, long replicaSize,
      String storageId) throws IOException {
    AvailableSpaceVolumeList volumesWithSpaces = new AvailableSpaceVolumeList(volumes);
	//判断是否所有磁盘都在阈值内,如果在阈值内就采用轮询的方式写入
    if (volumesWithSpaces.areAllVolumesWithinFreeSpaceThreshold()) {
      V volume = roundRobinPolicyBalanced.chooseVolume(volumes, replicaSize,
          storageId);
      if (LOG.isDebugEnabled()) {
        LOG.debug("All volumes are within the configured free space balance " +
            "threshold. Selecting " + volume + " for write of block size " +
            replicaSize);
      }
      return volume;
    } else {
      V volume = null;
      // If none of the volumes with low free space have enough space for the
      // replica, always try to choose a volume with a lot of free space.
      long mostAvailableAmongLowVolumes = volumesWithSpaces
          .getMostAvailableSpaceAmongVolumesWithLowAvailableSpace();
      List<V> highAvailableVolumes = extractVolumesFromPairs(
          volumesWithSpaces.getVolumesWithHighAvailableSpace());
      List<V> lowAvailableVolumes = extractVolumesFromPairs(
          volumesWithSpaces.getVolumesWithLowAvailableSpace());
      
      float preferencePercentScaler =
          (highAvailableVolumes.size() * balancedPreferencePercent) +
          (lowAvailableVolumes.size() * (1 - balancedPreferencePercent));
      float scaledPreferencePercent =
          (highAvailableVolumes.size() * balancedPreferencePercent) /
          preferencePercentScaler;
	  //如果最大可用的磁盘剩余容量小于副本大小或者在给定的阈值内
	  //写入高剩余容量的磁盘 
      if (mostAvailableAmongLowVolumes < replicaSize ||
          random.nextFloat() < scaledPreferencePercent) {
        volume = roundRobinPolicyHighAvailable.chooseVolume(
            highAvailableVolumes, replicaSize, storageId);
        if (LOG.isDebugEnabled()) {
          LOG.debug("Volumes are imbalanced. Selecting " + volume +
              " from high available space volumes for write of block size "
              + replicaSize);
        }
      } else {
	    //否则写入低剩余容量的磁盘
        volume = roundRobinPolicyLowAvailable.chooseVolume(
            lowAvailableVolumes, replicaSize, storageId);
        if (LOG.isDebugEnabled()) {
          LOG.debug("Volumes are imbalanced. Selecting " + volume +
              " from low available space volumes for write of block size "
              + replicaSize);
        }
      }
      return volume;
    }
  }
```

### 总结

   以上为笔者对`AvailableSpaceVolumeChoosingPolicy`的概述,希望本文对读者起到一定的帮助.

### 参考

* http://hadoop.apache.org/