---
layout: post
title: "Hadoop源码分析-23-Fsck"
date: 2020-04-08
tag: Hadoop源码分析
---

### 前言

     在集群出现数据块丢失,部分集群运维人员会盲目执行`hdfs fsck`命令去删除或者移动数据,如果对底层实现原理不是
    很了解，盲目执行该命令是非常非常危险的。本文笔者聊聊HDFS的查看状态和修复命令.加深一下读者和笔者对该命令的底层理解.

### 基本操作

  * 执行命令检查根目录的文件健康状态

   `hdfs fsck /`

```shell
Connecting to namenode via http://hadoop-2-02:50070/fsck?ugi=hadoop&path=%2F
FSCK started by hadoop (auth:SIMPLE) from /192.168.149.131 for path / at Mon May 18 22:04:31 EDT 2020

/orgs/bianqi2/edits_inprogress_0000000000000000725:  Under replicated BP-2048483368-192.168.149.131-1588850814296:blk_1073741828_1004. Target Replicas is 3 but found 1 live replica(s), 0 decommissioned replica(s), 0 decommissioning replica(s).

/orgs/bianqi4/edits_0000000000000001244-0000000000000001254:  Under replicated BP-2048483368-192.168.149.131-1588850814296:blk_1073741827_1003. Target Replicas is 3 but found 1 live replica(s), 0 decommissioned replica(s), 0 decommissioning replica(s).

/orgs/bianqi4/edits_inprogress_0000000000000001255:  Under replicated BP-2048483368-192.168.149.131-1588850814296:blk_1073741826_1002. Target Replicas is 3 but found 1 live replica(s), 0 decommissioned replica(s), 0 decommissioning replica(s).

/orgs/dingran/edits.stats:  Under replicated BP-2048483368-192.168.149.131-1588850814296:blk_1073741825_1001. Target Replicas is 3 but found 1 live replica(s), 0 decommissioned replica(s), 0 decommissioning replica(s).

/orgs/wangda2/edits_inprogress_0000000000000000725:  Under replicated BP-2048483368-192.168.149.131-1588850814296:blk_1073741829_1005. Target Replicas is 3 but found 1 live replica(s), 0 decommissioned replica(s), 0 decommissioning replica(s).

Status: HEALTHY
 Number of data-nodes:	1
 Number of racks:		1
 Total dirs:			27
 Total symlinks:		0

Replicated Blocks:
 Total size:	3153822 B
 Total files:	7
 Total blocks (validated):	7 (avg. block size 450546 B)
 Minimally replicated blocks:	7 (100.0 %)
 Over-replicated blocks:	0 (0.0 %)
 Under-replicated blocks:	5 (71.42857 %)
 Mis-replicated blocks:		0 (0.0 %)
 Default replication factor:	3
 Average block replication:	1.0
 Missing blocks:		0
 Corrupt blocks:		0
 Missing replicas:		10 (58.82353 %)

Erasure Coded Block Groups:
 Total size:	0 B
 Total files:	0
 Total block groups (validated):	0
 Minimally erasure-coded block groups:	0
 Over-erasure-coded block groups:	0
 Under-erasure-coded block groups:	0
 Unsatisfactory placement block groups:	0
 Average block group size:	0.0
 Missing block groups:		0
 Corrupt block groups:		0
 Missing internal blocks:	0
FSCK ended at Mon May 18 22:04:31 EDT 2020 in 1 milliseconds

The filesystem under path '/' is HEALTHY
```



* 直接执行如下命令，得到帮助信息

 `hdfs fsck` 

​    帮助信息

