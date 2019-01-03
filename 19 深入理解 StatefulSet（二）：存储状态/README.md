# 深入理解 StatefuleSet（二）：存储状态实践笔记


## 目录

- [实践环境](#实践环境)

- [实践一：在 POD 对象中使用 Ceph RBD](#实践一)

- [实践二：在 POD 对象中使用 Kubernetes Secret 对象认证 Ceph 存储集群](#实践二)

- [实践三：在 POD 对象中通过 Kubernetes PV & PVC 方式使用 Ceph RBD](#实践三)

- [实践四：在 StatefulSet 对象中通过模板自动创建 PVC 并与 PV 绑定 ](#实践四)

- [总结](#总结)

---

## 实践环境

| 系统及软件 | 版本 |
| :--- | :--- |
| CentOS | 7.6.1810 |
| Kernel | 4.18.9-1.el7.elrepo.x86_64 |
| Kubernetes | 1.11.3 |
| Docker CE | 18.09.0 |
| Ceph | jewel-10.2.11 |

本次实践需要提前搭建好 Kubernetes 集群与 Ceph 存储集群。

---

## 实践一

在 POD 对象中使用 RBD。


### 编写 rbd.yaml 文件

首先，编写一个声明了 Ceph RBD 类型的 Volume 的 Pod：[rbd.yaml](./rbd.yaml)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: rbd
spec:
  containers:
    - image: kubernetes/pause
      name: rbd-rw
      volumeMounts:
      - name: rbdpd
        mountPath: /mnt/rbd
  volumes:
    - name: rbdpd
      rbd:
        monitors:
        - '10.10.113.15:6789'
        - '10.10.113.16:6789'
        - '10.10.113.17:6789'
        pool: kube
        image: foo
        fsType: xfs
        readOnly: false
        user: admin
        keyring: /etc/ceph/keyring
#        imageformat: "2"
#        imagefeatures: "layering"
```

简要说明一下 volumes 下 rbd 配置的各个字段的含义：

- **monitors**： Ceph 集群的 Monitor 监视器，此次搭建的 Ceph 有 3 个 Monitor 节点。根据当前环境指定 Monitor 节点即可。

- **pool**：Ceph 集群的存储池，Ceph 集群默认创建的 pool 为 rbd，此次使用的 pool 为 kube，因此需要创建 kube 存储池。

- **image**：Ceph 块设备的磁盘镜像文件，此次使用的 image 为 foo，因此需要创建 foo 磁盘镜像。

- **fsType**：文件系统类型，此次使用的 fsType 为 xfs。

- **readOnly**：是否为只读模式。如果为只读模式不能进行格式化磁盘操作。

- **user**：Ceph 客户端访问 Ceph 集群所使用的用户名，此次使用 admin。

- **keyring**：Ceph 集群认证需要的密钥环，此次使用搭建集群时生成的 ceph.client.admin.keyring 文件。如果觉得不够安全，可以为指定用户新建指定权限的密钥环。

- **imageformat**：磁盘镜像的文件格式。

- **imagefeatures**：磁盘镜像支持的特性。CentOS 7.5 默认内核为  3.10.0-693.5.2.el7.x86_64，只支持 layering 和 exclusive-lock。如果需要支持更多的特性，需要升级内核。


### 创建 RBD 块设备

在创建 Pod 之前，需要根据上述描述的 Volume 来创建磁盘镜像，操作如下： 

1. 创建 Ceph pool
   
   ```bash
   # ceph osd pool create {pool-name} {pg-num} {pgp-num}
   # 此次环境一共有 6 个 OSD，因此 pg-num 取值为 512
   $ ceph osd pool create kube 512 512
   ```

   > pg-num 常用的值：
   >
   >- OSD 数量少于 5 个时，可把 pg-num 设置为 128。
   >- OSD 数量在 5 个到 10 个时，可把 pg-num 设置为 512。
   >- OSD 数量在 10 个 到 50 个时，可把 pg-num 设置为 4096。
   >- OSD 数量大于 50 个时，需要理解权衡方法，自己计算 pg-num 的取值。
   >- 自己计算 pg-num 取值时，可以借助 [pgcalc](https://ceph.com/pgcalc/) 工具。

2. 创建 RBD 磁盘镜像

   ```bash
   # 创建一个大小为 10240M 的 Ceph image
   $ rbd create foo --size 10240M --pool kube

   # 查看 kube 存储池中的磁盘镜像
   $ rbd list --pool kube
   foo

   # 查看 foo 磁盘镜像的信息
   $ rbd --image foo info --pool kube
   rbd image 'foo':
	size 10240 MB in 2560 objects
	order 22 (4096 kB objects)
	block_name_prefix: rbd_data.3a896b8b4567
	format: 2
	features: layering, exclusive-lock, object-map, fast-diff, deep-flatten
	flags:
   ```

3. 关闭内核不支持的特性
   
   ```bash
   # 默认开启的特性的属性值为 61（layering, exclusive-lock, object-map, fast-diff, deep-flatten），此时映射时会报错：
   # RBD image feature set mismatch. You can disable features unsupported by the kernel with "rbd feature disable"
   # 所以需要关闭不支持的特性，只保留 layering
   $ rbd feature disable foo exclusive-lock, object-map, fast-diff, deep-flatten --pool kube

   # 查看特性是否已经关闭
   $ rbd --image foo info --pool kube
   rbd image 'foo':
	size 10240 MB in 2560 objects
	order 22 (4096 kB objects)
	block_name_prefix: rbd_data.3a896b8b4567
	format: 2
	features: layering
	flags:
   ```


### 创建 Pod

1. 将 Ceph 密钥环拷贝至 node 节点

   ```bash
   # 找到搭建 Ceph 集群生成的密钥环文件 ceph.client.admin.keyring
   $ for node in k8s-node-{1,2,3}; do scp ceph.client.admin.keyring root@${node}:/etc/ceph/keyring; done
   ```

2. 创建 Pod 

   ```bash
   $ kubectl create -f rbd.yaml

   $ kubectl describe pod rbd
   Events:
   Type    Reason                  Age   From                     Message
   ----    ------                  ----  ----                     -------
   Normal  Scheduled               54s   default-scheduler        Successfully assigned default/rbd to k8s-node-3
   Normal  SuccessfulAttachVolume  54s   attachdetach-controller  AttachVolume.Attach succeeded for volume "rbdpd"
   Normal  Pulling                 45s   kubelet, k8s-node-3      pulling image "kubernetes/pause"
   Normal  Pulled                  36s   kubelet, k8s-node-3      Successfully pulled image "kubernetes/pause"
   Normal  Created                 36s   kubelet, k8s-node-3      Created container
   Normal  Started                 36s   kubelet, k8s-node-3      Started container

   $ kubectl get pods
   NAME      READY     STATUS    RESTARTS   AGE
   rbd       1/1       Running   0          19s
   ```

3. 验证 Pod 是否挂载 Ceph 存储集群

   ```bash
   # 在 k8s-node-3 节点上验证容器的挂载信息
   $ docker container ls
   16dcbd0cf614    kubernetes/pause    "/pause"    7 minutes ago    Up 7 minutes    k8s_rbd-rw_rbd_default_4bd5cf95-08e9-11e9-befb-0050569fa5b1_0

   $ docker container inspect --format='{{json .Mounts}}' 16dcbd0cf614 | jq
   [
     {
       "Type": "bind",
       "Source": "/var/lib/kubelet/pods/b454c3ab-0a41-11e9-aff7-0050569f6e43/containers/rbd-rw/32817b7a",
       "Destination": "/dev/termination-log",
       "Mode": "",
       "RW": true,
       "Propagation": "rprivate"
     },
     {
       "Type": "bind",
       "Source": "/var/lib/kubelet/pods/b454c3ab-0a41-11e9-aff7-0050569f6e43/volumes/kubernetes.io~rbd/rbdpd",
       "Destination": "/mnt/rbd",
       "Mode": "",
       "RW": true,
       "Propagation": "rprivate"
     },
     {
       "Type": "bind",
       "Source": "/var/lib/kubelet/pods/b454c3ab-0a41-11e9-aff7-0050569f6e43/volumes/kubernetes.io~secret/default-token-qsxnk",
       "Destination": "/var/run/secrets/kubernetes.io/serviceaccount",
       "Mode": "ro",
       "RW": false,
       "Propagation": "rprivate"
     },
     {
       "Type": "bind",
       "Source": "/var/lib/kubelet/pods/b454c3ab-0a41-11e9-aff7-0050569f6e43/etc-hosts",
       "Destination": "/etc/hosts",
       "Mode": "",
       "RW": true,
       "Propagation": "rprivate"
     }
   ]
   ```

---
   
## 实践二

在 POD 对象中使用 Kubernetes Secret 对象认证 Ceph 存储集群。

这种方式和实践和直接使用 keyring 文件认证的区别就是使用 Kubernetes Secret 对象，该 Secret 对象用于 Kubernetes Volume 插件通过 Cephx 认证访问 Ceph 存储集群。

### 创建 Secret 对象

1. 获取 Ceph keyring 并生成 Secret Key

   ```bash
   # 方式一
   $ grep key /etc/ceph/ceph.client.admin.keyring | awk '{ printf $NF}' | base64
   QVFCWXFpQmN6RmZUSUJBQW56Ym5jZVE5alhLMGJBUUxWUTVaWEE9PQ==

   # 方式二
   $ ceph auth get-key client.admin | base64
   QVFCWXFpQmN6RmZUSUJBQW56Ym5jZVE5alhLMGJBUUxWUTVaWEE9PQ==
   ```

2. 编写 ceph-secret.yaml 文件

   [ceph-secret.yaml](./secret/ceph-secret.yaml) 文件内容如下：

   ```yaml
   apiVersion: v1
   kind: Secret
   metadata:
     name: ceph-secret
   type: "kubernetes.io/rbd"  
   data:
     key: QVFCWXFpQmN6RmZUSUJBQW56Ym5jZVE5alhLMGJBUUxWUTVaWEE9PQ==
   ```

   key 的值即为第一步生成的 Secret Key。

3. 创建 Secret 对象

   ```bash
   $ kubectl create -f secret/ceph-secret.yaml
   
   # 查看 Secret 对象是否已经创建
   $ kubectl get secret
   NAME                  TYPE                                  DATA      AGE
   ceph-secret           kubernetes.io/rbd                     1         1m
   ```

### 创建 pod

1. 编写 rbd-with-secret.yaml 文件

   [rbd-with-secret.yaml](./rbd-with-secret.yaml) 文件内容如下：

   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: rbd
   spec:
     containers:
       - image: kubernetes/pause
         name: rbd-rw
         volumeMounts:
         - name: rbdpd
           mountPath: /mnt/rbd
     volumes:
       - name: rbdpd
         rbd:
           monitors:
           - '10.10.113.15:6789'
           - '10.10.113.16:6789'
           - '10.10.113.17:6789'
           pool: kube
           image: foo
           fsType: xfs
           readOnly: false
           user: admin
           secretRef:
             name: ceph-secret
   #        imageformat: "2"
   #        imagefeatures: "layering"
   ```

   发现，rbd-with-secret.yaml 与 rdb.yaml 的区别就在于前者使用的是 secretRef，后者使用的是 keyring。

2. 创建 Pod

   ```bash
   $ kubectl create -f rbd-with-secret.yaml

   $ kubectl describe pod rbd2
   Events:
   Type    Reason                  Age   From                     Message
   ----    ------                  ----  ----                     -------
   Normal  Scheduled               16s   default-scheduler        Successfully assigned default/rbd2 to k8s-node-3
   Normal  SuccessfulAttachVolume  16s   attachdetach-controller  AttachVolume.Attach succeeded for volume "rbdpd"
   Normal  Pulling                 12s   kubelet, k8s-node-3      pulling image "kubernetes/pause"
   Normal  Pulled                  9s    kubelet, k8s-node-3      Successfully pulled image "kubernetes/pause"
   Normal  Created                 9s    kubelet, k8s-node-3      Created container
   Normal  Started                 9s    kubelet, k8s-node-3      Started container

   $ kubectl get pods
   NAME      READY     STATUS    RESTARTS   AGE
   rbd2      1/1       Running   0          1h
   ```

3. 验证 Pod 是否挂载 Ceph 存储集群
    
   ```bash
   $ docker container ls
   242bfdce681f    kubernetes/pause    "/pause"    2 hours ago     Up 2 hours    k8s_rbd-rw_rbd2_default_16a28e20-0a43-11e9-aff7-0050569f6e43_0

   $ docker container inspect --format='{{json .Mounts}}' 242bfdce681f | jq
   [
     {
       "Type": "bind",
       "Source": "/var/lib/kubelet/pods/16a28e20-0a43-11e9-aff7-0050569f6e43/containers/rbd-rw/384ae4be",
       "Destination": "/dev/termination-log",
       "Mode": "",
       "RW": true,
       "Propagation": "rprivate"
     },
     {
       "Type": "bind",
       "Source": "/var/lib/kubelet/pods/16a28e20-0a43-11e9-aff7-0050569f6e43/volumes/kubernetes.io~rbd/rbdpd",
       "Destination": "/mnt/rbd",
       "Mode": "",
       "RW": true,
       "Propagation": "rprivate"
     },
     {
       "Type": "bind",
       "Source": "/var/lib/kubelet/pods/16a28e20-0a43-11e9-aff7-0050569f6e43/volumes/kubernetes.io~secret/default-token-qsxnk",
       "Destination": "/var/run/secrets/kubernetes.io/serviceaccount",
       "Mode": "ro",
       "RW": false,
       "Propagation": "rprivate"
     },
     {
       "Type": "bind",
       "Source": "/var/lib/kubelet/pods/16a28e20-0a43-11e9-aff7-0050569f6e43/etc-hosts",
       "Destination": "/etc/hosts",
       "Mode": "",
       "RW": true,
       "Propagation": "rprivate"
     }
   ]
   ```

---

## 实践三

在 POD 对象中通过 Kubernetes PV & PVC 方式使用 Ceph RBD。

### 创建 PV

首先，先创建一个 PV，认证方式使用 Secret 对象。之前实践中已经创建过了，故不再创建。

1. 编写 pv.yaml 文件
   
   [pv.yaml](./pv.yaml) 文件内容如下：

   ```yaml
   apiVersion: v1
   kind: PersistentVolume
   metadata:
     name: pv-volume
     labels:
       type: local
   spec:
     capacity:
       storage: 10Gi
     accessModes:
       - ReadWriteOnce
     rbd:
       monitors:
       - '10.10.113.15:6789'
       - '10.10.113.16:6789'
       - '10.10.113.17:6789'
       pool: kube
       image: foo
       fsType: xfs
       readOnly: false
       user: admin
       secretRef:
         name: ceph-secret
   ```

2. 创建 PV

   ```bash
   $ kubectl create -f pv.yaml 
   persistentvolume/pv-volume created

   $ kubectl get pv
   NAME        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM     STORAGECLASS   REASON    AGE
   pv-volume   10Gi       RWO            Retain           Available                                      24s
   ```

### 创建 PVC

然后，创建一个 PVC，声明资源请求。

1. 编写 pvc.yaml 文件

   [pvc.yaml](./pvc.yaml) 文件内容如下：

   ```yaml
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: pv-claim
   spec:
     accessModes:
     - ReadWriteOnce
     resources:
       requests:
         storage: 1Gi
   ```

2. 创建 PVC

   ```bash
   $ kubectl create -f pvc.yaml

   $ kubectl get pvc
   NAME       STATUS    VOLUME      CAPACITY   ACCESS MODES   STORAGECLASS   AGE
   pv-claim   Bound     pv-volume   10Gi       RWO                           36s
   ```

### 创建 Pod

最后创建一个应用 Pod，在应用 Pod 中，声明使用这个 PVC。

1. 编写 pvc-pod.yaml 文件

   [pvc-pod.yaml](./pvc-pod.yaml) 文件内容如下:

   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: pv-pod
   spec:
     containers:
       - name: pv-container
         image: nginx
         ports:
           - containerPort: 80
             name: "http-server"
         volumeMounts:
           - mountPath: "/usr/share/nginx/html"
             name: pv-storage
     volumes:
       - name: pv-storage
         persistentVolumeClaim:
           claimName: pv-claim
   ```

2. 创建 Pod

   ```bash
   $ kubectl create -f pvc-pod.yaml
   pod/pv-pod created

   $ kubectl describe pod pv-pod
   Events:
   Type    Reason                  Age   From                     Message
   ----    ------                  ----  ----                     -------
   Normal  Scheduled               1m    default-scheduler        Successfully assigned default/pv-pod to k8s-node-2
   Normal  SuccessfulAttachVolume  1m    attachdetach-controller  AttachVolume.Attach succeeded for volume "pv-volume"
   Normal  Pulling                 1m    kubelet, k8s-node-2      pulling image "nginx"
   Normal  Pulled                  26s   kubelet, k8s-node-2      Successfully pulled image "nginx"
   Normal  Created                 25s   kubelet, k8s-node-2      Created container
   Normal  Started                 25s   kubelet, k8s-node-2      Started container

   $ kubectl get pods
   NAME      READY     STATUS    RESTARTS   AGE
   pv-pod    1/1       Running   0          3m
   ```

3. 验证 Pod 是否挂载 Ceph 存储集群

   ```bash
   # 在 k8s-node-2 节点上验证容器的挂载信息
   $ docker container ls
   8086181373e2    nginx    "nginx -g 'daemon of…"    5 minutes ago    Up 5 minutes    k8s_pv-container_pv-pod_default_36755b95-0a62-11e9-aff7-0050569f6e43_0

   $ docker container inspect --format='{{json .Mounts}}' 8086181373e2 | jq
   [
     {
       "Type": "bind",
       "Source": "/var/lib/kubelet/pods/36755b95-0a62-11e9-aff7-0050569f6e43/volumes/kubernetes.io~rbd/pv-volume",
       "Destination": "/usr/share/nginx/html",
       "Mode": "",
       "RW": true,
       "Propagation": "rprivate"
     },
     {
       "Type": "bind",
       "Source": "/var/lib/kubelet/pods/36755b95-0a62-11e9-aff7-0050569f6e43/volumes/kubernetes.io~secret/default-token-qsxnk",
       "Destination": "/var/run/secrets/kubernetes.io/serviceaccount",
       "Mode": "ro",
       "RW": false,
       "Propagation": "rprivate"
     },
     {
       "Type": "bind",
       "Source": "/var/lib/kubelet/pods/36755b95-0a62-11e9-aff7-0050569f6e43/etc-hosts",
       "Destination": "/etc/hosts",
       "Mode": "",
       "RW": true,
       "Propagation": "rprivate"
     },
     {
       "Type": "bind",
       "Source": "/var/lib/kubelet/pods/36755b95-0a62-11e9-aff7-0050569f6e43/containers/pv-container/59b0ca2a",
       "Destination": "/dev/termination-log",
       "Mode": "",
       "RW": true,
       "Propagation": "rprivate"
     }
   ]
   ```

4. 启动终端验证是否已经挂载 RBD

   ```bash
   $ kubectl exec -ti pv-pod /bin/bash

   $ df -Th
   Filesystem              Type     Size  Used Avail Use% Mounted on
   ...
   /dev/rbd0               xfs       10G   33M   10G   1% /usr/share/nginx/html
   ...
   ```

---

## 实践四

在 StatefulSet 对象中通过模板自动创建 PVC 并与 PV 绑定。

为 StatefulSet 对象额外添加了一个 volumeClaimTemplates 字段。凡是被这个 StatefulSet 对象管理的 Pod，都会声明一个对应的 PVC。而这个 PVC 的定义，就来自于 volumeClaimTeplates 这个模板字段。并且，这个 PVC 的名字会被分配一个与这个 Pod 完全一致的编号。 

因为我之后编写的 [statefulset.yaml](./statefulset.yaml) 文件中的副本数为 2 ，所以会自动创建 2 个 PVC。而一个 PVC 对应一个 PV ，所以需要创建两个 PV。又因为一个 PV 对应一个 RBD，所以需要创建两个 RBD。

这个实践，我创建 2 个新的磁盘镜像以及 2 个新的 PV 提供给 StatefulSet 使用。

### 创建 RBD 块设备

以下为创建 RBD 的操作步骤（详细说明参考实践一）：

```bash
$ rbd create www-1 --size 10240M --pool kube
$ rbd create www-2 --size 10240M --pool kube

$ rbd list --pool kube
foo
www-1
www-2

$ rbd feature disable www-1 exclusive-lock, object-map, fast-diff, deep-flatten --pool kube
$ rbd feature disable www-2 exclusive-lock, object-map, fast-diff, deep-flatten --pool kube

$ rbd --image www-1 info --pool kube
rbd image 'www-1':
   size 10240 MB in 2560 objects
   order 22 (4096 kB objects)
   block_name_prefix: rbd_data.628f6b8b4567
   format: 2
   features: layering
   flags:
$ rbd --image www-2 info --pool kube
rbd image 'www-2':
   size 10240 MB in 2560 objects
   order 22 (4096 kB objects)
   block_name_prefix: rbd_data.62c66b8b4567
   format: 2
   features: layering
   flags:
```

### 创建 PV

1. 编写 pv-www-1.yaml 与 pv-www-2.yaml 文件

   [pv-www-1.yaml](./pv-www-1.yaml) 与 [pv-www-2.yaml](./pv-www-2.yaml) 文件内容如下：

   ```yaml
   # pv-www-1.yaml
   apiVersion: v1
   kind: PersistentVolume
   metadata:
     name: pv-www-1
     labels:
       type: local
   spec:
     capacity:
       storage: 10Gi
     accessModes:
       - ReadWriteOnce
     rbd:
       monitors:
       - '10.10.113.15:6789'
       - '10.10.113.16:6789'
       - '10.10.113.17:6789'
       pool: kube
       image: www-1
       fsType: xfs
       readOnly: false
       user: admin
       secretRef:
         name: ceph-secret
   #    imageformat: "2"
   #    imagefeatures: "layering"

   # pv-www-2.yaml
   apiVersion: v1
   kind: PersistentVolume
   metadata:
     name: pv-www-2
     labels:
       type: local
   spec:
     capacity:
       storage: 10Gi
     accessModes:
       - ReadWriteOnce
     rbd:
       monitors:
       - '10.10.113.15:6789'
       - '10.10.113.16:6789'
       - '10.10.113.17:6789'
       pool: kube
       image: www-2
       fsType: xfs
       readOnly: false
       user: admin
       secretRef:
         name: ceph-secret
   #    imageformat: "2"
   #    imagefeatures: "layering"
   ```

2. 创建 PV

   ```bash
   $ kubectl create -f pv-www-1.yaml
   persistentvolume/pv-www-1 created
   $ kubectl create -f pv-www-2.yaml
   persistentvolume/pv-www-2 created

   $ kubectl get pv
   NAME        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM              STORAGECLASS   REASON    AGE
   pv-volume   10Gi       RWO            Retain           Bound       default/pv-claim                            3h
   pv-www-1    10Gi       RWO            Retain           Available                                               1m
   pv-www-2    10Gi       RWO            Retain           Available                                               33s
   ```

### 创建 StatefulSet

1. 编写 statefulset.yaml 文件

   [statefulset.yaml](./statefulset.yaml) 文件内容如下：

   ```yaml
   apiVersion: apps/v1
   kind: StatefulSet
   metadata:
     name: web
   spec:
     serviceName: "nginx"
     replicas: 2
     selector:
       matchLabels:
         app: nginx
     template:
       metadata:
         labels:
           app: nginx
       spec:
         containers:
           - name: nginx
             image: nginx:1.9.1
             ports:
               - containerPort: 80
                 name: web
             volumeMounts:
             - name: www
               mountPath: /usr/share/nginx/html
     volumeClaimTemplates:
     - metadata:
         name: www
       spec:
         accessModes:
         - ReadWriteOnce
         resources:
           requests:
             storage: 1Gi
   ```

2. 创建 StatefulSet

   ```bash
   $ kubectl create -f statefulset.yaml
   statefulset.apps/web created

   $ kubectl describe statefulset web
   Events:
   Type    Reason            Age   From                    Message
   ----    ------            ----  ----                    -------
   Normal  SuccessfulCreate  52s   statefulset-controller  create Claim www-web-0 Pod web-0 in StatefulSet web success
   Normal  SuccessfulCreate  52s   statefulset-controller  create Pod web-0 in StatefulSet web successful
   Normal  SuccessfulCreate  47s   statefulset-controller  create Claim www-web-1 Pod web-1 in StatefulSet web success
   Normal  SuccessfulCreate  47s   statefulset-controller  create Pod web-1 in StatefulSet web successful

   $ kubectl get statefulset
   NAME      DESIRED   CURRENT   AGE
   web       2         2         1m

   $ kubectl get pods
   NAME      READY     STATUS    RESTARTS   AGE
   pv-pod    1/1       Running   0          41m
   web-0     1/1       Running   0          40s
   web-1     1/1       Running   0          30s

   $ kubectl get pvc -l app=nginx
   NAME        STATUS    VOLUME      CAPACITY   ACCESS MODES   STORAGECLASS   AGE
   www-web-0   Bound     pv-www-2    10Gi       RWO                           1m
   www-web-1   Bound     pv-www-1    10Gi       RWO                           57s

   $ kubectl get pv
   NAME        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS    CLAIM               STORAGECLASS   REASON    AGE
   pv-volume   10Gi       RWO            Retain           Bound     default/pv-claim                             3h
   pv-www-1    10Gi       RWO            Retain           Bound     default/www-web-1                            2m
   pv-www-2    10Gi       RWO            Retain           Bound     default/www-web-0                            2m
   ```

### 验证 PV 的使用

在 Pod 的 Volume 目录里写入一个文件：

```bash
$ for i in 0 1; do kubectl exec web-$i -- sh -c 'echo hello $(hostname) > /usr/share/nginx/html/index.html'; done
```

获取这 2 个容器的 Ip 地址：

```bash
$ for i in 0 1; do kubectl exec web-$i -- ip a | grep inet.*eth0; done
    inet 10.244.4.16/32 scope global eth0
    inet 10.244.5.17/32 scope global eth0
```

然后在容器外访问 Nginx：

```bash
$ curl 10.244.4.16 10.244.5.17
hello web-0
hello web-1
```

最后验证删掉这 2 个 Pod，重新创建出来的 2 个 Pod 的数据是否还存在：

```bash
$ kubectl delete pod -l app=nginx
pod "web-0" deleted
pod "web-1" deleted

# 等待 Pod 重新创建完成
$ kubectl get pod -l app=nginx
NAME      READY     STATUS    RESTARTS   AGE
web-0     1/1       Running   0          48s
web-1     1/1       Running   0          31s

$ for i in 0 1; do kubectl exec web-$i -- ip a | grep inet.*eth0; done
    inet 10.244.5.18/32 scope global eth0
    inet 10.244.4.17/32 scope global eth0

$ curl 10.244.5.18 10.244.4.17
hello web-0
hello web-1
```

可以发现，删除掉 Pod 之后，这个 Pod 对应的 PVC 与 PV 并不会被删除。而这个 Volume 里已经写入的数据，也依然会保存在 Ceph 存储集群中。

---

## 总结

1. **StatefulSet 的控制器直接管理的是 Pod。**<br/>
   StatefulSet 里不同 Pod 实例，不再像 RepilcaSet 中那样都是完全一样的，而是有了细微的区别。比如每个 Pod 的 hostname、名字等都是不同的，携带了编号的。而 StatefulSet 区分这些实例的方式，就是通过在 Pod 的名字里加上事先约定好的编号。

2. **Kubernetes 通过 Headless Service，为这些有编号的 Pod，在 DNS 服务器中生成带有同样编号的 DNS 记录。**<br/>
   只要 StatefulSet 能够保证这些 Pod 名字里的编号不变，那么 Service 里类似与 web-0.nginx.default.svc.cluster.local 这样的 DNS 记录也就不会变，而这条记录解析出来的 Pod 的 IP 地址，则会随着后端 Pod 的删除和再创建而自动更新。这当然是 Service 机制本身的能力，不需要 StatefulSet 操心。

3. **StatefulSet 还会为每一个 Pod 分配并创建同样编号的 PVC。**<br/>
   这样，Kubernetes 就可以通过 Persistent Volume 机制为这个 PVC 绑定上对应的 PV，从而保证了每一个 Pod 都拥有一个独立的 Volume。在这种情况下，即使 Pod 被删除，它所对应的 PVC 和 PV 依然会保留下来。所以当这个 Pod 被重新创建出来之后，Kubernetes 会为它找到同样编号的 PVC，挂载这个 PVC 对应的 Volume，从而获取以前保存在 Volume 里的数据。 

---