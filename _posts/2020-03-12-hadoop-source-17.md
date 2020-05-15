---
layout: post
title: "Hadoop源码分析-16-DataNode-shutdown"
date: 2020-03-12
tag: Hadoop源码分析
---

### 前言

     DataNode作为一个庞大的类,这里不一一分析,如果去一一分析意义不是很大,本文浅析DataNode关闭方法,希望读者可以对HDFS DataNode
  该角色上使用的相关线程和其他的服务从概念上有一个了解.
  
### 关闭DataNode方法

  1. 通常暴力关闭DataNode
 
  `hdfs --daemon stop datanode`
  
  2. 滚动升级静默时关闭DataNode
  
  `hdfs dfsadmin -shutdownDatanode <DATANODE_HOST:IPC_PORT> [upgrade]`  
   

###  DataNode关闭流程

  DataNode关闭流程如下图:
  
<div>
<img src="/images/posts/hadoop-source-17/hadoop01.png" height="780" width="980" />
</div>
  
  DataNode关闭流程梳理
  
> 1.  关闭指标日志采集器
> 2.  关闭用户自定义DataNode插件
> 3.  关闭DataXceiver
> 4.  关闭本地DataXciverServer
> 5.  关闭磁盘目录扫描
> 6.  关闭磁盘均衡
> 7.  关闭web-ui访问
> 8.  关闭磁盘检查器
> 9.  关闭JVM停顿检测器
> 10. 关闭配置文件重新配置任务
> 11. 关闭传输线程
> 12. 关闭Receiver线程组
> 13. 设置指标采集器活跃的Xceiver为0
> 14. 关闭/停止IPC
> 15. 关闭纠删码工作器
> 16. 关闭BP池管理
> 17. 释放文件锁
> 18. 关闭磁盘引用器
> 18. 关闭JMX指标采集器
> 19. 关闭JMX磁盘指标采集器
> 20. 注销JMX
> 21. 关闭短路读
> 22. 关闭跟踪器

### 源代码

  1. 关闭DataNode,判断是否是滚动更新关闭DataNode

 ```
  public synchronized void shutdownDatanode(boolean forUpgrade) throws IOException {
    checkSuperuserPrivilege();
    LOG.info("shutdownDatanode command received (upgrade={}). " +
        "Shutting down Datanode...", forUpgrade);
    //一次性语义,只能关闭一次防止多次关闭
    if (shutdownInProgress) {
      throw new IOException("Shutdown already in progress.");
    }
    shutdownInProgress = true;
	//判断是否是滚动更新集群
    shutdownForUpgrade = forUpgrade;

    //异步关闭
    Thread shutdownThread = new Thread("Async datanode shutdown thread") {
      @Override public void run() {
	    //如果不是滚动更新关闭 则等待1秒关闭
        if (!shutdownForUpgrade) {
          try {
            Thread.sleep(1000);
          } catch (InterruptedException ie) { }
        }
		//关闭
        shutdown();
      }
    };

    shutdownThread.setDaemon(true);
	//启动线程开始关闭
    shutdownThread.start();
  }
 ```
 
 执行关闭DataNode方法
 