```shell
Usage: hdfs fsck
	<path>	#要检查的目录
	-move	#移动损坏的文件到 /lost+found目录
	-delete	#删除损坏的文件
	-files	#打印要检查的文件
	-openforwrite	#打印正在写入的文件
	-includeSnapshots #是否包含该目录下的快照
	-list-corruptfileblocks	#打印出损坏的块
	-files -blocks	#打印出块的报告
	-files -blocks -locations	#打印每个块所在的位置
	-files -blocks -racks	#打印机架
	-files -blocks -replicaDetails	#打印副本具体信息
	-files -blocks -upgradedomains	#打印每一个块的升级域
	-storagepolicies	#打印存储的策略
	-maintenance	#打印进入维护状态的详细信息
	-showprogress	#输出正在处理程序正在处理
	-blockId	#打印该快的详细信息

```



### 执行基本流程

  执行命令流程

> 1. 当执行hdfs fsck命令时调用 hdfs shell脚本
> 2. shell 脚本执行DFSck类的main方法
> 3. DFSck类采用http GET方式调用FsckServlet类的doGet方法,
> 4. FsckServlet类调用NamenodeFsck类的fsck方法
> 5. NamenodeFsck类fsck执行一系列的逻辑方法

1. hdfs shell脚本内容,执行`org.apache.hadoop.hdfs.tools.DFSck` main方法

```shell
hadoop_add_subcommand "fsck" admin "run a DFS filesystem checking utility"
function hdfscmd_case
{
  subcmd=$1
  shift
    fsck)
      HADOOP_CLASSNAME=org.apache.hadoop.hdfs.tools.DFSck
    ... 
  esac
}

```

2. DFSck类 main方法如下

```java
  public static void main(String[] args) throws Exception {
      //执行DFSck的run方法
      res = ToolRunner.run(new DFSck(new HdfsConfiguration()), args);
  }
```

3. DFSck类 run方法如下

```java
  public int run(final String[] args) throws IOException {
    try {
      return UserGroupInformation.getCurrentUser().doAs(
          new PrivilegedExceptionAction<Integer>() {
            @Override
            public Integer run() throws Exception {
              //携带参数执行doWork方法  
              return doWork(args);
            }
          });
    } catch (InterruptedException e) {
      throw new IOException(e);
    }
  }
```

4. DFSck类 doWork(args)方法

    该方法内容比较多，只要是构造HTTP的GET请求的相关参数并且进行校验

