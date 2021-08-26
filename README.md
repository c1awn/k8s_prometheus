自用记录 kube-prometheus安装prometheus
>写在前面：  
>prometheus-operator官方地址：https://github.com/prometheus-operator/prometheus-operator  
kube-prometheus官方地址：https://github.com/prometheus-operator/kube-prometheus  
两个项目的关系：前者只包含了Prometheus Operator，后者既包含了Operator，又包含了Prometheus相关组件的部署及常用的Prometheus自定义监控，具体包含下面的组件  
The Prometheus Operator：创建CRD自定义的资源对象  
Highly available Prometheus：创建高可用的Prometheus  
Highly available Alertmanager：创建高可用的告警组件  
Prometheus node-exporter：创建主机的监控组件  
Prometheus Adapter for Kubernetes Metrics APIs：创建自定义监控的指标工具（例如可以通过nginx的request来进行应用的自动伸缩）  
kube-state-metrics：监控k8s相关资源对象的状态指标  
Grafana：进行图像展示  

正文开始
- prom是快速安装版，供快速体验Prometheus。不含持久化数据，不保存监控数据
- prom-nfs是动态PV持久化安装
> - 安装前注意使用的k8s系统版本，本次试验版本为1.21.0。生产版本可能<=1.18。k8s版本不一致可能出现异常。本文[主要参考](https://www.mairoot.com/?p=3168 "主要参考")
>- 仅供试验，不适合直接上生产。
>- [镜像版本选择参考](https://www.cnblogs.com/Heroge/p/12457148.html "镜像版本选择参考")

```
[root@k8s-master prom]# kubectl version
Client Version: version.Info{Major:"1", Minor:"21", GitVersion:"v1.21.0", GitCommit:"cb303e613a121a29364f75cc67d3d580833a7479", GitTreeState:"clean", BuildDate:"2021-04-08T16:31:21Z", GoVersion:"go1.16.1", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"21", GitVersion:"v1.21.0", GitCommit:"cb303e613a121a29364f75cc67d3d580833a7479", GitTreeState:"clean", BuildDate:"2021-04-08T16:25:06Z", GoVersion:"go1.16.1", Compiler:"gc", Platform:"linux/amd64"}
```

## 1. prom快速安装版
### 1.1.进入k8s master节点，下载项目，进入prom目录，修改暴露端口（理论上已经修改好）
```
 修改Prometheus和Grafana访问端口，模式为NodePort模式
$ cd ~/prom
$ vim prometheus/prometheus-service.yaml
spec:
  type: NodePort             # 新增，NodePort类型
  ports:
  - name: web
    port: 9090
    targetPort: web
    nodePort: 30009         # 新增，对集群外访问端口

$ vim grafana/grafana-service.yaml
spec:
  type: NodePort            # 新增，NodePort类型
  ports:
  - name: http
    port: 3000
    targetPort: http
    nodePort: 30003        # 新增，对集群外访问端口
```
### 1.2 下载镜像
由于众所周知的原因，直接用外网下载会出现故障。本次用阿里仓库代替。下面的镜像供参考:
```
#需要的镜像名称及连接
prometheus-operator:v0.37.0  quay.io/coreos/prometheus-operator:v0.37.0
alertmanager:v0.20.0         quay.io/prometheus/alertmanager:v0.20.0
grafana:6.6.0                grafana/grafana:6.6.0
kube-state-metrics:v1.9.5    quay.io/coreos/kube-state-metrics:v1.9.5   #v1.9.5有问题，建议用1.9.4版本
kube-rbac-proxy:v0.4.1       quay.io/coreos/kube-rbac-proxy:v0.4.1
node-exporter:v0.18.1        quay.io/prometheus/node-exporter:v0.18.1
k8s-prometheus-adapter-amd64:v0.5.0  quay.io/coreos/k8s-prometheus-adapter-amd64:v0.5.0
prometheus:v2.15.2           quay.io/prometheus/prometheus:v2.15.2
configmap-reload:v0.3.0      jimmidyson/configmap-reload:v0.3.0
prometheus-config-reloader:v0.37.0   quay.io/coreos/prometheus-config-reloader:v0.37.0

温馨提示：只需要在某一个node节点上把以上的镜像pull回来，再通过保存打包发送到其它的node节点并导入即可
 kube-state-metrics:v1.9.5有问题，会一直提示Error,用回1.9.4版本没有问题，有二个方法：
 方法1：可以把kube-state-metrics:v1.9.4下载回来。修改标签为quay.io/coreos/kube-state-metrics:v1.9.5
 方法2：保留标签为quay.io/coreos/kube-state-metrics:v1.9.4，去manifest文件里面，修改kube-state-metrics-*.yaml的所有文件的标签1.9.5为1.9.4
```
```
docker pull registry.cn-hangzhou.aliyuncs.com/yfhub/prometheus-operator:v0.37.0
docker pull registry.cn-hangzhou.aliyuncs.com/yfhub/alertmanager:v0.20.0
docker pull registry.cn-hangzhou.aliyuncs.com/yfhub/grafana:6.6.0
docker pull registry.cn-hangzhou.aliyuncs.com/yfhub/kube-state-metrics:v1.9.4
docker pull registry.cn-hangzhou.aliyuncs.com/yfhub/kube-rbac-proxy:v0.4.1
docker pull registry.cn-hangzhou.aliyuncs.com/yfhub/node-exporter:v0.18.1
docker pull registry.cn-hangzhou.aliyuncs.com/yfhub/k8s-prometheus-adapter-amd64:v0.5.0
docker pull rregistry.cn-hangzhou.aliyuncs.com/yfhub/prometheus:v2.15.2
docker pull registry.cn-hangzhou.aliyuncs.com/yfhub/configmap-reload:v0.3.0
docker pull registry.cn-hangzhou.aliyuncs.com/yfhub/prometheus-config-reloader:v0.37.0
```
### 1.3重新给镜像打tag
如果拉起pod的时候有image pull报错，describe pod查看拉取的是哪个版本，重新打tag。
```
docker tag registry.cn-hangzhou.aliyuncs.com/yfhub/prometheus-operator:v0.37.0 quay.io/coreos/prometheus-operator:v0.37.0

docker tag registry.cn-hangzhou.aliyuncs.com/yfhub/alertmanager:v0.20.0  quay.io/prometheus/alertmanager:v0.20.0

docker tag registry.cn-hangzhou.aliyuncs.com/yfhub/grafana:6.6.0  grafana/grafana:6.6.0

docker tag registry.cn-hangzhou.aliyuncs.com/yfhub/kube-state-metrics:v1.9.4 quay.io/coreos/kube-state-metrics:v1.9.5

docker tag registry.cn-hangzhou.aliyuncs.com/yfhub/kube-rbac-proxy:v0.4.1 quay.io/coreos/kube-rbac-proxy:v0.4.1

docker tag registry.cn-hangzhou.aliyuncs.com/yfhub/node-exporter:v0.18.1 quay.io/prometheus/node-exporter:v0.18.1

docker tag registry.cn-hangzhou.aliyuncs.com/yfhub/k8s-prometheus-adapter-amd64:v0.5.0   quay.io/coreos/k8s-prometheus-adapter-amd64:v0.5.0

docker tag registry.cn-hangzhou.aliyuncs.com/yfhub/prometheus:v2.15.2 quay.io/prometheus/prometheus:v2.15.2

docker tag registry.cn-hangzhou.aliyuncs.com/yfhub/configmap-reload:v0.3.0 jimmidyson/configmap-reload:v0.3.0

docker tag registry.cn-hangzhou.aliyuncs.com/yfhub/prometheus-config-reloader:v0.37.0 quay.io/coreos/prometheus-config-reloader:v0.37.0

docker tag registry.cn-hangzhou.aliyuncs.com/yfhub/prometheus-config-reloader:v0.37.0  quay.io/coreos/prometheus-config-reloader:v0.37.0
```
删除阿里tag，可选
```
docker rmi registry.cn-hangzhou.aliyuncs.com/yfhub/prometheus-operator:v0.37.0
docker rmi registry.cn-hangzhou.aliyuncs.com/yfhub/alertmanager:v0.20.0
docker rmi registry.cn-hangzhou.aliyuncs.com/yfhub/grafana:6.6.0
docker rmi registry.cn-hangzhou.aliyuncs.com/yfhub/kube-state-metrics:v1.9.4
docker rmi registry.cn-hangzhou.aliyuncs.com/yfhub/kube-rbac-proxy:v0.4.1
docker rmi registry.cn-hangzhou.aliyuncs.com/yfhub/node-exporter:v0.18.1
docker rmi registry.cn-hangzhou.aliyuncs.com/yfhub/k8s-prometheus-adapter-amd64:v0.5.0
docker rmi rregistry.cn-hangzhou.aliyuncs.com/yfhub/prometheus:v2.15.2
docker rmi registry.cn-hangzhou.aliyuncs.com/yfhub/configmap-reload:v0.3.0
docker rmi registry.cn-hangzhou.aliyuncs.com/yfhub/prometheus-config-reloader:v0.37.0
```
镜像打包，传输到其他节点，load image。当然其他节点也可以重复拉取阿里镜像，重新打tag。其他节点必须要有镜像，不然启动会失败。
```
dock save -o image_name.tar image
docker load -i image_name.tar
```
### 1.4 执行部署apply
```
kubectl apply -f setup/
kubectl apply -f adapter/
kubectl apply -f alertmanager/
kubectl apply -f node-exporter/
kubectl apply -f kube-state-metrics/
kubectl apply -f grafana/
kubectl apply -f prometheus/
kubectl apply -f serviceMonitor/
```
查看状态，如果有image pull报错，describe pod查看拉取的是哪个版本，重新打tag后系统自动部署。
```
[root@k8s-master prom]# kubectl get pod -n monitoring
NAME                                     READY   STATUS    RESTARTS   AGE
alertmanager-main-0                      2/2     Running   0          23h
alertmanager-main-1                      2/2     Running   0          23h
alertmanager-main-2                      2/2     Running   0          23h
grafana-c568fc65f-6khxf                  1/1     Running   0          22h
kube-state-metrics-74964b6cd4-hhr55      3/3     Running   0          28h
nfs-client-provisioner-9bff7844b-l5m2v   1/1     Running   13         22h
node-exporter-8dsdn                      2/2     Running   6          5d22h
node-exporter-l67rg                      2/2     Running   2          5d20h
node-exporter-tfhl2                      2/2     Running   10         5d22h
prometheus-adapter-5b8db7955f-4qvkq      1/1     Running   0          28h
prometheus-adapter-5b8db7955f-kx4zb      1/1     Running   0          28h
prometheus-k8s-0                         2/2     Running   0          22h
prometheus-operator-75d9b475d9-b7lvq     2/2     Running   0          28h
```
### 1.5访问页面
```
grafana：http://ip:30003/         账号密码：admin/admin
prometheus：http://ip:30009/           无账号密码
```
至此，快速安装已完成。期间碰到的问题只有image拉取报错，重新打tag后即可。
## 2.prom-nfs 持久化安装
### 2.1安装NFS
```
准备一台NFS服务端（这里使用master节点，IP为：192.168.15.130）：
$ sudo yum install -y nfs-utils rpcbind;mkdir -p /data/apps/nfs/pub
$ cat /etc/exports
/data/apps/nfs/pub 192.168.15.0/24(rw,async,no_root_squash)
$ sudo systemctl enable nfs && sudo systemctl restart nfs

NFS客户端（集群中的node节点都为客户端，IP为：192.168.15.130，192.168.15.131，192.168.15.132）：
$ sudo yum install -y nfs-utils rpcbind
$ showmount -e 192.168.15.130      --检查一下是否正常
Export list for 192.168.15.130:
/data/apps/nfs/pub 192.168.15.0/24
```
### 2.2 在已有快速安装的基础上部署
```
cd prom-nfs
$ kubectl apply -f nfs/
$ kubectl apply -f prometheus/prometheus-prometheus.yaml
$ kubectl apply -f grafana/grafana-deployment.yaml 
```
这一步遇到异常是grafana和prometheus的pod一直是pending，describe显示has unbound immediate PersistentVolumeClaims，进一步describe pvc，显示waiting for a volume to be created, either by external provisioner。搜索得到的[办法](https://kuboard.cn/install/faq/selfLink.html "办法")是
```
Kubernetes v1.20 (opens new window)开始，默认删除了 metadata.selfLink 字段，然而，部分应用仍然依赖于这个字段，例如 nfs-client-provisioner。如果仍然要继续使用这些应用，您将需要重新启用该字段。即：
 cat /etc/kubernetes/manifests/kube-apiserver.yaml
apiVersion: v1
···
    - --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
    - --feature-gates=RemoveSelfLink=false # 添加这个配置
```
修改后无需重启apiservice，apiservice不存在重启，可以重启的有kubelet。  
另外，PV和PVC有自己的规则，比如PVC<=PV的大小，超过会报错。更多细节需查阅文档。  
### 2.3 查看状态
```
[root@k8s-master prom]# kubectl get pvc -n monitoring
NAME                                 STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
grafana-pvc                          Bound    pvc-978703ea-c0ea-4978-94e6-d0f8d3e72e66   10Gi       RWO            nfs-storage    23h
prometheus-k8s-db-prometheus-k8s-0   Bound    pvc-b07ab296-e697-4e32-b1cd-47060c8b5c50   10Gi       RWO            nfs-storage    23h
[root@k8s-master prom]# kubectl get pv -n monitoring
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                                           STORAGECLASS   REASON   AGE
alert-pv                                   20Gi       RWO            Delete           Available                                                   prometheus              47d
grafana-pv                                 10Gi       RWO            Delete           Available                                                   prometheus              47d
prometheus-pv                              70Gi       RWO            Delete           Available                                                   prometheus              47d
pvc-978703ea-c0ea-4978-94e6-d0f8d3e72e66   10Gi       RWO            Delete           Bound       monitoring/grafana-pvc                          nfs-storage             3h33m
pvc-b07ab296-e697-4e32-b1cd-47060c8b5c50   10Gi       RWO            Delete           Bound       monitoring/prometheus-k8s-db-prometheus-k8s-0   nfs-storage             3h33m
[root@k8s-master prom]# kubectl get pods -n monitoring
NAME                                     READY   STATUS    RESTARTS   AGE
alertmanager-main-0                      2/2     Running   0          23h
alertmanager-main-1                      2/2     Running   0          23h
alertmanager-main-2                      2/2     Running   0          23h
grafana-c568fc65f-6khxf                  1/1     Running   0          23h
kube-state-metrics-74964b6cd4-hhr55      3/3     Running   0          29h
nfs-client-provisioner-9bff7844b-l5m2v   1/1     Running   13         23h
node-exporter-8dsdn                      2/2     Running   6          5d23h
node-exporter-l67rg                      2/2     Running   2          5d21h
node-exporter-tfhl2                      2/2     Running   10         5d23h
prometheus-adapter-5b8db7955f-4qvkq      1/1     Running   0          29h
prometheus-adapter-5b8db7955f-kx4zb      1/1     Running   0          29h
prometheus-k8s-0                         2/2     Running   0          22h
prometheus-operator-75d9b475d9-b7lvq     2/2     Running   0          29h
```
````
[root@k8s-master prom]# ls /data/apps/nfs/pub/ -lh  自动生成的持久文件，格式按照：[命名空间-服务名-pvc-随机码]组成
total 0
drwxrwxrwx 5 root root 61 Aug 26 14:22 monitoring-grafana-pvc-pvc-978703ea-c0ea-4978-94e6-d0f8d3e72e66
drwxrwxrwx 3 root root 27 Aug 26 10:50 monitoring-prometheus-k8s-db-prometheus-k8s-0-pvc-b07ab296-e697-4e32-b1cd-47060c8b5c50
```
以上
