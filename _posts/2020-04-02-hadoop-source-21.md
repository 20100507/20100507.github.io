---
layout: post
title: "Hadoop源码分析-21-HttpFS"
date: 2020-04-02
tag: Hadoop源码分析
---

### 前言

    HttpFs提供 了REST 风格的网关服务，可以支持所有的HDFS的操作，并且可以和 webhdfs进行交互.同时HttpFs可以和在不同版本的hadoop集群之间传输数据，避免了RPC的版本问题。

### 优缺点

  * 优点

> 1. 可避开防火墙访问HDFS，充当服务的网关
> 2. 可跨语言进行访问，由于是http访问的方式
> 3. webhdfs客户端可以使用HDFS已有的文件系统工具来访问HttpFS
> 4. 内置安全访问机制，可以自定义身份认证插件，提供 了HDFS的访问用户代理

  * 缺点

> 1. 后端为Jetty容器，存在高并发问题，有可能宕机


### 部署&使用


  * 修改配置

  修改core-site.xml配置文件，把#HTTPFSUSER#替换为要启动该服务的Linux用户 

```
  <property>
    <name>hadoop.proxyuser.#HTTPFSUSER#.hosts</name>
    <value>httpfs-host.foo.com</value>
  </property>
  <property>
    <name>hadoop.proxyuser.#HTTPFSUSER#.groups</name>
    <value>*</value>
  </property>

```

  * 启动httpfs&重启集群

  启动后默认监听端口为14000

```
  dfs-stop.sh
  dfs-start.sh
  hdfs --daemon start httpfs
```

  * 使用

1.   获取垃圾箱目录

   `curl 'http://192.168.149.131:14000/webhdfs/v1/user/foo?op=GETTRASHROOT&user.name=foo'`

2. 查看目录结构 hdfs dfs -ls /  

   `curl 'http://192.168.149.131:14000/webhdfs/v1/user/foo?op=LISTSTATUS&user.name=foo'`

3. cat文件 hdfs dfs -cat /orgs/dingran/edits.stats

   `curl 'http://192.168.149.131:14000/webhdfs/v1/orgs/dingran/edits.stats?op=OPEN&user.name=foo'`
   
4. 创建文件夹 hdfs dfs -mkdir /orgs/dingran/bianqi 
   
   `curl -X PUT 'http://192.168.149.131:14000/webhdfs/v1/orgs/dingran/bianqi?op=MKDIRS&user.name=hadoop'`
   
5. 删除文件(夹)  hdfs dfs -rm /orgs/dingran/XXBB

   `curl -X DELETE 'http://192.168.149.131:14000/webhdfs/v1/orgs/dingran/XXBB?op=DELETE&user.name=hadoop'`

6. 上传文件 hdfs dfs -put httpfs-test.txt  /orgs/dingran/bianqi.txt

   `curl -i -X PUT -T httpfs-test.txt "http://192.168.149.131:14000/webhdfs/v1/orgs/dingran/bianqi.txt?blocksize=1048576&replication=1&op=CREATE&data=true&user.name=hadoop&permission=755&noredirect=true&buffersize=20000&overwrite=true" -H "Content-Type:application/octet-stream"`
   
   

### 创建文件夹源码

  * HttpFSServer类

​     进入该类的put方法，put方法接受所有的http的put请求，该方法使用`switch`判断不同的操作码，其中以`MKDIRS`为例.

 ```java
@PUT
  @Path("{path:.*}")
  @Consumes({"*/*"})
  @Produces({MediaType.APPLICATION_JSON + "; " + JettyUtils.UTF_8})
  public Response put(InputStream is,
                       @Context UriInfo uriInfo,
                       @PathParam("path") String path,
                       @QueryParam(OperationParam.NAME) OperationParam op,
                       @Context Parameters params,
                       @Context HttpServletRequest request)
    throws IOException, FileSystemAccessException {
    UserGroupInformation user = HttpUserGroupInformation.get();
    Response response;
    path = makeAbsolute(path);
    MDC.put(HttpFSFileSystem.OP_PARAM, op.value().name());
    MDC.put("hostname", request.getRemoteAddr());
    switch (op.value()) {
      case MKDIRS: {
      case MKDIRS: {
        Short permission = params.get(PermissionParam.NAME,
                                       PermissionParam.class);
        Short unmaskedPermission = params.get(UnmaskedPermissionParam.NAME,
            UnmaskedPermissionParam.class);
        //根据路径权限权限掩码获取命令
        FSOperations.FSMkdirs command =
            new FSOperations.FSMkdirs(path, permission, unmaskedPermission);
        //执行创建文件夹
        JSONObject json = fsExecute(user, command);
        AUDIT_LOG.info("[{}] permission [{}] unmaskedpermission [{}]",
            path, permission, unmaskedPermission);
        //返回JSON响应
        response = Response.ok(json).type(MediaType.APPLICATION_JSON).build();
        break;
      }
      case SETTIMES: {
        ...
        break;
      }
      default: {
        throw new IOException(
          MessageFormat.format("Invalid HTTP PUT operation [{0}]",
                               op.value()));
      }
    }
    return response;
  }
 ```

* FSOperations类

  ```java
  @Override
    public JSONObject execute(FileSystem fs) throws IOException {
      FsPermission fsPermission = new FsPermission(permission);
      if (unmaskedPermission != -1) {
        fsPermission = FsCreateModes.create(fsPermission,
            new FsPermission(unmaskedPermission));
      }
      //创建文件夹
      boolean mkdirs = fs.mkdirs(path, fsPermission);
      return toJSON(HttpFSFileSystem.MKDIRS_JSON, mkdirs);
    }
  }
  ```

### 总结

       本文源代码仅仅是稍微简单提了一些 ，并没有说到启动以及`Filter`过滤器以及安全等模块.感兴趣的读者可以下载源代码阅读与理解.
       在Hadoop提供了一个独立的模块,名称为`hadoop-hdfs-httpfs`.笔者在阅读其中的文档发现了其中一个小问题，已经提交社区 iira编号为 HDFS-15376


### 参考

* http://hadoop.apache.org/docs/current/hadoop-hdfs-httpfs/ServerSetup.html
* https://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-hdfs/WebHDFS.html