```java
  private int doWork(final String[] args) throws IOException {
    final StringBuilder url = new StringBuilder();
    //访问WEBUI的URL+用户名
    url.append("/fsck?ugi=").append(ugi.getShortUserName());
    String dir = null;
    boolean doListCorruptFileBlocks = false;
    //获取命令行传递过来的参数 转换为GET请求的参数  
    for (int idx = 0; idx < args.length; idx++) {
      if (args[idx].equals("-move")) { url.append("&move=1"); }
      else if (args[idx].equals("-delete")) { url.append("&delete=1"); }
      else if (args[idx].equals("-files")) { url.append("&files=1"); }
      else if (args[idx].equals("-openforwrite")) { url.append("&openforwrite=1"); }
      else if (args[idx].equals("-blocks")) { url.append("&blocks=1"); }
      else if (args[idx].equals("-locations")) { url.append("&locations=1"); }
      else if (args[idx].equals("-racks")) { url.append("&racks=1"); }
      else if (args[idx].equals("-replicaDetails")) {
        url.append("&replicadetails=1");
      } else if (args[idx].equals("-upgradedomains")) {
        url.append("&upgradedomains=1");
      } else if (args[idx].equals("-storagepolicies")) {
        url.append("&storagepolicies=1");
      } else if (args[idx].equals("-showprogress")) {
        url.append("&showprogress=1");
      } else if (args[idx].equals("-list-corruptfileblocks")) {
        url.append("&listcorruptfileblocks=1");
        doListCorruptFileBlocks = true;
      } else if (args[idx].equals("-includeSnapshots")) {
        url.append("&includeSnapshots=1");
      } else if (args[idx].equals("-maintenance")) {
        url.append("&maintenance=1");
      } else if (args[idx].equals("-blockId")) {
        StringBuilder sb = new StringBuilder();
        idx++;
        while(idx < args.length && !args[idx].startsWith("-")){
          sb.append(args[idx]);
          sb.append(" ");
          idx++;
        }
        url.append("&blockId=").append(URLEncoder.encode(sb.toString(), "UTF-8"));
      } else if (args[idx].equals("-replicate")) {
        url.append("&replicate=1");
      } else if (!args[idx].startsWith("-")) {
        if (null == dir) {
          dir = args[idx];
        } else {
          System.err.println("fsck: can only operate on one path at a time '"
              + args[idx] + "'");
          printUsage(System.err);
          return -1;
        }

      } else {
        System.err.println("fsck: Illegal option '" + args[idx] + "'");
        printUsage(System.err);
        return -1;
      }
    }
    //如果没有传递目录则默认为根目录  
    if (null == dir) {
      dir = "/";
    }

    Path dirpath = null;
    URI namenodeAddress = null;
    try {
      dirpath = getResolvedPath(dir);
      //获取NN的http地址  
      namenodeAddress = getCurrentNamenodeAddress(dirpath);
    } catch (IOException ioe) {
      System.err.println("FileSystem is inaccessible due to:\n"
          + ioe.toString());
    }
    
    if (namenodeAddress == null) {
      //Error message already output in {@link #getCurrentNamenodeAddress()}
      System.err.println("DFSck exiting.");
      return 0;
    }

    url.insert(0, namenodeAddress.toString());
    url.append("&path=").append(URLEncoder.encode(
        Path.getPathWithoutSchemeAndAuthority(dirpath).toString(), "UTF-8"));
    System.err.println("Connecting to namenode via " + url.toString());

    if (doListCorruptFileBlocks) {
      return listCorruptFileBlocks(dir, url.toString());
    }
    URL path = new URL(url.toString());
    URLConnection connection;
    try {
      connection = connectionFactory.openConnection(path, isSpnegoEnabled);
    } catch (AuthenticationException e) {
      throw new IOException(e);
    }
    InputStream stream = connection.getInputStream();
    BufferedReader input = new BufferedReader(new InputStreamReader(
                                              stream, "UTF-8"));
    String line = null;
    String lastLine = NamenodeFsck.CORRUPT_STATUS;
    int errCode = -1;
    try {
      while ((line = input.readLine()) != null) {
        out.println(line);
        lastLine = line;
      }
    } finally {
      input.close();
    }
    //错误码  
    if (lastLine.endsWith(NamenodeFsck.HEALTHY_STATUS)) {
      errCode = 0;
    } else if (lastLine.endsWith(NamenodeFsck.CORRUPT_STATUS)) {
      errCode = 1;
    } else if (lastLine.endsWith(NamenodeFsck.NONEXISTENT_STATUS)) {
      errCode = 0;
    } else if (lastLine.contains("Incorrect blockId format:")) {
      errCode = 0;
    } else if (lastLine.endsWith(NamenodeFsck.DECOMMISSIONED_STATUS)) {
      errCode = 2;
    } else if (lastLine.endsWith(NamenodeFsck.DECOMMISSIONING_STATUS)) {
      errCode = 3;
    } else if (lastLine.endsWith(NamenodeFsck.IN_MAINTENANCE_STATUS))  {
      errCode = 4;
    } else if (lastLine.endsWith(NamenodeFsck.ENTERING_MAINTENANCE_STATUS)) {
      errCode = 5;
    }
    return errCode;
  }
```

5. 连接到`FsckServlet`类的`doGet`方法。