```
    public void shutdown() {
	//关注指标数据日志采集器
    stopMetricsLogger();
    if (plugins != null) {
      for (ServicePlugin p : plugins) {
        try {
		  //关闭用户自定义插件
          p.stop();
          LOG.info("Stopped plug-in {}", p);
        } catch (Throwable t) {
          LOG.warn("ServicePlugin {} could not be stopped", p, t);
        }
      }
    }

    //获取所有的BP池线程
    List<BPOfferService> bposArray = (this.blockPoolManager == null)
        ? new ArrayList<BPOfferService>()
        : this.blockPoolManager.getAllNamenodeThreads();
    // If shutdown is not for restart, set shouldRun to false early. 
    if (!shutdownForUpgrade) {
	  //如果不是关闭等待更新
      shouldRun = false;
    }
    //关闭dataXceiverServer
    if (dataXceiverServer != null) {
      try {
        xserver.sendOOBToPeers();
        ((DataXceiverServer) this.dataXceiverServer.getRunnable()).kill();
        this.dataXceiverServer.interrupt();
      } catch (Exception e) {
        // Ignore, since the out of band messaging is advisory.
        LOG.trace("Exception interrupting DataXceiverServer", e);
      }
    }

    // Record the time of initial notification
    long timeNotified = Time.monotonicNow();
    // 关闭本地DataXciverServer
    if (localDataXceiverServer != null) {
      ((DataXceiverServer) this.localDataXceiverServer.getRunnable()).kill();
      this.localDataXceiverServer.interrupt();
    }
    //关闭磁盘扫描和目录扫描
    // Terminate directory scanner and block scanner
    shutdownPeriodicScanners();
	//关闭磁盘平衡
    shutdownDiskBalancer();
    //关闭Web-UI
    // Stop the web server
    if (httpServer != null) {
      try {
        httpServer.close();
      } catch (Exception e) {
        LOG.warn("Exception shutting down DataNode HttpServer", e);
      }
    }
    //关闭磁盘检查
    volumeChecker.shutdownAndWait(1, TimeUnit.SECONDS);
    //本地存储检查
    if (storageLocationChecker != null) {
      storageLocationChecker.shutdownAndWait(1, TimeUnit.SECONDS);
    }
    //JVM停顿检查器
    if (pauseMonitor != null) {
      pauseMonitor.stop();
    }

    // shouldRun is set to false here to prevent certain threads from exiting
    // before the restart prep is done.
    this.shouldRun = false;
    
    // wait reconfiguration thread, if any, to exit
    //关闭正在运行的配置重新修改任务
	shutdownReconfigurationTask();

    LOG.info("Waiting up to 30 seconds for transfer threads to complete");
    //关闭所有的传输线程
	HadoopExecutors.shutdown(this.xferService, LOG, 15L, TimeUnit.SECONDS);

    // wait for all data receiver threads to exit
    if (this.threadGroup != null) {
      int sleepMs = 2;
      while (true) {
	     //关闭所有的数据接收线程
        // When shutting down for restart, wait 2.5 seconds before forcing
        // termination of receiver threads.
        if (!this.shutdownForUpgrade ||
            (this.shutdownForUpgrade && (Time.monotonicNow() - timeNotified
                > 1000))) {
          this.threadGroup.interrupt();
          break;
        }
        LOG.info("Waiting for threadgroup to exit, active threads is {}",
                 this.threadGroup.activeCount());
        if (this.threadGroup.activeCount() == 0) {
          break;
        }
        try {
          Thread.sleep(sleepMs);
        } catch (InterruptedException e) {}
        sleepMs = sleepMs * 3 / 2; // exponential backoff
        if (sleepMs > 200) {
          sleepMs = 200;
        }
      }
      this.threadGroup = null;
    }
    if (this.dataXceiverServer != null) {
      // wait for dataXceiverServer to terminate
      try {
        this.dataXceiverServer.join();
      } catch (InterruptedException ie) {
      }
    }
    if (this.localDataXceiverServer != null) {
      // wait for localDataXceiverServer to terminate
      try {
        this.localDataXceiverServer.join();
      } catch (InterruptedException ie) {
      }
    }
    if (metrics != null) {
	  //设置指标数据中Xceiver活跃数为0
      metrics.setDataNodeActiveXceiversCount(0);
    }

   // IPC server needs to be shutdown late in the process, otherwise
   // shutdown command response won't get sent.
    //关闭IPC
   if (ipcServer != null) {
      ipcServer.stop();
    }
   //关闭纠删码
    if (ecWorker != null) {
      ecWorker.shutDown();
    }

    if(blockPoolManager != null) {
      try {
	    //关闭块管理器
        this.blockPoolManager.shutDownAll(bposArray);
      } catch (InterruptedException ie) {
        LOG.warn("Received exception in BlockPoolManager#shutDownAll", ie);
      }
    }
    
    if (storage != null) {
      try {
	   //释放所有的文件锁
        this.storage.unlockAll();
      } catch (IOException ie) {
        LOG.warn("Exception when unlocking storage", ie);
      }
    }
    if (data != null) {
      data.shutdown();
    }
    if (metrics != null) {
	  //关闭指标采集器
      metrics.shutdown();
    }
    if (diskMetrics != null) {
	  //关闭磁盘指标采集器
      diskMetrics.shutdownAndWait();
    }
    if (dataNodeInfoBeanName != null) {
	 //注销MBean
      MBeans.unregister(dataNodeInfoBeanName);
      dataNodeInfoBeanName = null;
    }
    //关闭短路读
    if (shortCircuitRegistry != null) shortCircuitRegistry.shutdown();
    LOG.info("Shutdown complete.");
    synchronized(this) {
      // it is already false, but setting it again to avoid a findbug warning.
      this.shouldRun = false;
      // Notify the main thread.
      notifyAll();
    }
	//关闭跟踪器
    tracer.close();
  }

``` 
 
### 总结

   以上为笔者对DataNode关闭的概述,希望本文对读者起到一定的帮助.

### 参考

* https://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-hdfs/HdfsRollingUpgrade.html
* https://hadoop.apache.org/