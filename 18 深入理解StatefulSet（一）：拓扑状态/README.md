# 深入理解StatefulSet（一）：拓扑状态


## 目录

- [实践环境](#实践环境)

- [实践一：创建基于 Handless Service 方式的 StatefulSet 对象](#实践一)


## 实践环境

| 系统及软件 | 版本 |
| :--- | :--- |
| CentOS | 7.6.1810 |
| Kernel | 4.18.9-1.el7.elrepo.x86_64 |
| Kubernetes | 1.11.3 |
| Docker CE | 18.09.0 |


本次实践需要提前搭建好 Kubernetes 集群。


## 实践一

创建基于 Handless Service 方式的 StatefulSet 对象。

