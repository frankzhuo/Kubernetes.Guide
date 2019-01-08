# 深入理解 StatefulSet（三）：有状态应用实践


## 目录

- [实践环境](#实践环境)

- [实践一：使用 StatefulSet 搭建 MySQL 主从集群](#实践一)

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

使用 StatefulSet 搭建一个 MySQL 主从集群。

需求描述：

1. 部署的「有状态应用」是一个「主从复制（Master-Slave Replication）」的 MySQL 集群；

2. 有 1 个 Master 节点；

3. 有多个 Slave 节点；

4. Slave 节点需要能水平扩展；

5. 所有的写操作，只能在主节点上执行；

6. 读操作可以在所有节点上执行。

如果将部署 MySQL 主从集群的流程迁移到 Kubernetes 项目上，需要能「容器化」地解决下面的问题：

1. Master 节点和 Slave 节点需要有不同的配置文件；

2. Master 节点和 Slave 节点需要能传输备份信息文件；

3. Slave 节点第一次启动之前，需要执行一些初始化 SQL 操作。

### 使用 Namespace 为 MySQL 分配命名空间

1. 编写 mysql-namespace.yaml 文件

   [mysql-namespace.yaml](./mysql-namespace.yaml) 文件内容如下：

   ```yaml
   apiVersion: v1
   kind: Namespace
   metadata:
     name: mysql
     labels:
       app: mysql
   ```

2. 创建 Namespace

   ```bash
   $ kubectl create -f mysql-namespace.yaml 
   namespace/mysql created

   $ kubectl get namespace -l app=mysql
   NAME      STATUS    AGE
   mysql     Active    6s
   ```

### 使用 ConfigMap 为 Master/Slave 节点分配不同的配置文件

1. 编写 mysql-configmap.yaml 文件

   [mysql-configmap.yaml](./mysql-configmap) 文件内容如下：

   ```yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: mysql
     namespace: mysql
     labels:
       app: mysql
   data:
     master.cnf: |
       # Master 节点 MySQL 配置文件
       [mysqld]
       log-bin=mysqllog
       skip-name-resolve
     slave.cnf: |
       # Slave 节点 MySQL 配置文件
       [mysqld]
       super-read-only
       skip-name-resolve
       log-bin=mysql-bin
       replicate-ignore-db=mysql
   ```

2. 创建 ConfigMap

   ```bash
   $ kubectl create -f mysql-configmap.yaml
   configmap/mysql created

   $ kubectl -n mysql get cm
   NAME      DATA      AGE
   mysql     2         21s

   $ kubectl -n mysql describe cm mysql
   Name:         mysql
   Namespace:    mysql
   Labels:       app=mysql
   Annotations:  <none>

   Data
   ====
   master.cnf:
   ----
   # Master 节点 MySQL 配置文件
   [mysqld]
   log-bin=mysqllog
   skip-name-resolve

   slave.cnf:
   ----
   # Slave 节点 MySQL 配置文件
   [mysqld]
   super-read-only
   skip-name-resolve
   log-bin=mysql-bin
   replicate-ignore-db=mysql

   Events:  <none>
   ```

### 使用 Service 为 MySQL 提供读写分离

1. 编写 mysql-services.yaml

   [mysql-services.yaml](./mysql-services.yaml) 文件如下：

   ```yaml
   # Headless Service，通过 DNS 直接解析到 Service 代理的某一个 Pod 的 IP 地址
   apiVersion: v1
   kind: Service
   metadata:
     name: mysql
     namespace: mysql
     labels:
       app: mysql
   spec:
     ports:
     - name: mysql
       port: 3306
     clusterIP: None
     selector:
       app: mysql
   ---
   # Normal Service，通过 DNS 解析到 VIP
   apiVersion: v1
   kind: Service
   metadata:
     name: mysql-read
     namespace: mysql
     labels:
       app: mysql
   spec:
     ports:
     - name: mysql
       port: 3306
     selector:
       app: mysql
   ```

   - 用户所有写请求，必须以 DNS 记录的方式直接访问到 Master 节点，也就是 mysql-0.mysql 这条 DNS 记录。

   - 用户所有读请求，必须访问自动分配的 DNS 记录可以被转发到任意一个 Master 或 Slave 节点上，也就是 mysql-read 这条 DNS 记录。

2. 创建 Service

   ```bash
   $ kubectl create -f mysql-services.yaml 
   service/mysql created
   service/mysql-read created

   kubectl -n mysql get service -l app=mysql
   NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
   mysql        ClusterIP   None           <none>        3306/TCP   18s
   mysql-read   ClusterIP   10.100.51.22   <none>        3306/TCP   18s
   ```

### 使用 Secret 存储 MySQL ROOT 用户密码

1. 对密码进行 base64 编码

   ```bash
   $ echo -n "123456" | base64
   MTIzNDU2
   ```

   > 使用 echo 命令时，必须要加 -n 参数去掉结尾的换行符。

2. 编写 mysql-secret.yaml

   [mysql-secret.yaml](./mysql-secret.yaml) 文件内容如下：

   ```yaml
   apiVersion: v1
   kind: Secret
   metadata:
     name: mysql-secret
     namespace: mysql
     labels:
       app: mysql
   type: Opaque
   data:
     password: MTIzNDU2 # 123456
   ```

### 

3. 创建 Secret

   ```bash
   $ kubectl create -f mysql-secret.yaml 
   secret/mysql-secret created

   kubectl -n mysql get secret -l app=mysql
   NAME           TYPE      DATA      AGE
   mysql-secret   Opaque    1         14s
   ```

### 使用 StorageClass 为 MySQL 提供后端存储

1. 编写 storageclass.yaml 文件

   [storageclass.yaml](storageclass.yaml) 文件内容如下：

   ```yaml
   kind: StorageClass
   apiVersion: storage.k8s.io/v1
   metadata:
     name: block-service
   provisioner: kubernetes.io/rbd
   parameters:
     monitors: 10.10.113.15:6789,10.10.113.16:6789,10.10.113.17:6789
     adminId: admin
     adminSecretName: ceph-secret
     adminSecretNamespace: default
     pool: kube
     userId: admin
     userSecretName: ceph-secret
     userSecretNamespace: default
     fsType: xfs
     imageFormat: "2"
     imageFeatures: "layering"
   ```

   关于 StorageClass 内容以后学习中会有更详细的实践，各个字段可以参考官方资料：[Storage-class/ceph-rbd](https://kubernetes.io/docs/concepts/storage/storage-classes/#ceph-rbd)

2. 创建 StorageClass

   ```bash
   $ kubectl create -f storageclass.yaml 
   storageclass.storage.k8s.io/block-service created

   $ kubectl -n mysql get sc
   NAME            PROVISIONER         AGE
   block-service   kubernetes.io/rbd   7s
   ```

   > StorageClass 是全局可见的，不需要指定 namespace 空间。

### 使用 StatefulSet 搭建 MySQL 主从集群

首先，描述一个大致框架：

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: mysql
  labels:
   app: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  serviceName: mysql
  replicas: 3
  template:
    metadata:
      labels:
        app: mysql
    spec:
      initContainers:
      - name: init-mysql
      - name: clone-mysql
      containers:
      - name: mysql
      - name: xtrabackup
      volumes:
      - name: conf
        emptyDir: {}
      - name: config-map
        configMap:
          name: mysql
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes:
      - "ReadWriteOnce"
      storageClassName: block-service
      resources:
        requests:
          storage: 30Gi
```

这个框架大致描述如下：

1. 首先有 3 个节点：1 个 Master 节点与 2 个 Slave 节点；

2. 使用 init-mysql 这个 initContainers 进行配置文件的初始化；

3. 使用 clone-mysql 这个 initContainers 进行数据的传输；

4. 使用 xtrabackup 这个 sidecar 容器进行：
   
      1. 初始化 SQL 操作；

      2. 启动一个数据传输的服务提供给下一个节点进行数据的传输；

5. 最后，使用 mysql 这个容器提供数据库服务。


接下来，一部一部完成各个容器模板的描述：

1. 描述 init-mysql

   ```yaml
   ...
         - name: init-mysql
           image: mysql:5.7
           env:
           - name: MYSQL_ROOT_PASSWORD
             valueFrom:
               secretKeyRef:
                 name: mysql-secret
                 key: password
           command:
           - bash
           - "-c"
           - |
             set -ex
             # 从 Pod 的序号，生成 server-id
             [[ $(hostname) =~ -([0-9]+)$ ]] || exit 1
             ordinal=${BASH_REMATCH[1]}
             echo [mysqld] > /mnt/conf.d/server-id.cnf

             # 由于 server-id 不能为 0，因此给 ID 加 100 来避开它
             echo server-id=$((100 + $ordinal)) >> /mnt/conf.d/server-id.cnf

             # 如果 Pod 的序号为 0，说明它是 Master 节点，从 ConfigMap 里把 Master 的配置文件拷贝到 /mnt/conf.d 目录下
             # 否则，拷贝 ConfigMap 里的 Slave 的配置文件
             if [[ ${ordinal} -eq 0 ]]; then
               cp /mnt/config-map/master.cnf /mnt/conf.d
             else
               cp /mnt/config-map/slave.cnf /mnt/conf.d
             fi
           volumeMounts:
           - name: conf
             mountPath: /mnt/conf.d
           - name: config-map
             mountPath: /mnt/config-map
   ...
   ```

2. 描述 clone-mysql

   ```yaml
   ...
         - name: clone-mysql
           image: gcr.io/google-samples/xtrabackup:1.0
           env:
           - name: MYSQL_ROOT_PASSWORD
             valueFrom:
               secretKeyRef:
                 name: mysql-secret
                 key: password
           command:
           - bash
           - "-c"
           - |
             set -ex
             # 拷贝操作只需要在第一次启动时进行，所以数据已经存在则跳过
             [[ -d /var/lib/mysql/mysql ]] && exit 0

             # Master 节点（序号为 0）不需要这个操作
             [[ $(hostname) =~ -([0-9]+)$ ]] || exit 1
             ordinal=${BASH_REMATCH[1]}
             [[ $ordinal == 0 ]] && exit 0

             # 使用 ncat 指令，远程地从前一个节点拷贝数据到本地
             ncat --recv-only mysql-$(($ordinal-1)).mysql 3307 | xbstream -x -C /var/lib/mysql

             # 执行 --prepare，这样拷贝来的数据就可以用作恢复了
             xtrabackup --prepare --target-dir=/var/lib/mysql
           volumeMounts:
           - name: data
             mountPath: /var/lib/mysql
             subPath: mysql
           - name: conf
             mountPath: /etc/mysql/conf.d
   ...
   ```

3. 描述 xtrabackup

   ```yaml
   ...
         - name: xtrabackup
           image: gcr.io/google-samples/xtrabackup:1.0
           ports:
           - name: xtrabackup
             containerPort: 3307
           env:
           - name: MYSQL_ROOT_PASSWORD
             valueFrom:
               secretKeyRef:
                 name: mysql-secret
                 key: password
           command:
           - bash
           - "-c"
           - |
             set -ex
             cd /var/lib/mysql

             # 从备份信息文件里读取 MASTER_LOG_FILE 和 MASTER_LOG_POS 这 2 个字段的值，用来拼装集群初始化 SQL
             if [[ -f xtrabackup_slave_info ]]; then
               # 如果 xtrabackup_slave_info 文件存在，说明这个备份数据来自于另一个 Slave 节点
               # 这种情况下，XtraBackup 工具在备份的时候，就已经在这个文件里自动生成了 "CHANGE MASTER TO" SQL 语句
               # 所以，只需要把这个文件重命名为 change_master_to.sql.in，后面直接使用即可
               mv xtrabackup_slave_info change_master_to.sql.in
               # 所以，也就用不着 xtrabackup_binlog_info 了
               rm -f xtrabackup_binlog_info
             elif [[ -f xtrabackup_binlog_info ]]; then
               # 如果只是存在 xtrabackup_binlog_info 文件，说明备份来自于 Master 节点，就需要解析这个备份信息文件，读取所需的两个字段的值
               [[ $(cat xtrabackup_binlog_info) =~ ^(.*?)[[:space:]]+(.*?)$ ]] || exit 1
               rm xtrabackup_binlog_info
               # 把两个字段的值拼装成 SQL，写入 change_master_to.sql.in 文件
               echo "CHANGE MASTER TO MASTER_LOG_FILE='${BASH_REMATCH[1]}',\
                     MASTER_LOG_POS=${BASH_REMATCH[2]}" > change_master_to.sql.in
             fi

             # 如果存在 change_master_to.sql.in，就意味着需要做集群初始化工作
             if [[ -f change_master_to.sql.in ]]; then
               # 但一定要先等 MySQL 容器启动之后才能进行下一步连接 MySQL 的操作
               echo "Waiting for mysqld to be ready（accepting connections）"
               until mysql -h 127.0.0.1 -uroot -p${MYSQL_ROOT_PASSWORD} -e "SELECT 1"; do sleep 1; done

               echo "Initializing replication from clone position"
               # 将文件 change_master_to.sql.in 改个名字
               # 防止这个 Container 重启的时候，因为又找到了 change_master_to.sql.in，从而重复执行一遍初始化流程
               mv change_master_to.sql.in change_master_to.sql.orig
               # 使用 change_master_to.sql.orig 的内容，也就是前面拼装的 SQL，组成一个完整的初始化和启动 Slave 的 SQL 语句
               mysql -h 127.0.0.1 -uroot -p${MYSQL_ROOT_PASSWORD} << EOF
             $(< change_master_to.sql.orig),
               MASTER_HOST='mysql-0.mysql.mysql',
               MASTER_USER='root',
               MASTER_PASSWORD='${MYSQL_ROOT_PASSWORD}',
               MASTER_CONNECT_RETRY=10;
             START SLAVE;
             EOF
             fi

             # 使用 ncat 监听 3307 端口。
             # 它的作用是，在收到传输请求的时候，直接执行 xtrabackup --backup 命令，备份 MySQL 的数据并发送给请求者
             exec ncat --listen --keep-open --send-only --max-conns=1 3307 -c \
               "xtrabackup --backup --slave-info --stream=xbstream --host=127.0.0.1 --user=root --password=${MYSQL_ROOT_PASSWORD}"
           volumeMounts:
           - name: data
             mountPath: /var/lib/mysql
             subPath: mysql
           - name: conf
             mountPath: /etc/mysql/conf.d
   ...
   ```

   还记得之前的实践提到过：

   当创建 Handless Service 之后，它所代理的 Pod 的 IP 地址，都会被绑定一个固定格式的 DNS 记录，如下所示：

   ```
   <pod-name>.<svc-name>.<namespace>.svc.cluster.local
   ```

   所以指定 mysql 主机的 DNS 记录应该是 mysql-0.mysql.mysql。


4. 描述 mysql

   ```yaml
   ...
         - name: mysql
           image: mysql:5.7
           env:
   #        - name: MYSQL_ALLOW_EMPTY_PASSWORD
   #          value: "1"
           - name: MYSQL_ROOT_PASSWORD
             valueFrom:
               secretKeyRef:
                 name: mysql-secret
                 key: password
           ports:
           - name: mysql
             containerPort: 3306
           volumeMounts:
           - name: data
             mountPath: /var/lib/mysql
             subPath: mysql
           - name: conf
             mountPath: /etc/mysql/conf.d
           resources:
             requests:
               cpu: 500m
               memory: 1Gi
           livenessProbe:
             exec:
               command: ["mysqladmin", "ping", "-uroot", "-p${MYSQL_ROOT_PASSWORD}"]
             initialDelaySeconds: 30
             periodSeconds: 10
             timeoutSeconds: 5
           readinessProbe:
             exec:
               # 通过 TCP 连接方法进行健康检查
               command: ["mysqladmin", "ping", "-uroot", "-p${MYSQL_ROOT_PASSWORD}"]
             initialDelaySeconds: 5
             periodSeconds: 2
             timeoutSeconds: 1
   ...
   ```

5. 最终 mysql-statefulset.yaml 文件

   [mysql-statefulset.yaml](mysql-statefulset.yaml) 文件如下：

   ```yaml
   apiVersion: apps/v1
   kind: StatefulSet
   metadata:
     name: mysql
     namespace: mysql
     labels:
       app: mysql
   spec:
     selector:
       matchLabels:
         app: mysql
     serviceName: mysql
     replicas: 3
     template:
       metadata:
         labels:
           app: mysql
       spec:
         initContainers:
         - name: init-mysql
           image: mysql:5.7
           env:
           - name: MYSQL_ROOT_PASSWORD
             valueFrom:
               secretKeyRef:
                 name: mysql-secret
                 key: password
           command:
           - bash
           - "-c"
           - |
             set -ex
             # 从 Pod 的序号，生成 server-id
             [[ $(hostname) =~ -([0-9]+)$ ]] || exit 1
             ordinal=${BASH_REMATCH[1]}
             echo [mysqld] > /mnt/conf.d/server-id.cnf

             # 由于 server-id 不能为 0，因此给 ID 加 100 来避开它
             echo server-id=$((100 + $ordinal)) >> /mnt/conf.d/server-id.cnf

             # 如果 Pod 的序号为 0，说明它是 Master 节点，从 ConfigMap 里把 Master 的配置文件拷贝到 /mnt/conf.d 目录下
             # 否则，拷贝 ConfigMap 里的 Slave 的配置文件
             if [[ ${ordinal} -eq 0 ]]; then
               cp /mnt/config-map/master.cnf /mnt/conf.d
             else
               cp /mnt/config-map/slave.cnf /mnt/conf.d
             fi
           volumeMounts:
           - name: conf
             mountPath: /mnt/conf.d
           - name: config-map
             mountPath: /mnt/config-map
         - name: clone-mysql
           image: gcr.io/google-samples/xtrabackup:1.0
           env:
           - name: MYSQL_ROOT_PASSWORD
             valueFrom:
               secretKeyRef:
                 name: mysql-secret
                 key: password
           command:
           - bash
           - "-c"
           - |
             set -ex
             # 拷贝操作只需要在第一次启动时进行，所以数据已经存在则跳过
             [[ -d /var/lib/mysql/mysql ]] && exit 0

             # Master 节点（序号为 0）不需要这个操作
             [[ $(hostname) =~ -([0-9]+)$ ]] || exit 1
             ordinal=${BASH_REMATCH[1]}
             [[ $ordinal == 0 ]] && exit 0

             # 使用 ncat 指令，远程地从前一个节点拷贝数据到本地
             ncat --recv-only mysql-$(($ordinal-1)).mysql 3307 | xbstream -x -C /var/lib/mysql

             # 执行 --prepare，这样拷贝来的数据就可以用作恢复了
             xtrabackup --prepare --target-dir=/var/lib/mysql
           volumeMounts:
           - name: data
             mountPath: /var/lib/mysql
             subPath: mysql
           - name: conf
             mountPath: /etc/mysql/conf.d
         containers:
         - name: mysql
           image: mysql:5.7
           env:
   #        - name: MYSQL_ALLOW_EMPTY_PASSWORD
   #          value: "1"
           - name: MYSQL_ROOT_PASSWORD
             valueFrom:
               secretKeyRef:
                 name: mysql-secret
                 key: password
           ports:
           - name: mysql
             containerPort: 3306
           volumeMounts:
           - name: data
             mountPath: /var/lib/mysql
             subPath: mysql
           - name: conf
             mountPath: /etc/mysql/conf.d
           resources:
             requests:
               cpu: 500m
               memory: 1Gi
           livenessProbe:
             exec:
               command: ["mysqladmin", "ping", "-uroot", "-p${MYSQL_ROOT_PASSWORD}"]
             initialDelaySeconds: 30
             periodSeconds: 10
             timeoutSeconds: 5
           readinessProbe:
             exec:
               # 通过 TCP 连接方法进行健康检查
               command: ["mysqladmin", "ping", "-uroot", "-p${MYSQL_ROOT_PASSWORD}"]
             initialDelaySeconds: 5
             periodSeconds: 2
             timeoutSeconds: 1
         - name: xtrabackup
           image: gcr.io/google-samples/xtrabackup:1.0
           ports:
           - name: xtrabackup
             containerPort: 3307
           env:
           - name: MYSQL_ROOT_PASSWORD
             valueFrom:
               secretKeyRef:
                 name: mysql-secret
                 key: password
           command:
           - bash
           - "-c"
           - |
             set -ex
             cd /var/lib/mysql

             # 从备份信息文件里读取 MASTER_LOG_FILE 和 MASTER_LOG_POS 这 2 个字段的值，用来拼装集群初始化 SQL
             if [[ -f xtrabackup_slave_info ]]; then
               # 如果 xtrabackup_slave_info 文件存在，说明这个备份数据来自于另一个 Slave 节点
               # 这种情况下，XtraBackup 工具在备份的时候，就已经在这个文件里自动生成了 "CHANGE MASTER TO" SQL 语句
               # 所以，只需要把这个文件重命名为 change_master_to.sql.in，后面直接使用即可
               mv xtrabackup_slave_info change_master_to.sql.in
               # 所以，也就用不着 xtrabackup_binlog_info 了
               rm -f xtrabackup_binlog_info
             elif [[ -f xtrabackup_binlog_info ]]; then
               # 如果只是存在 xtrabackup_binlog_info 文件，说明备份来自于 Master 节点，就需要解析这个备份信息文件，读取所需的两个字段的值
               [[ $(cat xtrabackup_binlog_info) =~ ^(.*?)[[:space:]]+(.*?)$ ]] || exit 1
               rm xtrabackup_binlog_info
               # 把两个字段的值拼装成 SQL，写入 change_master_to.sql.in 文件
               echo "CHANGE MASTER TO MASTER_LOG_FILE='${BASH_REMATCH[1]}',\
                     MASTER_LOG_POS=${BASH_REMATCH[2]}" > change_master_to.sql.in
             fi

             # 如果存在 change_master_to.sql.in，就意味着需要做集群初始化工作
             if [[ -f change_master_to.sql.in ]]; then
               # 但一定要先等 MySQL 容器启动之后才能进行下一步连接 MySQL 的操作
               echo "Waiting for mysqld to be ready（accepting connections）"
               until mysql -h 127.0.0.1 -uroot -p${MYSQL_ROOT_PASSWORD} -e "SELECT 1"; do sleep 1; done

               echo "Initializing replication from clone position"
               # 将文件 change_master_to.sql.in 改个名字
               # 防止这个 Container 重启的时候，因为又找到了 change_master_to.sql.in，从而重复执行一遍初始化流程
               mv change_master_to.sql.in change_master_to.sql.orig
               # 使用 change_master_to.sql.orig 的内容，也就是前面拼装的 SQL，组成一个完整的初始化和启动 Slave 的 SQL 语句
               mysql -h 127.0.0.1 -uroot -p${MYSQL_ROOT_PASSWORD} << EOF
             $(< change_master_to.sql.orig),
               MASTER_HOST='mysql-0.mysql.mysql',
               MASTER_USER='root',
               MASTER_PASSWORD='${MYSQL_ROOT_PASSWORD}',
               MASTER_CONNECT_RETRY=10;
             START SLAVE;
             EOF
             fi

             # 使用 ncat 监听 3307 端口。
             # 它的作用是，在收到传输请求的时候，直接执行 xtrabackup --backup 命令，备份 MySQL 的数据并发送给请求者
             exec ncat --listen --keep-open --send-only --max-conns=1 3307 -c \
               "xtrabackup --backup --slave-info --stream=xbstream --host=127.0.0.1 --user=root --password=${MYSQL_ROOT_PASSWORD}"
           volumeMounts:
           - name: data
             mountPath: /var/lib/mysql
             subPath: mysql
           - name: conf
             mountPath: /etc/mysql/conf.d
         volumes:
         - name: conf
           emptyDir: {}
         - name: config-map
           configMap:
             name: mysql
     volumeClaimTemplates:
     - metadata:
         name: data
       spec:
         accessModes:
         - "ReadWriteOnce"
         storageClassName: block-service
         resources:
           requests:
             storage: 30Gi
   ```

6. 创建 StatefulSet

   ```bash
   $ kubectl create -f storageclass.yaml 
   storageclass.storage.k8s.io/block-service created

   # 查看各个对象是否已经创建
   $ kubectl -n mysql get service,secret,statefulset,pod,pvc -l app=mysql
   NAME                 TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
   service/mysql        ClusterIP   None           <none>        3306/TCP   14m
   service/mysql-read   ClusterIP   10.100.51.22   <none>        3306/TCP   14m

   NAME                  TYPE      DATA      AGE
   secret/mysql-secret   Opaque    1         11m

   NAME                     DESIRED   CURRENT   AGE
   statefulset.apps/mysql   3         3         2m

   NAME          READY     STATUS    RESTARTS   AGE
   pod/mysql-0   2/2       Running   0          2m
   pod/mysql-1   2/2       Running   0          2m
   pod/mysql-2   2/2       Running   0          1m

   NAME                                 STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS    AGE
   persistentvolumeclaim/data-mysql-0   Bound     pvc-bd3ac369-1307-11e9-aff7-0050569f6e43   30Gi       RWO            block-service   2m
   persistentvolumeclaim/data-mysql-1   Bound     pvc-c814b974-1307-11e9-aff7-0050569f6e43   30Gi       RWO            block-service   2m
   persistentvolumeclaim/data-mysql-2   Bound     pvc-dfd7dee8-1307-11e9-aff7-0050569f6e43   30Gi       RWO            block-service   1m
   ```

7. 验证主从状态

   ```bash
   $ kubectl -n mysql exec mysql-1 -c mysql -- bash -c "mysql -uroot -p123456 -e 'show slave status \G'"
   ...
   *************************** 1. row ***************************
                  Slave_IO_State: Waiting for master to send event
                     Master_Host: mysql-0.mysql.mysql
                     Master_User: root
                     Master_Port: 3306
                   Connect_Retry: 10
                 Master_Log_File: mysqllog.000003
             Read_Master_Log_Pos: 154
                  Relay_Log_File: mysql-1-relay-bin.000002
                   Relay_Log_Pos: 319
           Relay_Master_Log_File: mysqllog.000003
                Slave_IO_Running: Yes
               Slave_SQL_Running: Yes
                 Replicate_Do_DB: 
             Replicate_Ignore_DB: mysql
   ...
   ```

### 排错方法

我实践的过程中最大的问题出错最多的在于 xtrabackup 这个容器，所以经常需要看容器内的日志来排错：

```bash
$ kubectl -n mysql logs -f mysql-1 -c xtrabackup
# 在 Slave 节点上一定要看到同步 Master 节点的日志
...
190108 08:01:15 Executing UNLOCK TABLES
190108 08:01:15 All tables unlocked
190108 08:01:15 [00] Streaming ib_buffer_pool to <STDOUT>
190108 08:01:15 [00]        ...done
190108 08:01:15 Backup created in directory '/var/lib/mysql/xtrabackup_backupfiles'
MySQL binlog position: filename 'mysql-bin.000001', position '154'
MySQL slave binlog position: master host 'mysql-0.mysql.mysql', filename 'mysqllog.000003', position '154'
...
```

如果你的 MySQL 集群出错想删掉 StatefulSet 重新再来，一定要删掉对应的 PVC 与 PV。如果不删掉 PVC 与 PV，Slave 节点的 xtrabackup 容器内会判断相应文件已经存在，不会再执行初始化集群操作。

### 验证集群

1. 通过 Headless Service 在 Master 节点上写入数据

   ```bash
   kubectl run mysql-client --image=mysql:5.7 -i -t --rm --restart=Never -- mysql -h mysql-0.mysql.mysql -uroot -p123456 << EOF
   > CREATE DATABASE test;
   > CREATE TABLE test.messages (message VARCHAR(250));
   > INSERT INTO test.messages VALUES ('hello');
   > EOF
   If you don't see a command prompt, try pressing enter.
   pod "mysql-client" deleted
   ```


2. 通过 Normal Service 在所有节点上读取数据

   ```bash
   $ kubectl run mysql-client --image=mysql:5.7 -i -t --rm --restart=Never -- \
   > mysql -h mysql-read.mysql -uroot -p123456 -e "SELECT * FROM test.messages"
   mysql: [Warning] Using a password on the command line interface can be insecure.
   +---------+
   | message |
   +---------+
   | hello   |
   +---------+
   pod "mysql-client" deleted
   ```

### 扩展 Slave 节点

1. 扩展 Salve 节点

   ```bash
   $ kubectl -n mysql scale statefulset mysql --replicas=4
   statefulset.apps/mysql scaled

   $ kubectl -n mysql get pod -l app=mysql
   NAME      READY     STATUS    RESTARTS   AGE
   mysql-0   2/2       Running   0          24m
   mysql-1   2/2       Running   0          24m
   mysql-2   2/2       Running   0          23m
   mysql-3   2/2       Running   0          47s
   ```

2. 验证扩展节点是否正常

   ```bash
   $ kubectl run mysql-client --image=mysql:5.7 -i -t --rm --restart=Never -- \
   > mysql -h mysql-3.mysql.mysql -uroot -p123456 -e "SELECT * FROM test.messages"
   mysql: [Warning] Using a password on the command line interface can be insecure.
   +---------+
   | message |
   +---------+
   | hello   |
   +---------+
   pod "mysql-client" deleted
   ```

---

## 总结

这是 Oracle 官方给出的 [mysql-statefuleset.yaml](https://github.com/oracle/kubernetes-website/blob/master/docs/tasks/run-application/mysql-statefulset.yaml)，实践起来挺费尽的。 尤其是加了增加了 root 密码之后，又遇见了很多新的坑。

首先要感谢一位大佬的支持，我在这位大佬的 Github 提了 Issues 并得到及时的回复帮助我解决了问题。

这位大佬的案例参见：[k8s-example](https://github.com/Mr-Linus/k8s-example/tree/master/mysql-MS-WR)。

总之，在我实践和学习教程的过程中，有几点体会：

1. 「人格分裂」：在解决需求的过程中，不同的 Pod 扮演不同的角色时，就需要使用 StatefulSet 对象来创建这些 Pod，并思考如何去通过 initContainer 或者 Sidecar Container 去协助你的 Pod 完成自己角色需要的任务。

2. 「阅后即焚」：很多「有状态应用」的节点，只是在第一次启动时才需要做额外处理。所以在编写 Yaml 文件时，一定要考虑「容器重启」的情况，不要让这一次的操作干扰到下一次的容器启动。还有，实践的过程中也需要注意这一点，如果你实践的过程中想推翻重来，也记得清除掉数据让重启启动时一定认为是第一次启动才行。

3. 「容器之间平等无序」：除非是 initContainer，否则一个 Pod 之间的多个容器之间，是完全平等的，所以在设计 Sidecar Container 时，一定要控制好每个步骤的顺序，进行前置检查。