```java
public void doGet(HttpServletRequest request, HttpServletResponse response
      ) throws IOException {
    @SuppressWarnings("unchecked")
    //获取url参数
    final Map<String,String[]> pmap = request.getParameterMap();
    final PrintWriter out = response.getWriter();
    //获取远程请求的ip地址
    final InetAddress remoteAddress =
      InetAddress.getByName(request.getRemoteAddr());
    //获取servlet上下文
    final ServletContext context = getServletContext();
    //获取配置
    final Configuration conf = NameNodeHttpServer.getConfFromContext(context);
    //获取用户组信息
    final UserGroupInformation ugi = getUGI(request, conf);
    try {
      //运行
      ugi.doAs(new PrivilegedExceptionAction<Object>() {
        @Override
        public Object run() throws Exception {
          //获取NN
          NameNode nn = NameNodeHttpServer.getNameNodeFromContext(context);
          //获取Namesystem
          final FSNamesystem namesystem = nn.getNamesystem();
          //获取BM
          final BlockManager bm = namesystem.getBlockManager();
          final int totalDatanodes = 
              namesystem.getNumberOfDatanodes(DatanodeReportType.LIVE); 
            //创建fsck
          new NamenodeFsck(conf, nn,
              bm.getDatanodeManager().getNetworkTopology(), pmap, out,
              //执行fsck命令
              totalDatanodes, remoteAddress).fsck();
          return null;
        }
      });
    } catch (InterruptedException e) {
      response.sendError(400, e.getMessage());
    }
  }
```

   这个Servlet是什么时候启动的呢？进入`NameNodeHttpServer`类 `setupServlets`方法，而`NameNodeHttpServer`这个类在NN启动的时候即会立即启动.

```java
  private static void setupServlets(HttpServer2 httpServer, Configuration conf) {
    httpServer.addInternalServlet("startupProgress",
        StartupProgressServlet.PATH_SPEC, StartupProgressServlet.class);
    //启动内部的fsck 端口是webui的端口
    httpServer.addInternalServlet("fsck", "/fsck", FsckServlet.class,
        true);
    httpServer.addInternalServlet("imagetransfer", ImageServlet.PATH_SPEC,
        ImageServlet.class, true);
    httpServer.addInternalServlet(IsNameNodeActiveServlet.SERVLET_NAME,
        IsNameNodeActiveServlet.PATH_SPEC,
        IsNameNodeActiveServlet.class);
  }
```

6. 进入核心的`NamenodeFsck`类，该类维护了FSck所有的核心功能.该类的相关功能下一小节展开梳理

```java
  public void fsck() {
     ...
  }
```

​    小结：

​      综上我们可以总结出来大体粗略的执行流程图

<div>
<img src="/images/posts/hadoop-source-23/hadoop01.png" height="180" width="780" />
</div>

   这里我们也可以了解到其实执行该命令我们完全可以绕过命令行，直接在WEB-UI执行该命令即可，例如直接在

浏览器中输入如下命令`http://192.168.149.132:50070/fsck?ugi=hadoop`,浏览器中返回如下：

<div>
<img src="/images/posts/hadoop-source-23/hadoop02.png" height="480" width="780" />
</div>
### 核心执行流程

1. 首先我们进入`NamenodeFsck`类的fsck方法

