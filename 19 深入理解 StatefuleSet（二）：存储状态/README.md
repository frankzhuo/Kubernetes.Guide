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
        readOnly: true
        user: admin
        keyring: /etc/ceph/keyring
        imageformat: "2"
        imagefeatures: "layering"
```

简要说明一下 volumes 下 rbd 配置的各个字段的含义：

- **monitors**： Ceph 集群的 Monitor 监视器，此次搭建的 Ceph 有 3 个 Monitor 节点。根据当前环境指定 Monitor 节点即可。

- **pool**：Ceph 集群的存储池，Ceph 集群默认创建的 pool 为 rbd，此次使用的 pool 为 kube，因此需要创建 kube 存储池。

- **image**：Ceph 块设备的磁盘镜像文件，此次使用的 image 为 foo，因此需要创建 foo 磁盘镜像。

- **fsType**：文件系统类型，此次使用的 fsType 为 xfs。

- **readOnly**：是否为只读模式，此次为测试使用，开启只读。

- **user**：Ceph 客户端访问 Ceph 集群所使用的用户名，此次使用 admin。

- **keyring**：Ceph 集群认证需要的密钥环，此次使用搭建集群时生成的 ceph.client.admin.keyring 文件。如果觉得不够安全，可以为指定用户新建指定权限的密钥环。

- **imageformat**：磁盘镜像的文件格式。

- **imagefeatures**：磁盘镜像支持的特性。CentOS 7.5 默认内核为  3.10.0-693.5.2.el7.x86_64，只支持 layering 和 exclusive-lock。如果需要支持更多的特性，需要升级内核。


