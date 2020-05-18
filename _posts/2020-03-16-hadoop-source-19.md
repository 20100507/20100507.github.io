---
layout: post
title: "Hadoop源码分析-19-OffineEditsViewer"
date: 2020-03-16
tag: Hadoop源码分析
---

### 前言

     学习过HDFS都听说过Edits文件,Edits文件是元数据合并为Image之前的文件.该文件中涵盖了大量的操作码.以及事务ID等.元数据是集群的
 非常非常重要的一部分. 假如集群出现了元数据Edits文件损坏.需要修复Edits文件或者要查看Edits文件内容,可以使用HDFS提供`hdfs oev`查看
 和修复.本文谈谈edit文件的解析和反解析. 
  
### 基本操作

  1. 把二进制文件转换为人可读的XML文件,-p采用xml处理器-i输入的二进制文件-o输出的文件
  
   `hdfs oev -p xml -i edits -o edits.xml` 
   
  2. 把xml文件转换为二进制的edits文件,-p采用binary处理器-i输入xml文件-o输出的二进制文件
  
   `hdfs oev -p binary -i edits.xml -o edits`
   
  3. 统计edits文件中操作码的操作次数
   
   `hdfs oev -p stats -i edits -o edits.stats`
  
### 类结构

   粗略结构如下图
     
   <div>
   <img src="/images/posts/hadoop-source-19/hadoop01.png" height="580" width="680" />
   </div>

   <div>
   <img src="/images/posts/hadoop-source-19/hadoop02.png" height="580" width="680" />
   </div>

### 大体执行流程

   执行流程如下图
   
   <div>
   <img src="/images/posts/hadoop-source-19/hadoop03.png" height="580" width="680" />
   </div>


> 1. shell脚本调用OfflineEditsViewer类main方法
> 2. main方法调用go方法,同时传入相关的参数
> 3. go方法中调用visitor工厂类,创建三种process
> 4. 调用loader工厂根据不同的process创建不同的edit加载器 加载相应的文件

### 源代码跟踪

   以二进制文件转换XML文件,执行`hdfs oev -p binary -i edits.xml -o edits`为代表.跟踪源代码如下

   1. 首先打开`OfflineEditsViewer`找到main方法下的go方法
   
```
  public int go(String inputFileName, String outputFileName, String processor,
    Flags flags, OfflineEditsVisitor visitor)
    {
      try {
        if (visitor == null) {
          // 创建visitor
          // outputFileName             输出文件名称
          // processor                  使用什么处理器 new xmlvisitor或binaryvisitor
          // flags.getPrintToScreen()   是否打印到屏幕
          visitor = OfflineEditsVisitorFactory.getEditsVisitor(
              outputFileName, processor, flags.getPrintToScreen());
        }
         //根据visitor创建Loader
         //visitor 类型
        // inputFileName输入的文件名称
        // xmlInput 输入文件类型
        // flags 1. 是否打印屏幕 2. 是否修复Txid 3.是否启用恢复模式
        OfflineEditsLoader loader = OfflineEditsLoaderFactory.
            createLoader(visitor, inputFileName, xmlInput, flags);
	    //加载edit
        loader.loadEdits();
      } catch(Exception e) {
        System.err.println("Encountered exception. Exiting: " + e.getMessage());
        e.printStackTrace(System.err);
        return -1;
      }
      return 0;
    }
```  
  
  2. 打开`OfflineEditsVisitorFactory`中的OfflineEditsVisitor方法
  
```
static public OfflineEditsVisitor getEditsVisitor(String filename,
  String processor, boolean printToScreen) throws IOException {
  //创建二进制处理器
  if(StringUtils.equalsIgnoreCase("binary", processor)) {
    return new BinaryEditsVisitor(filename);
  }
  OfflineEditsVisitor vis;
  //创建文件输出流
  OutputStream fout = Files.newOutputStream(Paths.get(filename));
  OutputStream out = null;
  try {
    if (!printToScreen) {
      out = fout;
    }
    else {
      //创建两个输出位置 1. 屏幕 2. 文件
      OutputStream outs[] = new OutputStream[2];
      outs[0] = fout;
      outs[1] = System.out;
      out = new TeeOutputStream(outs);
    }
    //创建xml处理器
    if(StringUtils.equalsIgnoreCase("xml", processor)) {
      vis = new XmlEditsVisitor(out);
    //创建统计处理器  
    } else if(StringUtils.equalsIgnoreCase("stats", processor)) {
      vis = new StatisticsEditsVisitor(out);
    } else {
      throw new IOException("Unknown processor " + processor +
        " (valid processors: xml, binary, stats)");
    }
    out = fout = null;
    return vis;
  } finally {
    IOUtils.closeStream(fout);
    IOUtils.closeStream(out);
  }
}
```

  3. 进入`OfflineEditsLoader`,其中的内部静态类为静态工厂
  
```
    static class OfflineEditsLoaderFactory {
    static OfflineEditsLoader createLoader(OfflineEditsVisitor visitor,
        String inputFileName, boolean xmlInput,
        OfflineEditsViewer.Flags flags) throws IOException {
      if (xmlInput) {
        return new OfflineEditsXmlLoader(visitor, new File(inputFileName), flags);
      } else {
        File file = null;
        EditLogInputStream elis = null;
        OfflineEditsLoader loader = null;
        try {
          file = new File(inputFileName);
          elis = new EditLogFileInputStream(file, HdfsServerConstants.INVALID_TXID,
              HdfsServerConstants.INVALID_TXID, false);
          loader = new OfflineEditsBinaryLoader(visitor, elis, flags);
        } finally {
          if ((loader == null) && (elis != null)) {
            elis.close();
          }
        }
        return loader;
      }
    }
  }

```  

  4. 进入`OfflineEditsBinaryLoader `类,加载二进制edits文件
  
```
  public void loadEdits() throws IOException {
  try {
    visitor.start(inputStream.getVersion(true));
    while (true) {
      try {
        //读取操作码
        FSEditLogOp op = inputStream.readOp();
        //操作码为空 直接退出循环
        if (op == null)
          break;
        //启动修复TxId 
        if (fixTxIds) {
          //如果TxId小于0 获取txId
          if (nextTxId <= 0) {
            nextTxId = op.getTransactionId();
            //如果依然小于0 重置txId为1
            if (nextTxId <= 0) {
              nextTxId = 1;
            }
          }
          op.setTransactionId(nextTxId);
          nextTxId++;
        }
        //重新写回edits 为xml文件
        visitor.visitOp(op);
      } catch (IOException e) {
        if (!recoveryMode) {
          // Tell the visitor to clean up, then re-throw the exception
          LOG.error("Got IOException at position " +
            inputStream.getPosition());
          visitor.close(e);
          throw e;
        }
    }
    visitor.close(null);
  } finally {
    IOUtils.cleanupWithLogger(LOG, inputStream);
  }
}
```

  5. 进入`XmlEditsVisitor`类,重新回写xml文件
  
```
   public void visitOp(FSEditLogOp op) throws IOException {
    try {
      op.outputToXml(contentHandler);
    }
    catch (SAXException e) {
      throw new IOException("SAX error: " + e.getMessage());
    }
  }
}
```  

### 参考

* https://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-hdfs/HdfsEditsViewer.html