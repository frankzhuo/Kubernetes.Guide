# 深入理解 StatefuleSet（二）：存储状态实践笔记


## 目录

- [实践环境](#实践环境)

- [实践一：在 POD 中使用 RBD 块设备](#实践一)


## 实践环境

| 系统及软件 | 版本 |
| :--- | :--- |
| CentOS | 7.6.1810 |
| Kernel | 4.18.9-1.el7.elrepo.x86_64 |
| Kubernetes | 1.11.3 |
| Docker CE | 18.09.0 |
| Ceph | jewel-10.2.11 |

本次实践需要提前搭建好 Kubernetes 集群与 Ceph 存储集群。


## 实践一

在 POD 中使用 RBD 块设备。


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
   ```

3. 验证 Pod 是否挂载 Ceph 集群

   ```bash
   $ kubectl describe pod rbd
   Events:
   Type     Reason                  Age   From                     Message
   ----     ------                  ----  ----                     -------
   Normal   Scheduled               3m    default-scheduler        Successfully assigned default/rbd to k8s-node-2
   Normal   SuccessfulAttachVolume  3m    attachdetach-controller  AttachVolume.Attach succeeded for volume "rbdpd"
   Warning  FailedMount             1m    kubelet, k8s-node-2      Unable to mount volumes for pod "rbd_default(4bd5cf95-08e9-11e9-befb-0050569fa5b1)": timeout expired waiting for volumes to attach or mount for pod "default"/"rbd". list of unmounted volumes=[rbdpd]. list of unattached volumes=[rbdpd default-token-qsxnk]
   Normal   Pulling                 1m    kubelet, k8s-node-2      pulling image "kubernetes/pause"
   Normal   Pulled                  1m    kubelet, k8s-node-2      Successfully pulled image "kubernetes/pause"
   Normal   Created                 1m    kubelet, k8s-node-2      Created container
   Normal   Started                 1m    kubelet, k8s-node-2      Started container

   # 在 node 节点上验证容器的挂载信息
   $ docker container ls
   16dcbd0cf614    kubernetes/pause    "/pause"    7 minutes ago    Up 7 minutes    k8s_rbd-rw_rbd_default_4bd5cf95-08e9-11e9-befb-0050569fa5b1_0

   $ docker container inspect --format='{{json .Mounts}}' 16dcbd0cf614 | jq
   [
   {
     "Type": "bind",
     "Source": "/var/lib/kubelet/pods/4bd5cf95-08e9-11e9-befb-0050569fa5b1/volumes/kubernetes.io~rbd/rbdpd",
     "Destination": "/mnt/rbd",
     "Mode": "",
     "RW": true,
     "Propagation": "rprivate"
   },
   ]
   ```
   
