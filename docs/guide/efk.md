### 第一部分：EFK

`EFK` 插件是`k8s`项目的一个日志解决方案，它包括三个组件：[Elasticsearch](), [Fluentd](), [Kibana]()；Elasticsearch 是日志存储和日志搜索引擎，Fluentd 负责把`k8s`集群的日志发送给 Elasticsearch, Kibana 则是可视化界面查看和检索存储在 Elasticsearch 的数据。

### 准备 

下载官方最新[release](https://github.com/kubernetes/kubernetes/release)，进入目录: `kubernetes/cluster/addons/fluentd-elasticsearch`，参考官方配置的基础上使用本项目`manifests/efk/`部署，以下为几点主要的修改：

+ 官方提供的`kibana-deployment.yaml`中的参数`SERVER_BASEPATH`在k8s v1.8 版本以后部署需要按照本项目调整
+ 修改官方docker镜像，方便国内下载加速

### 安装

``` bash
$ kubectl create -f /etc/ansible/manifests/efk/
$ kubectl create -f /etc/ansible/manifests/efk/es-without-pv/
```

**注意**：Fluentd 是以 DaemonSet 形式运行且只会调度到有`beta.kubernetes.io/fluentd-ds-ready=true`标签的节点，所以对需要收集日志的节点逐个打上标签：

``` bash
$ kubectl label nodes 192.168.1.2 beta.kubernetes.io/fluentd-ds-ready=true
node "192.168.1.2" labeled
```

### 验证

``` bash
kubectl get pods -n kube-system|grep -E 'elasticsearch|fluentd|kibana'
elasticsearch-logging-0                    1/1       Running   0          19h
elasticsearch-logging-1                    1/1       Running   0          19h
fluentd-es-v2.0.2-6c95c                    1/1       Running   0          17h
fluentd-es-v2.0.2-f2xh8                    1/1       Running   0          8h
fluentd-es-v2.0.2-pv5q5                    1/1       Running   0          8h
kibana-logging-d5cffd7c6-9lz2p             1/1       Running   0          1m
```
kibana Pod 第一次启动时会用较长时间(10-20分钟)来优化和 Cache 状态页面，可以查看 Pod 的日志观察进度，如下等待 `Ready` 状态

``` bash
$ kubectl logs -n kube-system kibana-logging-d5cffd7c6-9lz2p -f
...
{"type":"log","@timestamp":"2018-03-13T07:33:00Z","tags":["listening","info"],"pid":1,"message":"Server running at http://0:5601"}
{"type":"log","@timestamp":"2018-03-13T07:33:00Z","tags":["status","ui settings","info"],"pid":1,"state":"green","message":"Status changed from uninitialized to green - Ready","prevState":"uninitialized","prevMsg":"uninitialized"}
```

### 访问 Kibana

这里介绍 `kube-apiserver`方式访问，获取访问 URL

``` bash
$ kubectl cluster-info | grep Kibana
Kibana is running at https://192.168.1.10:8443/api/v1/namespaces/kube-system/services/kibana-logging/proxy
```
浏览器访问 URL：`https://192.168.1.10:8443/api/v1/namespaces/kube-system/services/kibana-logging/proxy`，然后使用`basic auth（参照hosts文件设置，默认：用户admin 密码test1234）`或者`证书` 的方式认证后即可，关于认证可以参考[dashboard文档](dashboard.md)

首次登陆需要在`Management` - `Index Patterns` 创建 `index pattern`，可以使用默认的 logstash-* pattern，点击 Create; 创建Index后，稍等几分钟就可以在 Discover 菜单看到 ElasticSearch logging 中汇聚的日志；

### 第二部分：日志持久化之静态PV
日志数据是存放于 `Elasticsearch POD`中，但是默认情况下它使用的是`emptyDir`存储类型，所以当 `POD`被删除或重新调度时，日志数据也就丢失了。以下讲解使用`NFS` 服务器手动（静态）创建`PV` 持久化保存日志数据的例子。

#### 配置 NFS

+ 准备一个nfs服务器，如果没有可以参考[nfs-server](nfs-server.md)创建。 
+ 准备nfs服务器的共享目录，即修改`/etc/exports` 包含如下，修改后重启`systemctl restart nfs-server`。

``` bash
/share          192.168.1.*(rw,sync,insecure,no_subtree_check,no_root_squash)
/share/es0      192.168.1.*(rw,sync,insecure,no_subtree_check,no_root_squash)
/share/es1      192.168.1.*(rw,sync,insecure,no_subtree_check,no_root_squash)
/share/es2      192.168.1.*(rw,sync,insecure,no_subtree_check,no_root_squash)
```

#### 使用静态 PV安装 EFK

``` bash
# 如果之前已经安装了默认的EFK，请用以下两个命令先删除它
$ kubectl delete -f /etc/ansible/manifests/efk/
$ kubectl delete -f /etc/ansible/manifests/efk/es-without-pv/

# 安装静态PV 的 EFK
$ kubectl create -f /etc/ansible/manifests/efk/
$ kubectl create -f /etc/ansible/manifests/efk/es-static-pv/
```
+ 目录`es-static-pv` 下首先是利用 NFS服务预定义了三个 PV资源，然后在 `es-statefulset.yaml`定义中使用 `volumeClaimTemplates` 去匹配使用预定义的 PV资源；注意 PV参数：`accessModes` `storageClassName` `storage`容量大小必须两边匹配。 

#### 验证安装

+ 1.集群中查看 `pod` `pv` `pvc` 等资源

``` bash
$ kubectl get pods -n kube-system|grep -E 'elasticsearch|fluentd|kibana'
elasticsearch-logging-0                    1/1       Running   0          10m
elasticsearch-logging-1                    1/1       Running   0          10m
fluentd-es-v2.0.2-6c95c                    1/1       Running   0          10m
fluentd-es-v2.0.2-f2xh8                    1/1       Running   0          10m
fluentd-es-v2.0.2-pv5q5                    1/1       Running   0          10m
kibana-logging-d5cffd7c6-9lz2p             1/1       Running   0          10m

$ kubectl get pv
NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                                                       STORAGECLASS       REASON    AGE
pv-es-0   4Gi        RWX            Recycle          Bound       kube-system/elasticsearch-logging-elasticsearch-logging-0   es-storage-class             1m
pv-es-1   4Gi        RWX            Recycle          Bound       kube-system/elasticsearch-logging-elasticsearch-logging-1   es-storage-class             1m
pv-es-2   4Gi        RWX            Recycle          Available                                                               es-storage-class             1m

$ kubectl get pvc --all-namespaces
NAMESPACE     NAME                                            STATUS    VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS       AGE
kube-system   elasticsearch-logging-elasticsearch-logging-0   Bound     pv-es-0   4Gi        RWX            es-storage-class   2m
kube-system   elasticsearch-logging-elasticsearch-logging-1   Bound     pv-es-1   4Gi        RWX            es-storage-class   1m
```

+ 2.网页访问 `kibana`查看具体的日志，如上须等待（约15分钟） `kibana Pod`优化和 Cache 状态页面，达到 `Ready` 状态。

+ 3.登陆 NFS Server 查看对应目录和内部数据

``` bash
$ ls /share
es0  es1  es2
```

### 第三部分：日志持久化之动态PV
`PV` 作为集群的存储资源，`StatefulSet` 依靠它实现 POD的状态数据持久化，但是当 `StatefulSet`动态伸缩时，它的 `PVC`请求也会变化，如果每次都需要管理员手动去创建对应的 `PV`资源，那就很不方面；因此 K8S还提供了 `provisioner`来动态创建 `PV`，不仅节省了管理员的时间，还可以根据不同的 `StorageClasses`封装不同类型的存储供 PVC 选用。

+ 此功能需要 `API-SERVER` 参数 `--admission-control`字符串设置中包含 `DefaultStorageClass`，本项目中已经开启。
+ `provisioner`指定 Volume 插件的类型，包括内置插件（如 kubernetes.io/glusterfs）和外部插件（如 external-storage 提供的 ceph.com/cephfs，nfs-client等），以下讲解使用 `nfs-client-provisioner`来动态创建 `PV`来持久化保存 `EFK`的日志数据。

#### 配置 NFS（同上）

确保 `/etc/exports` 配置如下共享目录，并确保 `/share`目录可读可写权限，否则可能因为权限问题无法动态生成 PV的对应目录。
``` bash
/share          192.168.1.*(rw,sync,insecure,no_subtree_check,no_root_squash)
```

#### 使用动态 PV安装 EFK

``` bash
# 如果之前已经安装了默认的EFK或者静态PV EFK，请用以下命令先删除它
$ kubectl delete -f /etc/ansible/manifests/efk/
$ kubectl delete -f /etc/ansible/manifests/efk/es-without-pv/
$ kubectl delete -f /etc/ansible/manifests/efk/es-static-pv/

# 安装动态PV 的 EFK
$ kubectl create -f /etc/ansible/manifests/efk/
$ kubectl create -f /etc/ansible/manifests/efk/es-dynamic-pv/
```
+ 首先 `nfs-client-provisioner.yaml` 创建一个工作 POD，它监听集群的 PVC请求，并当 PVC请求来到时调用 `nfs-client` 去请求 `nfs-server`的存储资源，成功后即动态生成对应的 PV资源。
+ `nfs-dynamic-storageclass.yaml` 定义 NFS存储类型的类型名 `nfs-dynamic-class`，然后在 `es-statefulset.yaml`中必须使用这个类型名才能动态请求到资源。

#### 验证安装

+ 1.集群中查看 `pod` `pv` `pvc` 等资源

``` bash
$ kubectl get pods -n kube-system|grep -E 'elasticsearch|fluentd|kibana'
elasticsearch-logging-0                    1/1       Running   0          10m
elasticsearch-logging-1                    1/1       Running   0          10m
fluentd-es-v2.0.2-6c95c                    1/1       Running   0          10m
fluentd-es-v2.0.2-f2xh8                    1/1       Running   0          10m
fluentd-es-v2.0.2-pv5q5                    1/1       Running   0          10m
kibana-logging-d5cffd7c6-9lz2p             1/1       Running   0          10m

$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS    CLAIM                                                       STORAGECLASS        REASON    AGE
pvc-50644f36-358b-11e8-9edd-525400cecc16   4Gi        RWX            Delete           Bound     kube-system/elasticsearch-logging-elasticsearch-logging-0   nfs-dynamic-class             10m
pvc-5b105ee6-358b-11e8-9edd-525400cecc16   4Gi        RWX            Delete           Bound     kube-system/elasticsearch-logging-elasticsearch-logging-1   nfs-dynamic-class             10m

$ kubectl get pvc --all-namespaces
NAMESPACE     NAME                                            STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS        AGE
kube-system   elasticsearch-logging-elasticsearch-logging-0   Bound     pvc-50644f36-358b-11e8-9edd-525400cecc16   4Gi        RWX            nfs-dynamic-class   10m
kube-system   elasticsearch-logging-elasticsearch-logging-1   Bound     pvc-5b105ee6-358b-11e8-9edd-525400cecc16   4Gi        RWX            nfs-dynamic-class   10m
```

+ 2.网页访问 `kibana`查看具体的日志，如上须等待（约15分钟） `kibana Pod`优化和 Cache 状态页面，达到 `Ready` 状态。

+ 3.登陆 NFS Server 查看对应目录和内部数据

``` bash
$ ls /share # 可以看到类似如下的目录生成
kube-system-elasticsearch-logging-elasticsearch-logging-0-pvc-50644f36-358b-11e8-9edd-525400cecc16
kube-system-elasticsearch-logging-elasticsearch-logging-1-pvc-5b105ee6-358b-11e8-9edd-525400cecc16
```

### 参考

1. [EFK 配置](https://github.com/kubernetes/kubernetes/tree/master/cluster/addons/fluentd-elasticsearch)
1. [nfs-client-provisioner](https://github.com/kubernetes-incubator/external-storage/tree/master/nfs-client)
1. [persistent-volume](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims)
1. [storage-classes](https://kubernetes.io/docs/concepts/storage/storage-classes/)


[前一篇](ingress.md) -- [目录](index.md) -- [后一篇](harbor.md)