```java
  public void fsck() {
    final long startTime = Time.monotonicNow();
    try {
      ...   
     //对照上图中输出的 FSCK started by hadoop (auth:SIMPLE) from /192.168.149.1 for              //path / at Mon May 18 21:23:30 EDT 2020  
      String msg = "FSCK started by " + UserGroupInformation.getCurrentUser()
          + " from " + remoteAddress + " for path " + path + " at " + new Date();
      LOG.info(msg);
      out.println(msg);
      namenode.getNamesystem().logFsckEvent(path, remoteAddress);
      if (snapshottableDirs != null) {
        SnapshottableDirectoryStatus[] snapshotDirs =
            namenode.getRpcServer().getSnapshottableDirListing();
        if (snapshotDirs != null) {
          for (SnapshottableDirectoryStatus dir : snapshotDirs) {
            snapshottableDirs.add(dir.getFullPath().toString());
          }
        }
      }
      final HdfsFileStatus file = namenode.getRpcServer().getFileInfo(path);
      if (file != null) {
        if (showCorruptFileBlocks) {
          //列出损坏的块
          listCorruptFileBlocks();
          return;
        }
        if (this.showStoragePolcies) {
          storageTypeSummary = new StoragePolicySummary(
              namenode.getNamesystem().getBlockManager().getStoragePolicies());
        }
        Result replRes = new ReplicationResult(conf);
        Result ecRes = new ErasureCodingResult(conf);
        //根据路径 文件 以及副本以及纠删码进行检查
        check(path, file, replRes, ecRes);
        out.print("\nStatus: ");
        out.println(replRes.isHealthy() && ecRes.isHealthy() ? "HEALTHY" : "CORRUPT");
        out.println(" Number of data-nodes:\t" + totalDatanodes);
        out.println(" Number of racks:\t\t" + networktopology.getNumOfRacks());
        out.println(" Total dirs:\t\t\t" + totalDirs);
        out.println(" Total symlinks:\t\t" + totalSymlinks);
        out.println("\nReplicated Blocks:");
        out.println(replRes);
        out.println("\nErasure Coded Block Groups:");
        out.println(ecRes);

        if (this.showStoragePolcies) {
          out.print(storageTypeSummary);
        }

        out.println("FSCK ended at " + new Date() + " in "
            + (Time.monotonicNow() - startTime + " milliseconds"));
        if (internalError) {
          throw new IOException("fsck encountered internal errors!");
        }
        if (replRes.isHealthy() && ecRes.isHealthy()) {
          out.print("\n\nThe filesystem under path '" + path + "' " + HEALTHY_STATUS);
        } else {
          out.print("\n\nThe filesystem under path '" + path + "' " + CORRUPT_STATUS);
        }

      } else {
        out.print("\n\nPath '" + path + "' " + NONEXISTENT_STATUS);
      }
    } catch (Exception e) {
      String errMsg = "Fsck on path '" + path + "' " + FAILURE_STATUS;
      LOG.warn(errMsg, e);
      out.println("FSCK ended at " + new Date() + " in "
          + (Time.monotonicNow() - startTime + " milliseconds"));
      out.println(e.getMessage());
      out.print("\n\n" + errMsg);
    } finally {
      out.close();
    }
  }
```

2. 如果showCorruptFileBlocks参数携带执行如下方法

```java
  private void listCorruptFileBlocks() throws IOException {
    //获取 携带快照的损坏的Block文件
    final List<String> corrputBlocksFiles = namenode.getNamesystem()
        .listCorruptFileBlocksWithSnapshot(path, snapshottableDirs,
            currentCookie);
    int numCorruptFiles = corrputBlocksFiles.size();
    String filler;
    if (numCorruptFiles > 0) {
      filler = Integer.toString(numCorruptFiles);
    } else if (currentCookie[0].equals("0")) {
      filler = "no";
    } else {
      filler = "no more";
    }
    out.println("Cookie:\t" + currentCookie[0]);
    for (String s : corrputBlocksFiles) {
      out.println(s);
    }
    out.println("\n\nThe filesystem under path '" + path + "' has " + filler
        + " CORRUPT files");
    out.println();
  }
```

3. 进入`check`方法

```java
  void check(String parent, HdfsFileStatus file, Result replRes, Result ecRes)
      throws IOException {
    String path = file.getFullName(parent);
    //格式更加美观
    if ((totalDirs + totalSymlinks + replRes.totalFiles + ecRes.totalFiles)
            % 1000 == 0) {
      out.println();
      out.flush();
    }
    //如果是目录 检查目录
    if (file.isDirectory()) {
      checkDir(path, replRes, ecRes);
      return;
    }
    //如果文件目录是软连接
    if (file.isSymlink()) {
      if (showFiles) {
        out.println(path + " <symlink>");
      }
      totalSymlinks++;
      return;
    }
    LocatedBlocks blocks = getBlockLocations(path, file);
    if (blocks == null) { // the file is deleted
      return;
    }
    //两种结果  EC 或者 REP
    final Result r = file.getErasureCodingPolicy() != null ? ecRes: replRes;
    //文件处理
    collectFileSummary(path, file, r, blocks);
    //块处理
    collectBlocksSummary(parent, file, r, blocks);
  }
```

