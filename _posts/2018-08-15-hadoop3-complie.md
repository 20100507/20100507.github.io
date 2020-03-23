---
layout: post
title: "Hadoop3.2.1编译(支持所有native库)"
date: 2018-08-15   
tag: 大数据Hadoop
---

### 前言
    
	Hadoop3新特性,支持纠删码容错处理,支持节点内部磁盘平衡数据,以及多个standyby节点支持.单个集群最大规模可达1W节点.以下记录编译Hadoop3.2.1过程

### 下载安装相关依赖

**下载Hadoop3.2.1源码**

> wget https://github.com/apache/hadoop/archive/branch-3.2.1.zip

**下载JDK&安装JDK**
  
***将java.sh放到/etc/profile.d目录下***

> java.sh的网盘地址: https://pan.baidu.com/s/14SUNP__Ea2p-2L06azP_kw 5afy 
 
**下载安装maven**

***将maven.sh放到/etc/profile.d目录下***

> 1. wget http://mirror.bit.edu.cn/apache/maven/maven-3/3.6.3/binaries/apache-maven-3.6.3-bin.tar.gz
> 2. maven.sh网盘地址：https://pan.baidu.com/s/1drap4XfkZHTap6NRC5dTjg uaos 
> 3. 注释掉hadoop 最外层pom.xml文件中的内容

<br/>

```
 <repositories>
<!--
    <repository>
      <id>${distMgmtSnapshotsId}</id>
      <name>${distMgmtSnapshotsName}</name>
      <url>${distMgmtSnapshotsUrl}</url>
    </repository>
    <repository>
      <id>repository.jboss.org</id>
      <url>https://repository.jboss.org/nexus/content/groups/public/</url>
      <snapshots>
        <enabled>false</enabled>
      </snapshots>
    </repository>
-->
  </repositories>
```

**下载编译安装protobuf**

> 1. protobuf网盘地址: https://pan.baidu.com/s/1bPaGpX7ym14eOAlxefHgUw wtxe
> 2. ./configure --prefix=/opt/protobuf/ 
> 3. make && make install

**下载编译安装cmake3.1**

> 1. cmake网盘地址: https://pan.baidu.com/s/1UaoH70yYEWHMyBf7HnG9ng h6nu
> 2. ./bootstrap
> 3. make && make install

**下载安装findbugs**

> 1. findbugs网盘地址: https://pan.baidu.com/s/1MxByv9EwLIDYEJsL-sxWmA wztw 
> 2. tar -zxvf findbugs.tar.gz /opt

**支持纠删码编码[ISA-L问题解决]**

> 1. nasm rpm包百度网盘地址: https://pan.baidu.com/s/1_KfRyRVBwC8cQWa-TQUzAw pd9i
> 2. git clone https://github.com/01org/isa-l.git
> 3. cd isa-l
> 4. yum install -y yasm
> 5. wget http://mirror.centos.org/centos/8/PowerTools/x86_64/os/Packages/nasm-2.13.03-2.el8.x86_64.rpm
> 6. rpm -ivh nasm-2.13.03-2.el8.x86_64.rpm
> 7. make -f Makefile.unx
> 8. cp bin/libisal.so bin/libisal.so.2 /lib64

**修改系统环境变量**

> 1. 修改/etc/profile文件,添加如下内容
> 2. `export FINDBUGS_HOME=/opt/findbugs-3.0.1`<br/>
     `export PATH=$FINDBUGS_HOME/bin:$PATH`<br/>
     `export PATH=/opt/protobuf/bin:$PATH`<br/>
     `export PATH=/usr/local/bin:$PATH`<br/>

**yum安装相关依赖**

> 1. `yum -y install build-essential autoconf automake libtool cmake zlib1g-dev pkg-config libssl-dev libsasl2-dev`<br/>
     `yum install -y cyrus-sasl*`<br/>
     `yum install -y libgsasl-devel*`<br/>
     `yum install fuse-devel -y`<br/>
     `yum -y install protobuf-devel`<br/>
     `yum -y install snappy`<br/>
     `yum -y install bzip2`<br/>



**检测是否安装成功**

1. 检查cmake

<div align="left">
<img src="/images/posts/hadoop3/cmake.png" height="180" width="1180" />  
</div>

2. 检查protobuf

<div align="left">
<img src="/images/posts/hadoop3/protobuf.png" height="180" width="1180" /> 
</div>

3. 检查maven

<div align="left">
<img src="/images/posts/hadoop3/maven.png" height="180" width="1180" />
</div>

### 编译Hadoop3.2.1源码

**执行mvn命令编译源代码**

```
mvn clean package -Pdist,native -DskipTests -Dtar  -Drequire.snappy
```
**如下图编译成功**
<div align="left">
<img src="/images/posts/hadoop3/hadoop3.png" height="920" width="1180" />
</div>
**native库全部都支持**
<div align="left">
<img src="/images/posts/hadoop3/code.png" height="340" width="1180" />
</div>

**编译好的安装包如下**

> 1. hadoop3.2.1安装包百度网盘链接: 链接：https://pan.baidu.com/s/15HC1ODQRMKjFGDpjK077WA 提取码：zbcs

### 支持LZO压缩

**添加lzo压缩支持**

> 1. 网盘地址: https://pan.baidu.com/s/1PtEf7Y5osPyB0VWxs7y2rw uw47
> 2. 拷贝到 /opt/hadoop/share/hadoop/mapreduce/lib 目录同时配置相关配置文件即可支持LZO压缩