# 深入理解StatefulSet（一）：拓扑状态


## 目录

- [实践环境](#实践环境)

- [实践一：创建基于 Handless Service 方式的 StatefulSet 对象](#实践一)

- [总结](#总结)


## 实践环境

| 系统及软件 | 版本 |
| :--- | :--- |
| CentOS | 7.6.1810 |
| Kernel | 4.18.9-1.el7.elrepo.x86_64 |
| Kubernetes | 1.11.3 |
| Docker CE | 18.09.0 |


本次实践需要提前搭建好 Kubernetes 集群。

---

## 实践一

创建基于 Handless Service 方式的 StatefulSet 对象。

Kubernetes Service 的访问方式：

- 第一种方式，以 Service 的 VIP 方式。

- 第二种方式，以 Service 的 DNS 方式。 这种方式具体可以分为两种方法：
  
  - 第一种处理方法，是 Normal Service，通过 DNS 解析到 VIP，后续的流程就和 VIP 方式相同了。
  
  - 第二种处理方法，是 Handless Service，通过 DNS 直接解析到 Service 代理的某一个 Pod 的 IP 地址。 

因此，Handless Service 是直接以 DNS 记录的方式解析出被代理 Pod 的 IP 地址，并不需要分配一个 VIP。

### 创建 Handless Service

1. 编写 svc.yaml 文件

   [svc.yaml](./svc.yaml) 文件内容如下：

   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: nginx
     labels:
       app: nginx
   spec:
     ports:
       - port: 80
         name: web
     clusterIP: None
     selector:
       app: nginx
   ```

   可以看出，Handless Service 仍是一个标准的 Service 的 Yaml 文件。只不过，它的 ClusterIP 字段的值是 None，即这个 Service 创建后并不会被分配一个 VIP，而是会以 DNS 记录的方式暴露出它所代理的 Pod。

   当创建 Handless Service 之后，它所代理的 Pod 的 IP 地址，都会被绑定一个固定格式的 DNS 记录，如下所示：

   ```
   <pod-name>.<svc-name>.<namespace>.svc.cluster.local
   ```

2. 创建 Handless Service

   ```bash
   $ kubectl create -f svc.yaml 
   service/nginx created

   # 查看 svc
   $ kubectl get service nginx
   NAME      TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
   nginx     ClusterIP   None         <none>        80/TCP    46s
   ```

### 创建 StatefulSet

有了 Handless Service, 并且对应的 StatefulSet 管理的 Pod 都会被绑定为一个固定格式的 DNS 记录，所以只需知道 Pod 的名字与对应 Service 的名字就可以准确的通过 DNS 记录访问到 Pod 的 IP 地址。

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
   ```

   可以发现，StatefulSet 与 Deployment 的 Yaml 文件内容的区别只是多了个 serviceName 字段。

   这个字段的作用就是告诉 StatefulSet 控制器，在执行控制循环（Control Loop）的时候，使用 nginx 这个 Handless Service 来保证 Pod 的「可解析身份」。

2. 创建 StatefulSet

   ```bash
   $ kubectl create -f statefulset.yaml
   statefulset.apps/web created

   # 此时在开启一个终端或者另一个 Master 节点，实时观察 Pod 创建的过程
   # 我在 K8s-master-2 节点上进行观察
   $ kubectl get pods -w -l app=nginx
   NAME      READY     STATUS             RESTARTS  AGE
   web-0     0/1       Pending            0         0s
   web-0     0/1       Pending            0         0s
   web-0     0/1       ContainerCreating  0         0s
   web-0     0/1       ContainerCreating  0         1s
   web-0     1/1       Running            0         2s
   web-1     0/1       Pending            0         0s
   web-1     0/1       Pending            0         0s
   web-1     0/1       ContainerCreating  0         0s
   web-1     0/1       ContainerCreating  0         1s
   web-1     1/1       Running            0         1s

   $ kubectl get statefulset web
   NAME      DESIRED   CURRENT   AGE
   web       2         2         35s

   $ kubectl describe statefulset web
   Events:
   Type    Reason            Age   From                    Message
   ----    ------            ----  ----                    -------
   Normal  SuccessfulCreate  2m    statefulset-controller  create Pod web-0 in StatefulSet web successful
   Normal  SuccessfulCreate  2m    statefulset-controller  create Pod web-1 in StatefulSet web successful

   $ kubectl describe pods -l app=nginx
   Name:           web-0
   ...
   Events:
   Type    Reason     Age   From                 Message
   ----    ------     ----  ----                 -------
   Normal  Scheduled  4m    default-scheduler    Successfully assigned default/web-0 to k8s-node-3
   Normal  Pulled     4m    kubelet, k8s-node-3  Container image "nginx:1.9.1" already present on machine
   Normal  Created    4m    kubelet, k8s-node-3  Created container
   Normal  Started    4m    kubelet, k8s-node-3  Started container

   Name:           web-1
   ...
   Events:
   Type    Reason     Age   From                 Message
   ----    ------     ----  ----                 -------
   Normal  Scheduled  4m    default-scheduler    Successfully assigned default/web-1 to k8s-node-2
   Normal  Pulled     4m    kubelet, k8s-node-2  Container image "nginx:1.9.1" already present on machine
   Normal  Created    4m    kubelet, k8s-node-2  Created container
   Normal  Started    4m    kubelet, k8s-node-2  Started container
   ```

   可以发现，这些 Pod 的创建，是严格按照编号顺序进行创建的。在 web-0 进入到 Running 状态、并且细分状态（Conditions）成为 Ready 之前，web-1 会一直处于 Pending 状态。

3. 验证主机名称

   当 2 个 Pod 都进入 Running 状态之后，可以查看它们的 Hostname。

   ```bash
   $ kubectl exec web-0 -- sh -c 'hostname'
   web-0

   $ kubectl exec web-1 -- sh -c 'hostname'
   web-1
   ```

   可以发现，这 2 个 Pod 的 hostname 与 Pod 名字是一致的，都被分配了对应的编号。

### 验证 DNS 解析解析记录

1. 验证 DNS 解析之前，首先确保 Kubernetes 集群中 CoreDNS 或 Kube DNS 状态正常。

   ```bash
   $ kubectl -n kube-system get pod,svc -l k8s-app=kube-dns
   NAME                            READY     STATUS    RESTARTS   AGE
   pod/kube-dns-778b5d7bd5-5qxmz   3/3       Running   0          2d
   pod/kube-dns-778b5d7bd5-7jcr5   3/3       Running   0          2d

   NAME               TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)         AGE
   service/kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP   2d
   ```

2. 启动一个一次性的 Pod ，使用 nslookup 命令解析一下 Pod 对应的 Headless Service

   ```bash
   $ kubectl run -i --tty --image=busybox:1.28.4 dns-test --restart=Never --rm /bin/sh
   $ nslookup web-0.nginx
   Server:    10.96.0.10
   Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

   Name:      web-0.nginx
   Address 1: 10.244.5.20 web-0.nginx.default.svc.cluster.local

   $ nslookup web-1.nginx
   Server:    10.96.0.10
   Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

   Name:      web-1.nginx
   Address 1: 10.244.4.33 web-1.nginx.default.svc.cluster.local
   ```

   可以发现，在这个 StatefulSet 中这两个新的 Pod 的「网络标识」（web-0.nginx 和 web-1.nginx）都解析到了正确的 IP 地址。

3. 删除 Pod ，再次观察 Pod 的状态变化

   ```bash
   $ kubectl delete pod -l app=nginx 
   pod "web-0" deleted
   pod "web-1" deleted

   $ kubectl get pods -w -l app=nginx
   NAME      READY     STATUS             RESTARTS   AGE
   web-0     1/1       Running            0          13m
   web-1     1/1       Running            0          13m


   web-0     1/1       Terminating        0          13m
   web-1     1/1       Terminating        0          13m
   web-1     0/1       Terminating        0          13m
   web-0     0/1       Terminating        0          13m
   web-0     0/1       Terminating        0          13m
   web-0     0/1       Terminating        0          13m
   web-0     0/1       Pending            0          0s
   web-0     0/1       Pending            0          0s
   web-0     0/1       ContainerCreating  0          0s
   web-0     0/1       ContainerCreating  0          1s
   web-0     1/1       Running            0          2s
   web-1     0/1       Terminating        0          13m
   web-1     0/1       Terminating        0          13m
   web-1     0/1       Pending            0          0s
   web-1     0/1       Pending            0          0s
   web-1     0/1       ContainerCreating  0          0s
   web-1     0/1       ContainerCreating  0          1s
   web-1     1/1       Running            0          1s
   ```

   可以发现，Pod 删除后，Kubernetes 依然会按照原先的编号顺序，创建出两个新的 Pod。并且 Kubernetes 依然为它们分配了与原来相同的「网络标识」。

   通过这种方法，Kubernetes 就成功地将 Pod 的拓扑状态（哪个节点先启动，哪个节点后启动），按照 Pod 的「名字 + 编号」的方式固定了下来。

4. 再次使用 nslookup 解析 Pod 对应的 Headless Service

   ```bash
   $ kubectl run -i --tty --image=busybox:1.28.4 dns-test --restart=Never --rm /bin/sh
   $ nslookup web-0.nginx
   Server:    10.96.0.10
   Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

   Name:      web-0.nginx
   Address 1: 10.244.4.34 web-0.nginx.default.svc.cluster.local

   $ nslookup web-1.nginx
   Server:    10.96.0.10
   Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

   Name:      web-1.nginx
   Address 1: 10.244.5.22 web-1.nginx.default.svc.cluster.local
   ```

   可以发现，虽然容器内 IP 地址发生变化，但 StatefulSet 中这两个新的 Pod 的「网络标识」（web-0.nginx 和 web-1.nginx）依然能解析到正确的 IP 地址。

   Kubernetes 为每一个 Pod 提供了一个固定并且唯一的访问入口，即这个 Pod 对应的 DNS 记录。

---

## 总结

1. StatefulSet 这个控制器的主要作用之一，就是使用 Pod 模板创建 Pod 的时候，对它们进行编号，并且按照编号顺序逐一的完成创建工作。

2. 当 StatefulSet 的「控制循环」发现 Pod 的「实际状态」与「期望状态」不一致，需要新建或者删除 Pod 进行「调谐」的时候，它会严格按照这些 Pod 的编号顺序，逐一完成这些操作。

3. 通过 Headless Service 的方式，StatefulSet 为每个 Pod 创建了一个固定并且稳定的 DNS 记录，来作为它访问的入口。

---