4. 文件处理

```java
  private void collectFileSummary(String path, HdfsFileStatus file, Result res,
      LocatedBlocks blocks) throws IOException {
    long fileLen = file.getLen();
    boolean isOpen = blocks.isUnderConstruction();
    if (isOpen && !showOpenFiles) {
      res.totalOpenFilesSize += fileLen;
      res.totalOpenFilesBlocks += blocks.locatedBlockCount();
      res.totalOpenFiles++;
      return;
    }
    res.totalFiles++;
    res.totalSize += fileLen;
    res.totalBlocks += blocks.locatedBlockCount();
    String redundancyPolicy;
    ErasureCodingPolicy ecPolicy = file.getErasureCodingPolicy();
    if (ecPolicy == null) { // a replicated file
      redundancyPolicy = "replicated: replication=" +
          file.getReplication() + ",";
    } else {
      redundancyPolicy = "erasure-coded: policy=" + ecPolicy.getName() + ",";
    }

    if (showOpenFiles && isOpen) {
      out.print(path + " " + fileLen + " bytes, " + redundancyPolicy + " " +
        blocks.locatedBlockCount() + " block(s), OPENFORWRITE: ");
    } else if (showFiles) {
      out.print(path + " " + fileLen + " bytes, " + redundancyPolicy + " " +
        blocks.locatedBlockCount() + " block(s): ");
    } else if (res.totalFiles % 100 == 0) {
      out.print('.');
    }
  }
```

5. 数据块处理

