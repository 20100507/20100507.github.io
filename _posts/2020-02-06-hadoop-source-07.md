---
layout: post
title: "Hadoop源码分析-07-INode"
date: 2020-02-06
tag: Hadoop源码分析
---

### 前言

    经常使用HDFS,一定听说过文件文件夹修改权限,但是听说过INode的可能不多,如果你对Linux底层比较熟悉的话,可能会了解到Linux存储文件的方式
  是采用INode方式存储,HDFS也是仿照Linux底层实现了一套INode元数据文件的存储方式.其中涉及到在开发和使用中不常用的功能,比如XAttr,ACL,快照
  等.本文笔者就介绍和简单分析一下HDFS INode结构.

### 类结构梳理

如下图INode周边类的结构

<div>
<img src="/images/posts/hadoop-source-07/hadoop01.jpg" height="1100" width="1180" />
</div>

* 结构梳理

  从上图我们可以看出,整体围绕`INode`类展开,以`INodeAttributes`为最底层的接口,其中包含了设置用户,访问时间,ACL,XAttr,修改时间等.

### 核心类详细解析

> * INode
> * INodeAttributes
> * INodeAttributeProvider
> * INodeDirectoryAttributes
> * INodeDirectory
> * INodeFileAttributes
> * INodeFile
> * INodeMap
> * INodeReference
> * INodesInPath
> * INodeWithAdditionalFields
> * INodeId
> * INodeSymlink

以下我们一一讲解每个类的用处,以部分类的部分方法的逻辑解析

* INode

   判断是否是根节点

```
final boolean isRoot() {
  return getLocalNameBytes().length == 0;
}
```

   设置用户，其中包含了快照版本ID

```
final INode setUser(String user, int latestSnapshotId) {
  recordModification(latestSnapshotId);
  setUser(user);
  return this;
}
```

   设置权限，这里所指的权限是依据Linux权限

```
INode setPermission(FsPermission permission, int latestSnapshotId) {
  recordModification(latestSnapshotId);
  setPermission(permission);
  return this;
}
```

  添加XAttr【文件或者文件夹的其他属性】

```
INode setPermission(FsPermission permission, int latestSnapshotId) {
  recordModification(latestSnapshotId);
  setPermission(permission);
  return this;
}
```

  添加ACL权限，区别于Linux的权限

```
  abstract void addAclFeature(AclFeature aclFeature);
```

  添加XAttr【文件或者文件夹的其他属性】

```
  abstract void addXAttrFeature(XAttrFeature xAttrFeature);
```
  获取指定目录的的祖先目录

```
   public final boolean isAncestorDirectory(final INodeDirectory dir) {
  for(INodeDirectory p = getParent(); p != null; p = p.getParent()) {
    if (p == dir) {
      return true;
    }
  }
  return false;
}
```
  查看最后修改时间

```
abstract long getModificationTime(int snapshotId);
```

* INodeAttributes

  INodeAttributes部分方法如下:

  判断是否是目录

```
   public boolean isDirectory();
```

  获取权限

```
  public FsPermission getFsPermission()
```

* INodeAttributeProvider

   将路径解析为String数组
   
```
  String[] getPathElements(String path) {
  path = path.trim();
  if (path.charAt(0) != Path.SEPARATOR_CHAR) {
    throw new IllegalArgumentException("It must be an absolute path: " +
        path);
  }
  int numOfElements = StringUtils.countMatches(path, Path.SEPARATOR);
  if (path.length() > 1 && path.endsWith(Path.SEPARATOR)) {
    numOfElements--;
  }
  String[] pathElements = new String[numOfElements];
  int elementIdx = 0;
  int idx = 0;
  int found = path.indexOf(Path.SEPARATOR_CHAR, idx);
  while (found > -1) {
    if (found > idx) {
      pathElements[elementIdx++] = path.substring(idx, found);
    }
    idx = found + 1;
    found = path.indexOf(Path.SEPARATOR_CHAR, idx);
  }
  if (idx < path.length()) {
    pathElements[elementIdx] = path.substring(idx);
  }
  return pathElements;
}
```

* INodeDirectoryAttributes

   获取配额的数量

```
public QuotaCounts getQuotaCounts();
```
  
  其中包含两个静态内部类 包括快照拷贝，基于配额的拷贝`SnapshotCopy``CopyWithQuota`

* INodeFile

  INode和Block以及BlockInfo之间的映射
  
* INodeFileAttributes

  是否是条纹的 纠删码

```
boolean isStriped();
```

* INodeId

  父节点的INodeId 为16385 子节点分别为16384..16383..16382  0为未来保留做兼容

* INodeMap

  INodeId和INode映射关系

* INodeReference

  对索引节点的匿名引用,当文件/目录存储在某些快照中并被重命名/移动到其他位置时,它可能具有多个访问路径。

* INodesInPath

  从给定的路径获取INode

* INodeSymlink

  HDFS软连接INode

### 总结

   以上为笔者总结INode源代码,感兴趣的读者可以继续深入学习,希望本文对读者起到一定的帮助.

### 参考

* https://github.com/apache/hadoop
* http://hadoop.apache.org/
