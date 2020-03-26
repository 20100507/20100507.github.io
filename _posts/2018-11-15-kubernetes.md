---
layout: post
title: "Kubernetes记录"
date: 2018-11-15
tag: 容器Kubernetes
---

### 前言
    
	笔者在开发和运维部门的Kubernetes经常遇到一些需要处理的问题,整理如下,作为备忘,同时希望对读者可以起到一定帮助.

### 常用的命令

  查看endpoint关系
> kubectl get ep

   删除被驱逐的pod
> kubectl get pods \| grep Evicted | awk \'{print $1}' | xargs kubectl delete pod

   创建https证书密文
> kubectl create secret tls nginx-test --cert=tls.crt --key=tls.key

   查看k8s自己证书过期时间
> 1. wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
> 2. ./cfssl-certinfo_linux-amd64 -cert /etc/kubernetes/pki/etcd/peer.crt

   node节点加入master节点

   
```
1. kubeadm token list
2. 默认情况下，token的有效期是24小时，如果token已经过期的话，
3. kubeadm token create
4. 如果找不到hash可以执行：
5. openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
6. 登录到node3服务器重新执行加入集群命令就OK了
7. ps:若出现Failed to connect to API Server "<master-ip>:<master-port（6443）>": cluster CA found in cluster-info configmap is invalid: public key sha256:xxxxxxxxxx not pinned
8. kubeadm join --token <token> <master-ip>:<master-port（6443）> --discovery-token-unsafe-skip-ca-verification

ps:token 过期的话会加入不成功提示Unauthorized，换个不过期的token就好了
```

   回滚到某一个指定镜像

> kubectl set image deployment nginx-deployment nginx=192.168.101.88:5000/nginx:1.9.1

   查看部署历史
> kubectl set image deployment nginx-deployment nginx=192.168.101.88:5000/nginx:1.9.1

   回滚到上一个版本
   
> kubectl set image deployment nginx-deployment nginx=192.168.101.88:5000/nginx:1.9.1

  回滚到指定的版本

> kubectl rollout undo deployment nginx-deployment --to-revision=2

  手动扩容pods

> kubectl scale deployment nginx-deployment --replicas=10

  标记污点

> 1. kubectl taint nodes bitnei-saas-k8s-node14 key=value:NoSchedule
> 2. kubectl taint nodes bitnei-saas-k8s-node15 key=value:NoSchedule
> 3. kubectl taint nodes bitnei-saas-k8s-node15 key=value:NoExecute
> 4. kubectl taint nodes bitnei-saas-k8s-node14 key=value:NoExecute

  删除污点

> 1. kubectl taint nodes bitnei-saas-k8s-node14 key=value:NoSchedule-
> 2. kubectl taint nodes bitnei-saas-k8s-node15 key=value:NoSchedule-
> 3. kubectl taint nodes bitnei-saas-k8s-node15 key=value:NoExecute-
> 4. kubectl taint nodes bitnei-saas-k8s-node14 key=value:NoExecute-


### 私有化仓库地址修改

 docker登陆输入用户名免密

> docker login 192.168.64.158

  修改如下配置

```
[root@bitnei-test-k8s-jenkins ~]# cat /etc/docker/daemon.json 
{
      "insecure-registries": [
        "192.168.64.158"
      ]
}
```
### 安装客户端

> 1. 拷贝config 文件到目标机器的/root/.kube目录下
> 2. 拷贝相关的公钥和私钥到客户端
> 3. kubectl get nodes 即可查看


### 无法删除namespace

  kubernetes无法删除namespace 提示 Terminating
  
1. 导出资源清单

> kubectl get namespace xxxxxx -o json > tmp.json

2. 修改如下截图所示

<div align="left">
<img src="/images/posts/kubernetes/1.png" height="380" width="1180" />  
</div>

3. 当前机器打开一个新的窗口,启动代理

> kubectl proxy --port=8081

4. 访问如下链接

> curl -k -H "Content-Type: application/json" -X PUT --data-binary @tmp.json http://127.0.0.1:8001/api/v1/namespaces/namespace的名字/finalize

