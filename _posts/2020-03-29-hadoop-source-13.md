---
layout: post
title: "Hadoop源码分析-13-DataXceiver"
date: 2020-03-02
tag: Hadoop源码分析
---

### 前言

    在运维集群中我们会经常看到有DataXcevier Peer is not connect 这样的错误.其实在熟知的DataNode背后,有一个核心的线程为DataXcevier,这个线程
 处理数据的流入和流出.直接关系到整个HDFS数据的流转.本文浅析一下DataXceiver这个线程类及其周边的类关系.

### 核心类概述

> 1. DataXceiver
> 2. Receiver
> 3. Sender
> 4. DataXceiverServer

  概述:`DataXceiver`线程用来处理输入和输出数据流.`Receiver`是接收者,`Sender`为发送者.`DataXceiverServer`为监听器,监听客户端和其他的
DataNode的请求,相当于`DataXceiver`的发动机.生产`DataXceiver`,真正干活的是`DataXceiver`.

### DataXceiver
  
  该类是一个线程类,首先我们查看他的`run()`方法.
  
```
  public void run() {
    int opsProcessed = 0;
    Op op = null;

    try {
      synchronized(this) {
        xceiver = Thread.currentThread();
      }
      //添加连接端点
      dataXceiverServer.addPeer(peer, Thread.currentThread(), this);
      ....
      do {
	    //更新线程的名字
        updateCurrentThreadName("Waiting for operation #" + (opsProcessed + 1));
        try {
          if (opsProcessed != 0) {
            assert dnConf.socketKeepaliveTimeout > 0;
            peer.setReadTimeout(dnConf.socketKeepaliveTimeout);
          } else {
            peer.setReadTimeout(dnConf.socketTimeout);
          }
		  //读取操作码
          op = readOp();
        } catch (InterruptedIOException ignored) {
          break;
        } catch (EOFException | ClosedChannelException e) {
          break;
        } catch (IOException err) {
		//增加网络错误
          incrDatanodeNetworkErrors();
          throw err;
        }
        if (opsProcessed != 0) {
          peer.setReadTimeout(dnConf.socketTimeout);
        }
        opStartTime = monotonicNow();
        //开始移交给Receiver处理
        processOp(op);
        ++opsProcessed;
      } while ((peer != null) &&
          (!peer.isClosed() && dnConf.socketKeepaliveTimeout > 0));
      ....
  }
```
  
### Receiver
    
   Receiver 处理操作码如下
   
> * WRITE_BLOCK 写块
> * READ_BLOCK   读块
> * READ_METADATA 读取块元数据
> * REPLACE_BLOCK 替换块
> * COPY_BLOCK     拷贝块
> * BLOCK_CHECKSUM  块校验
> * TRANSFER_BLOCK  传输块
> * REQUEST_SHORT_CIRCUIT_FDS 短路请求文件描述符
> * RELEASE_SHORT_CIRCUIT_FDS 短路释放文件描述符
> * REQUEST_SHORT_CIRCUIT_SHM  短路请求共享内存域
> * BLOCK_GROUP_CHECKSUM       纠删码校验

  
   通过Switch判断处理
   
```
 protected final void processOp(Op op) throws IOException {
    switch(op) {
    case READ_BLOCK:
      opReadBlock();
      break;
    case WRITE_BLOCK:
      opWriteBlock(in);
      break;
    case REPLACE_BLOCK:
      opReplaceBlock(in);
      break;
    case COPY_BLOCK:
      opCopyBlock(in);
      break;
    case BLOCK_CHECKSUM:
      opBlockChecksum(in);
      break;
    case BLOCK_GROUP_CHECKSUM:
      opStripedBlockChecksum(in);
      break;
    case TRANSFER_BLOCK:
      opTransferBlock(in);
      break;
    case REQUEST_SHORT_CIRCUIT_FDS:
      opRequestShortCircuitFds(in);
      break;
    case RELEASE_SHORT_CIRCUIT_FDS:
      opReleaseShortCircuitFds(in);
      break;
    case REQUEST_SHORT_CIRCUIT_SHM:
      opRequestShortCircuitShm(in);
      break;
    default:
      throw new IOException("Unknown op " + op + " in data stream");
    }
  }
```   

  目前大致流程可以梳理如下

<div>
<img src="/images/posts/hadoop-source-13/hadoop01.png" height="880" width="1180" />
</div>   
  
### Sender

  既然有`Receiver`,必然相应的有`Sender`,即有发送方法,发送操作码给`Receiver`

```
private static void send(final DataOutputStream out, final Op opcode,
    final Message proto) throws IOException {
  LOG.trace("Sending DataTransferOp {}: {}",
      proto.getClass().getSimpleName(), proto);
  op(out, opcode);
  proto.writeDelimitedTo(out);
  out.flush();
}

```

  目前大致流程可以梳理如下

<div>
<img src="/images/posts/hadoop-source-13/hadoop02.png" height="880" width="1180" />
</div>   

  

### DataXceiverServer
 
 `DataXceiverServer`作为监听者,也可以称作发动机,监听来着客户端和其他的DataNode请求.也是一个线程类,我们可以查看他的run方法
 
```
@Override
public void run() {
  Peer peer = null;
  while (datanode.shouldRun && !datanode.shutdownForUpgrade) {
    try {
      peer = peerServer.accept();

      // Make sure the xceiver count is not exceeded
      int curXceiverCount = datanode.getXceiverCount();
      if (curXceiverCount > maxXceiverCount) {
        throw new IOException("Xceiver count " + curXceiverCount
            + " exceeds the limit of concurrent xceivers: "
            + maxXceiverCount);
      }
      //new DataXceiver 类然后运行该线程
      new Daemon(datanode.threadGroup,
          DataXceiver.create(peer, datanode, this))
          .start();
    } catch (SocketTimeoutException ignored) {
      .... 
    } catch (Throwable te) {
      LOG.error("{}:DataXceiverServer: Exiting.", datanode.getDisplayName(),
          te);
      datanode.shouldRun = false;
    }
  }
}
```

  最后的流程梳理如下:

<div>
<img src="/images/posts/hadoop-source-13/hadoop03.png" height="880" width="1180" />
</div>

 其中涉及到了添加peer到`HashMap<Peer, DataXceiver> peersXceiver = new HashMap<>();`

### 总结

   以上为笔者对`DataXceiver`的总结,没有具体谈到如何监听,感兴趣的读者可以继续深入理解,希望本文对读者起到一定的帮助.

### 参考

* http://hadoop.apache.org/