```java
  private void collectBlocksSummary(String parent, HdfsFileStatus file,
      Result res, LocatedBlocks blocks) throws IOException {
    String path = file.getFullName(parent);
    boolean isOpen = blocks.isUnderConstruction();
    if (isOpen && !showOpenFiles) {
      return;
    }
    //丢失块个数  
    int missing = 0;
    //损坏块个数  
    int corrupt = 0;  
    long missize = 0;
    long corruptSize = 0;
    int underReplicatedPerFile = 0;
    int misReplicatedPerFile = 0;
    StringBuilder report = new StringBuilder();
    int blockNumber = 0;
    final LocatedBlock lastBlock = blocks.getLastLocatedBlock();
    List<BlockInfo> misReplicatedBlocks = new LinkedList<>();
    for (LocatedBlock lBlk : blocks.getLocatedBlocks()) {
      ExtendedBlock block = lBlk.getBlock();
      //排除UC块  
      if (!blocks.isLastBlockComplete() && lastBlock != null &&
          lastBlock.getBlock().equals(block)) {
        continue;
      }

      final BlockInfo storedBlock = blockManager.getStoredBlock(
          block.getLocalBlock());
      final int minReplication = blockManager.getMinStorageNum(storedBlock);
      // 统计 正在下线的已经下线的 正在进入维护模式的已经进入维护模式的 副本数
      NumberReplicas numberReplicas = blockManager.countNodes(storedBlock);
      int decommissionedReplicas = numberReplicas.decommissioned();
      int decommissioningReplicas = numberReplicas.decommissioning();
      int enteringMaintenanceReplicas =
          numberReplicas.liveEnteringMaintenanceReplicas();
      int inMaintenanceReplicas =
          numberReplicas.maintenanceNotForReadReplicas();
      res.decommissionedReplicas +=  decommissionedReplicas;
      res.decommissioningReplicas += decommissioningReplicas;
      res.enteringMaintenanceReplicas += enteringMaintenanceReplicas;
      res.inMaintenanceReplicas += inMaintenanceReplicas;

      // count total replicas
      int liveReplicas = numberReplicas.liveReplicas();
      //存活的副本加上正在下线+已经下线+进入维护+已经处于维护  
      int totalReplicasPerBlock = liveReplicas + decommissionedReplicas
          + decommissioningReplicas
          + enteringMaintenanceReplicas
          + inMaintenanceReplicas;
      res.totalReplicas += totalReplicasPerBlock;

      boolean isMissing;
      //是否是纠删码  
      if (storedBlock.isStriped()) {
        //如果总共的副本小于最小的副本纠删码丢失  
        isMissing = totalReplicasPerBlock < minReplication;
      } else {
        //如果没有副本了认为丢失  
        isMissing = totalReplicasPerBlock == 0;
      }

      // 计算达到副本要求 副本个数
      short targetFileReplication;
      //判断纠删码  
      if (file.getErasureCodingPolicy() != null) {
        assert storedBlock instanceof BlockInfoStriped;
        targetFileReplication = ((BlockInfoStriped) storedBlock)
            .getRealTotalBlockNum();
      } else {
        targetFileReplication = file.getReplication();
      }
      res.numExpectedReplicas += targetFileReplication;

      // 计算缺少多少副本
      if(totalReplicasPerBlock < minReplication){
        res.numUnderMinReplicatedBlocks++;
      }

      // 计算过多复制了 多少副本
      if (liveReplicas > targetFileReplication) {
        res.excessiveReplicas += (liveReplicas - targetFileReplication);
        res.numOverReplicatedBlocks += 1;
      }

      // 统计坏了多少块
      boolean isCorrupt = lBlk.isCorrupt();
      if (isCorrupt) {
        //增加损坏的块个数和数据大小  
        res.addCorrupt(block.getNumBytes());
        corrupt++;
        corruptSize += block.getNumBytes();
        out.print("\n" + path + ": CORRUPT blockpool " +
            block.getBlockPoolId() + " block " + block.getBlockName() + "\n");
      }

      //计算还需要复制多少副本
      if (totalReplicasPerBlock >= minReplication)
        res.numMinReplicatedBlocks++;

      //块处于复制 状态下丢失的副本数
      if (totalReplicasPerBlock < targetFileReplication && !isMissing) {
        res.missingReplicas += (targetFileReplication - totalReplicasPerBlock);
        res.numUnderReplicatedBlocks += 1;
        underReplicatedPerFile++;
        if (!showFiles) {
          out.print("\n" + path + ": ");
        }
        out.println(" Under replicated " + block + ". Target Replicas is "
            + targetFileReplication + " but found "
            + liveReplicas+ " live replica(s), "
            + decommissionedReplicas + " decommissioned replica(s), "
            + decommissioningReplicas + " decommissioning replica(s)"
            + (this.showMaintenanceState ? (enteringMaintenanceReplicas
            + ", entering maintenance replica(s) and " + inMaintenanceReplicas
            + " in maintenance replica(s).") : "."));
      }

      // 副本存放策略不正确 的块的个数
      BlockPlacementStatus blockPlacementStatus = bpPolicies.getPolicy(
          lBlk.getBlockType()).verifyBlockPlacement(lBlk.getLocations(),
          targetFileReplication);
      if (!blockPlacementStatus.isPlacementPolicySatisfied()) {
        res.numMisReplicatedBlocks++;
        misReplicatedPerFile++;
        if (!showFiles) {
          if(underReplicatedPerFile == 0)
            out.println();
          out.print(path + ": ");
        }
        out.println(" Replica placement policy is violated for " +
                    block + ". " + blockPlacementStatus.getErrorDescription());
        //重新放置块
          if (doReplicate) {
          misReplicatedBlocks.add(storedBlock);
        }
      }

      // 统计存储类型
      if (this.showStoragePolcies && lBlk.getStorageTypes() != null) {
        countStorageTypeSummary(file, lBlk);
      }

      // report
      String blkName = block.toString();
      report.append(blockNumber + ". " + blkName + " len=" +
          block.getNumBytes());
      //块丢失  输出统计信息  
      if (isMissing && !isCorrupt) {
        report.append(" MISSING!");
        res.addMissing(blkName, block.getNumBytes());
        missing++;
        missize += block.getNumBytes();
        if (storedBlock.isStriped()) {
          report.append(" Live_repl=" + liveReplicas);
          String info = getReplicaInfo(storedBlock);
          if (!info.isEmpty()){
            report.append(" ").append(info);
          }
        }
      } else {
        report.append(" Live_repl=" + liveReplicas);
        String info = getReplicaInfo(storedBlock);
        if (!info.isEmpty()){
          report.append(" ").append(info);
        }
      }
      report.append('\n');
      blockNumber++;
    }

    //输出正在UC状态的块信息
    if (!blocks.isLastBlockComplete() && lastBlock != null) {
      ExtendedBlock block = lastBlock.getBlock();
      String blkName = block.toString();
      BlockInfo storedBlock = blockManager.getStoredBlock(
          block.getLocalBlock());
      BlockUnderConstructionFeature uc =
          storedBlock.getUnderConstructionFeature();
      if (uc != null) {
        DatanodeStorageInfo[] storages = uc.getExpectedStorageLocations();
        report.append('\n').append("Under Construction Block:\n")
            .append(blockNumber).append(". ").append(blkName).append(" len=")
            .append(block.getNumBytes())
            .append(" Expected_repl=" + storages.length);
        String info = getReplicaInfo(storedBlock);
        if (!info.isEmpty()) {
          report.append(" ").append(info);
        }
      }
    }

    //输出块损坏并且处理 
    if ((missing > 0) || (corrupt > 0)) {
      if (!showFiles) {
        if (missing > 0) {
          out.print("\n" + path + ": MISSING " + missing
              + " blocks of total size " + missize + " B.");
        }
        if (corrupt > 0) {
          out.print("\n" + path + ": CORRUPT " + corrupt
              + " blocks of total size " + corruptSize + " B.");
        }
      }
      res.corruptFiles++;
      //处于UC状态 不要处理它  
      if (isOpen) {
        LOG.info("Fsck: ignoring open file " + path);
      } else {
        //如果是移动 则移动到 /lost+found 目录
        if (doMove) copyBlocksToLostFound(parent, file, blocks);
        //如果是删除 则删除  
        if (doDelete) deleteCorruptedFile(path);
      }
    }

    if (showFiles) {
      if (missing > 0 || corrupt > 0) {
        if (missing > 0) {
          out.print(" MISSING " + missing + " blocks of total size " +
              missize + " B\n");
        }
        if (corrupt > 0) {
          out.print(" CORRUPT " + corrupt + " blocks of total size " +
              corruptSize + " B\n");
        }
      } else if (underReplicatedPerFile == 0 && misReplicatedPerFile == 0) {
        out.print(" OK\n");
      }
      if (showBlocks) {
        out.print(report + "\n");
      }
    }
    
    //开始处理副本不足的块  
    if (doReplicate && !misReplicatedBlocks.isEmpty()) {
      int processedBlocks = this.blockManager.processMisReplicatedBlocks(
              misReplicatedBlocks);
      if (processedBlocks < misReplicatedBlocks.size()) {
        LOG.warn("Fsck: Block manager is able to process only " +
                processedBlocks +
                " mis-replicated blocks (Total count : " +
                misReplicatedBlocks.size() +
                " ) for path " + path);
      }
      res.numBlocksQueuedForReplication += processedBlocks;
    }
  }
```

小结： 

​      一定不要盲目的认为fsck可以修复损坏的块丢失所有副本的块。

​     如果文件确确实实丢失了通过该命令是无法恢复的，该命令可以让副本不足的块立即开始复制，并不能修复已经丢失所有副本的块。

​    对于已经损坏的块采取如下措施：

> 1. 移动到/lost+found目录的方式
> 2. 直接删除

​    大体执行流程图如下：

<div>
<img src="/images/posts/hadoop-source-23/hadoop03.png" height="480" width="780" />
</div>

### 总结

       本文粗略的分析了fsck源代码，希望可以对读者起到一定的帮助作用


### 参考

* http://hadoop.apache.org
* http://www.github.com/apache/hadoop/hadoop-hdfs/