5. kubectl get ns 可以看到已经删除

<div align="left">
<img src="/images/posts/kubernetes/2.png" height="380" width="1180" />  
</div>


### k8s彻底卸载

**卸载k8s**

> kubeadm reset -f
> modprobe -r ipip
> lsmod
> rm -rf ~/.kube/
> rm -rf /etc/kubernetes/
> rm -rf /etc/systemd/system/kubelet.service.d
> rm -rf /etc/systemd/system/kubelet.service
> rm -rf /usr/bin/kube*
> rm -rf /etc/cni
> rm -rf /opt/cni
> rm -rf /var/lib/etcd
> rm -rf /var/etcd


### kubernetes日志收集解决方案

    `kubernetes`日志收集可以采用挂载NFS方式收集,或者采用守护容器的方式,启动`filebeat`采集日志发送到ES中,来查询日志.相对于第一种方案<br/>
  在生产环境测试发现对pod的运行会产生一定的速度响应上的影响,第二种方案会消耗一部分每一个pod的资源.生产环境建议采用第二种方案.以下为<br/>
  采集`nginx`日志为例,采集jar日志,或者`Python`,`golang`应用日志可以触类旁通.
  
1. 搭建ES比较简单,注意修改linux系统文件打开的限制.在此笔者略过.

2. 如下为nginx pods配置文件,请读者不要直接复制粘贴,注意yaml文件的缩进

```
   apiVersion: v1
   kind: Service
   metadata:
     name: pre-saas-nginx-auth
     namespace: saas-test
   spec:
     selector:
       app: pre-saas-nginx-auth
     ports:
     - name: http
       port: 80
       targetPort: 80
   
   ---
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: pre-saas-nginx-auth
     namespace: saas-test
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: pre-saas-nginx-auth
     template:
       metadata:
         labels:
           app: pre-saas-nginx-auth
       spec:
         containers:
         - name: pre-saas-nginx-auth
           image: 10.10.26.34/saas_test/pre-saas-nginx-auth:20190923090031
           imagePullPolicy: Always
           ports:
           - name: http
             containerPort: 80
           volumeMounts:
           - name: nginx-logs
             mountPath: /var/log/nginx/
         # 此处为filebeat 守护容器
         - name: filebeat
           image: docker.elastic.co/beats/filebeat:6.4.2
           args: [
             "-c", "/etc/filebeat.yml",
             "-e",
           ]
           resources:
             limits:
               memory: 500Mi
             requests:
               cpu: 100m
               memory: 100Mi
           securityContext:
             runAsUser: 0
   		# 由于同一个pod内部共享文件系统,所以把nginx容器日志目录挂载到filebeat容器中
           volumeMounts:
   		  # 这里采用configMap方式注入配置文件
           - name: filebeat-config
             mountPath: /etc/filebeat.yml
             subPath: filebeat.yml
           - name: nginx-logs
             mountPath: /var/log/nginx/
   
         volumes:
         - name: nginx-logs
           emptyDir: {}
         - name: filebeat-config
           configMap:
             name: filebeat-auth-nginx-config
```

3. 如下为filebeat的configmap配置文件,需要读者在es中自己手动创建索引

```
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: filebeat-front-nginx-config
     namespace: saas-test
     
   data:
     filebeat.yml: |-
       filebeat.prospectors:
       - input_type: log
	   # 采集此路径文件
         paths:
           - "/var/log/nginx/*"
	   # 索引名称
       setup.template.name: "saas-test-front-nginx-log"
	   # 索引模板
       setup.template.pattern: "saas-test-front-nginx-log-*"
       output.elasticsearch:
	   # ES地址
         hosts: ["10.10.21.6:9200"]
         index: "saas-test-front-nginx-logs"
```

### 总结

	以上为笔者在开发和运维部门k8s服务中遇到问题和笔记记录,希望对读者起到一定的帮助作用.

### 参考

* https://www.kancloud.cn/huyipow/kubernetes/722822
* https://www.kubernetes.org.cn/5462.html
* https://sealyun.com/post/sealos2.0/
* https://www.jianshu.com/p/71e14c31cb82
* https://mritd.me/
