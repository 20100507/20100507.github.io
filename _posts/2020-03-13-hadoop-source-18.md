---
layout: post
title: "Hadoop源码分析-18-DataNodeDiskMetrics"
date: 2020-03-13
tag: Hadoop源码分析
---

### 前言

     DataNodeDiskMetric是磁盘慢盘检测工具,可以把DataNode慢盘对外以接口的方式暴露,方面集群管理人员对磁盘的读写现状
  有一个清晰的认识.本文我们简单分析一下慢盘检测的机制,其中包括离群值计算概念.
  
### JMX接口暴露

  1. 访问http://datanode ip:9864/jmx 如下截图,可以看到慢盘检测后获取慢盘的数组
  
<div>
<img src="/images/posts/hadoop-source-18/hadoop01.png" height="380" width="580" />
</div>     

### DataNodeDiskMetrics源代码

  执行流程如下
  
> 1. 构造方法中new OutliterDetector【离群检测】,启动检测线程
> 2. 获取DataNode上磁盘的元数据操作,真实数据的读取写入上一次指标数据
> 3. 每次线程轮询一次就更新一次离群值


  构造方法源代码:
  
```
    public DataNodeDiskMetrics(DataNode dn, long diskOutlierDetectionIntervalMs) {
    //获取DataNode
	this.dn = dn;
    //线程运行的间隔
	this.detectionInterval = diskOutlierDetectionIntervalMs;
    //new 离群检测
	slowDiskDetector = new OutlierDetector(MIN_OUTLIER_DETECTION_DISKS,
        SLOW_DISK_LOW_THRESHOLD_MS);
	//是否需要一直后台运行
    shouldRun = true;
	//启动线程
    startDiskOutlierDetectionThread();
  }
```  

  启动线程核心代码:
  
```
  private void startDiskOutlierDetectionThread() {
    slowDiskDetectionDaemon = new Daemon(new Runnable() {
      @Override
      public void run() {
        while (shouldRun) {
          if (dn.getFSDataset() != null) {
            Map<String, Double> metadataOpStats = Maps.newHashMap();
            Map<String, Double> readIoStats = Maps.newHashMap();
            Map<String, Double> writeIoStats = Maps.newHashMap();
            FsDatasetSpi.FsVolumeReferences fsVolumeReferences = null;
            try {
			  //获取所有磁盘引用
              fsVolumeReferences = dn.getFSDataset().getFsVolumeReferences();
              Iterator<FsVolumeSpi> volumeIterator = fsVolumeReferences
                  .iterator();
              while (volumeIterator.hasNext()) {
			    //获取所有的磁盘
                FsVolumeSpi volume = volumeIterator.next();
                //获取磁盘上的指标数据
				DataNodeVolumeMetrics metrics = volume.getMetrics();
                //获取磁盘的名字
				String volumeName = volume.getBaseURI().getPath();
                //元数据的指标数据
                metadataOpStats.put(volumeName,
                    metrics.getMetadataOperationMean());
                //实际数据read
				readIoStats.put(volumeName, metrics.getReadIoMean());
                //实际数据的write
				writeIoStats.put(volumeName, metrics.getWriteIoMean());
              }
            } finally {
              if (fsVolumeReferences != null) {
                try {
                  fsVolumeReferences.close();
                } catch (IOException e) {
                  LOG.error("Error in releasing FS Volume references", e);
                }
              }
            }
            if (metadataOpStats.isEmpty() && readIoStats.isEmpty()
                && writeIoStats.isEmpty()) {
              LOG.debug("No disk stats available for detecting outliers.");
              continue;
            }
            //更新离群值
            detectAndUpdateDiskOutliers(metadataOpStats, readIoStats,
                writeIoStats);
          }

          try {
            Thread.sleep(detectionInterval);
          } catch (InterruptedException e) {
            LOG.error("Disk Outlier Detection thread interrupted", e);
            Thread.currentThread().interrupt();
          }
        }
      }
    });
    slowDiskDetectionDaemon.start();
  }  
```

  大致工作流程如下:
  
<div>
<img src="/images/posts/hadoop-source-18/hadoop02.png" height="380" width="580" />
</div>

### OutlierDetector源代码

  使用中位数的绝对差比较慢盘
  
```
    public Map<String, Double> getOutliers(Map<String, Double> stats) {
    if (stats.size() < minNumResources) {
      return ImmutableMap.of();
    }
    // 计算平均绝对差
    final List<Double> sorted = new ArrayList<>(stats.values());
    Collections.sort(sorted);
    //获取中位数
    final Double median = computeMedian(sorted);
    //计算平均绝对差
    final Double mad = computeMad(sorted);
    Double upperLimitLatency = Math.max(
        lowThresholdMs, median * MEDIAN_MULTIPLIER);
    upperLimitLatency = Math.max(
        upperLimitLatency, median + (DEVIATION_MULTIPLIER * mad));
    final Map<String, Double> slowResources = new HashMap<>();
    // 更新慢磁盘
    for (Map.Entry<String, Double> entry : stats.entrySet()) {
      if (entry.getValue() > upperLimitLatency) {
        slowResources.put(entry.getKey(), entry.getValue());
      }
    }
    return slowResources;
  }
```

### 总结

   以上为笔者对`DataNodeDiskMetrics`的概述,希望本文对读者起到一定的帮助.

### 参考

* https://issues.apache.org/jira/browse/HDFS-11461
* https://issues.apache.org/jira/browse/HDFS-14235