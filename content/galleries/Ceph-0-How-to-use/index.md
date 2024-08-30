---
title: "Ceph-0: install and use"
date: "2024-08-26T10:10:22+08:00"
draft: false
categories:
  - study
tags:
  - paper
series:
  - "Ceph调研"
---

## 参考连接

本文内容参考：[Ceph 官网](https://docs.ceph.com/en/reef)

官网推荐的安装方式是使用[cephadm](https://docs.ceph.com/en/reef/cephadm/install/#cephadm-deploying-new-cluster)

如果要将 Ceph 与 Kubernetes 集成，官网推荐使用[Rook](https://rook.io/)

## 一、普通安装方式（非 K8s 集群）

### 1. 前置要求

系统推荐： 官方目前[推荐](https://docs.ceph.com/en/latest/start/os-recommendations/#platforms)使用 Ubuntu 20.04

系统内软件需求：

- Python3
- Systemd
- Podman or Docker (用于运行容器)

  > 安装 docker 请参考[官网教程](https://docs.docker.com/engine/install)，若遇到网络问题，请参考官网的手动安装教程。

- 时间同步（如 chrony）
- lvm2(提供存储)
- openssh

```bash
# 查看是否安装相应的软件
# python version
python3 --version
# systemd version
systemctl --version
# docker version, 安装教程请参考docker官网
docker version
# chrony version, 安装使用"sudo apt install chrony"命令
chronyc -v
sudo systemctl status chronyd.service # 查看chrony的service
# lvm2
dpkg -l | grep lvm2 # 查看是否安装lvm2
# 若没有安装则安装
sudo apt update && sudo apt install -y lvm2

# cephadm 基于发行版的安装方法
sudo apt install -y cephadm
# 检查是否安装成功
sudo which cephadm
```

### 2. ceph 初始化

Ceph 集群由**monitor**节点管理，**monitor**节点可以有多个，官方推荐大多数情况下 5 个节点是比较合适的，超过 7 个的**monitor**节点没有额外好处。

> 参考链接：[Running the bootstrap command](https://docs.ceph.com/en/reef/cephadm/install/#running-the-bootstrap-command)

```bash
cephadm bootstrap --mon-ip *<mon-ip>* # mon-ip 就是monitor的ip
```

上述命令做了下列事情：

1. 在本地主机上为新集群创建**monitor**和**Manager daemon**。
2. 为集群生成新的 ssh key 并将其添加到 root 用户的`/root/.ssh/authorized_keys`中。（ceph 的运行和管理要求使用无须密码的 root 权限，所以直接用 root 最方便，可以使用--ssh-user 指定为非 root 用户）
3. 将集群的 ssh key 的公钥写入`/etc/ceph/ceph.pub`。
4. 将`client.admin`管理员密钥的副本写入`/etc/ceph/ceph.client.admin.keyring`。
5. 将`_admin`标签添加到运行命令的主机。默认情况下，具有此标签的主机都将获得`/etc/ceph/ceph.conf`以及`/etc/ceph/ceph.client.admin.keyring`的副本。

此步完成以后，根据命令行的提示可以在网页访问`<mon_ip>:8443`，输入命令行显示的用户名和密码即可登录查看。

尝试使用 ceph cli:

```bash
# 进入ceph daemon 容器
sudo cephadm shell
# 查看服务以及状态
ceph orch ls
ceph orch status
```

补充，若想直接使用 ceph 命令而不进入容器，则执行下列命令：

```bash
cephadm add-repo --release reef
cephadm install ceph-common
# 验证是否安装好
stat /sbin/mount.ceph
```

### 3. 添加节点

添加的节点不一定需要安装`cephadm`，但需要满足[前置要求](#1-前置要求)。

在满足要求后，在已经部署好的**monitor**节点上执行命令：

```bash
# 此命令实际上是将ceph.pub复制到了指定ip的用户目录的authorized_keys里
# 同Bootstrap中一样，user必须具有免密的root权限，而且必须具有ssh访问<usernam@ip>的权限
ssh-copy-id -f -i /etc/ceph/ceph.pub uername@ip
```

在设置好后，在**monitor**的 daemon 容器里执行：

```bash
# 添加指定主机，可以追加"--labels=_admin"等添加标签
ceph orch host add <hostname> <hostip>
# 给指定主机追加label
ceph orch host label add <host> <label>
```

有一些特殊的标签：

- \_no_schedule：不在该节点上调度或部署守护进程
- \_no_conf_keyring：不要在此节点上部署配置文件或密钥环
- \_no_autotune_memory：不要自动调整该节点上守护进程占用的内存
- \_admin：将 client.admin 和 ceph.conf 分发到该节点（如果需要在某节点上访问 ceph cli，则需要把这个标签加上）

### 4. 删除节点

可以通过下列操作删除加入的节点：

```bash
# 移除指定主机上的OSD
ceph orch host drain <host>
# 从集群中删除主机，如果host上有正在使用的osd，则删除会失败
ceph orch host rm <host> --rm-crush-entry
```

#### 5. 添加存储

集群节点上需要有未使用的块设备，如在 pve 下新添加的设备，设备的最小容量是 5G。

```bash
# 让ceph自动使用任何可用和未使用的设备
ceph orch apply osd --all-available-devices
# 将指定设备作为OSD
ceph orch daemon add osd <host>:<device-path>
```

有了 OSD 后，需要在 OSD 上提供 cephfs：

```bash
ceph fs volume create <fs_name> --placement="<placement spec>"
# 一个例子，将名字为foo的fs放置在具有mds标签的host上
ceph fs volume create foo --placement="label:mds"
```

也可以通过`ceph orch apply -i <mds>.yaml`部署：

```YAML
service_type: mds
service_id: fs_name
placement:
  count: 3
  label: mds
```

### 6. 在客户机使用 ceph

此部分参考：[Mount Cephfs: Prerequisites](https://docs.ceph.com/en/reef/cephfs/mount-prerequisites/)

首先，在要使用 ceph 的客户机上运行以下命令：

```bash
mkdir -p -m /etc/ceph
# 下面这个命令实际是在monitor上生成了集群的最小配置信息，
# 并将其复制进了指定路径，可以手动操作这步
ssh <user>@<mon-host> "sudo ceph config generate-minimal-conf" | sudo tee /etc/ceph/ceph.conf
# 调整权限
chmod 644 /etc/ceph/ceph.conf
# 创建一个用户用于访问指定的cephfs，并且将keyring复制到客户机，可以手动执行
ssh <user>@<mon-host> "sudo ceph fs authorize <cephfs_name> client.<ceph_user> / rw" | sudo tee /etc/ceph/ceph.client.<ceph_user>.keyring
# 更改权限
chmod 600 /etc/ceph/ceph.client.foo.keyring
```

客户机使用 ceph 需要安装 cephadm。在安装好 cephadm 后，使用下列命令安装需要的工具：

```bash
cephadm add-repo --release reef
cephadm install ceph-common
# 验证是否安装好
stat /sbin/mount.ceph
```

安装好后使用下列命令，mount 指定的 cephfs，即可使用：

```bash
# mount -t ceph 192.168.20.81:6789:/ /mnt/cephs -o name=test,fs=foo
# 由于已经配置好/etc/ceph/ceph.conf以及keyring了，因此secret和mon_addr等选项可以不用设置
mount -t ceph :/ /mnt/cephs -o name=test,fs=foo
```

### 7. 其他命令

1. 将主机置于维护模式或退出维护模式（停止主机上的所有 Ceph 守护进程）

```bash
# --force会忽略warnning，但无法忽略alerts
# --yes-i-really-mean-it会跳过所有安全检查，并且强制使主机进入维护模式（可能导致数据丢失）
ceph orch host maintenance enter <hostname> [--force] [--yes-i-really-mean-it]
ceph orch host maintenance exit <hostname>
```

2. 某些极其可能不会向内核注册设备的移除或插入事件，此时可以在这些设备上运行`rescan`。

```bash
ceph orch host rescan <hostname> [--with-summary]
```

3. 可以使用 `ceph orch apply -i <file.yaml>` 通过**yaml**文件一次性添加多个 host：

> 详细配置参考：[Ceph Service Specification](https://docs.ceph.com/en/reef/cephadm/services/#orchestrator-cli-service-spec)

```yaml
# yaml 文件示例
service_type: host
hostname: node-00
addr: 192.168.0.10
labels:
  - example1
  - example2
---
service_type: host
hostname: node-01
addr: 192.168.0.11
labels:
  - grafana
---
service_type: host
hostname: node-02
addr: 192.168.0.12
```

## 二、集成到 K8s 集群中（使用 Rook）

> 参考资料：[Rook 官网](https://rook.io)

### 1. 前置要求

> 请参考官网的[前置要求](https://rook.io/docs/rook/latest-release/Getting-Started/Prerequisites/prerequisites/#kubernetes-version)

- 现有的 k8s 集群， k8s 版本要求为 v1.26-v1.31 (截止 2024-08-29)
- 使用 [helm](https://helm.sh) 安装，因此需要在 master 上安装 helm
- 节点 CPU 架构只支持：amd64/x86_64 和 arm64
- ceph 需要节点上具有以下类型的本地存储设备来提供存储：

  - Raw devices (原始设备)
  - Raw partitions (原始分区)
  - LVM Logical Volumes (no formatted filesystem) (无格式化的 LVM 逻辑卷)
  - Persistent Volumes available from a storage class in block mode (block 模式下 storage class 里可用的持久卷)

- 某些情况下 ceph 会使用到`lvm`工具，可以使用`sudo apt install lvm2`安装（具体参考官网的说明，安装命令因发行版而异）

### 2. 在 k8s 集群部署 Rook

Rook 官网提供了[使用 helm](https://rook.io/docs/rook/latest-release/Helm-Charts/operator-chart) 的安装方式:

```bash
# 可以克隆下它的仓库，要使用到部分示例yaml文件
git clone --single-branch --branch v1.15.0 https://github.com/rook/rook.git
# value.yaml 在 deploy/charts 路径下
helm repo add rook-release https://charts.rook.io/release
helm install --create-namespace --namespace rook-ceph rook-ceph rook-release/rook-ceph -f values.yaml

# 删除方式
# 查看
helm ls --namespace rook-ceph
# 删除
helm delete --namespace rook-ceph rook-ceph
```

### 3. 创建 ceph 集群

使用 helm 部署好 rook-operator 后，需要手动创建一个 ceph 集群。其实是通过`kubectl create -f`传入一个 yaml，`rook-operator`会自动将 k8s 集群节点加入到 ceph 集群。

```bash
# 默认的cluster.yaml在rook git仓库的 deploy/examples 路径下
kubectl create -f cluster.yaml
# 使用下列命令查看集群内的pod情况
 kubectl -n rook-ceph get pod
```

### 4. 使用 rook-toolbox 调试集群

rook-toolbox 是一个工具箱 pod，在里面可以使用 ceph 命令查看集群信息，进行调试：

```bash
# 示例yaml在 deploy/examples/toolbox.yaml 可直接使用
kubectl create -f deploy/examples/toolbox.yaml
# 可以使用下列命令进入pod，进行测试
kubectl -n rook-ceph exec -it <podname> -- bash
# 常用命令：
# ceph status
# ceph osd status
# ceph df
# ceph fs volume info <fsname>
# ceph fs volume ls
```

### 5. 创建 cephfs

可以使用下列 yaml 创建一个 cephfs，用于提供存储：

```yaml
# filesystem.yaml
apiVersion: ceph.rook.io/v1
kind: CephFilesystem
metadata:
  name: myfs # 此处的name是自定义的
  namespace: rook-ceph
spec:
  metadataPool:
    replicated:
      size: 3
  dataPools:
    - name: replicated
      replicated:
        size: 3
  preserveFilesystemOnDelete: true
  metadataServer:
    activeCount: 1
```

使用上面的 `filesystem.yaml` 创建一个文件系统

```bash
kubectl create -f filesystem.yaml
# 查看创建的文件系统
kubectl -n rook-ceph get CephFilesystem
# 在rook-toolbox内查看
ceph fs ls
ceph fs volume ls
ceph fs volume info myfs # myfs 是上面yaml中设置的名称
```

### 6. 制备静态 pv

> 此部分参考[官网](https://rook.io/docs/rook/latest-release/Storage-Configuration/Shared-Filesystem-CephFS/filesystem-storage)
> 首先需要创建一个`StorageClass`，这是 k8s 与 CSI 驱动互操作所需的：

```yaml
# deploy/examples/csi/cephfs/storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-cephfs
# Change "rook-ceph" provisioner prefix to match the operator namespace if needed
provisioner: rook-ceph.cephfs.csi.ceph.com
parameters:
  # clusterID is the namespace where the rook cluster is running
  # If you change this namespace, also change the namespace below where the secret namespaces are defined
  clusterID: rook-ceph

  # CephFS filesystem name into which the volume shall be created
  fsName: myfs # myfs是上面创建的filesystem的名字

  # Ceph pool into which the volume shall be created
  # Required for provisionVolume: "true"
  pool: myfs-replicated

  # The secrets contain Ceph admin credentials. These are generated automatically by the operator
  # in the same namespace as the cluster.
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-cephfs-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/controller-expand-secret-name: rook-csi-cephfs-provisioner
  csi.storage.k8s.io/controller-expand-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-cephfs-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph

reclaimPolicy: Delete
```

> 动态制备的[官网例子](https://rook.io/docs/rook/latest-release/Storage-Configuration/Shared-Filesystem-CephFS/filesystem-storage/#consume-the-shared-filesystem-k8s-registry-sample)

由于项目中主要使用静态制备 pv 的方式来共享数据，且制备方式较为复杂，因此重点介绍：

1. 复制`rook-csi-cephfs-nod`

   - 使用`kubectl -n rook-ceph get secret`可以看到有一个名为`rook-csi-cephfs-nod`的 secret。
   - 使用`kubectl -n rook-ceph get secret -o yaml > <filename>`命令将其输出为 yaml 文件。
   - 将其中的`adminID`和`adminKey`分别改为`userID`和`userKey`，secret 的名称改为`rook-csi-cephfs-nod-user`。
   - 添加此 secret`kubectl apply -f <filename>`

2. 先使用下面的 yaml 文件创建一个 PVC：

```yaml
# pvc.yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: my-pvc # 名字自行更改
  namespace: first-namespace # 如果都是默认的话，此处应该为rook-ceph
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 100Gi # 容量按需调整
  storageClassName: rook-cephfs
  volumeMode: Filesystem
```

使用`kubectl get pvc -n rook-ceph`命令可以查看到新创建的 pvc。系统会动态制备一个 pv 给这个 pvc 使用，可以使用`kubectl get pv`查看到。

使用`kubectl get pv <pvname> -o yaml > <filename>`获取到 pv 的配置信息(例子来源于官网)：

```yaml
# pv.yaml
kind: PersistentVolume
apiVersion: v1
metadata:
  name: pvc-a02dd277-cb26-4c1e-9434-478ebc321e22
  annotations:
    pv.kubernetes.io/provisioned-by: rook.cephfs.csi.ceph.com
  finalizers:
    - kubernetes.io/pv-protection
spec:
  capacity:
    storage: 100Gi
  csi:
    driver: rook.cephfs.csi.ceph.com
    volumeHandle: >-
      0001-0011-rook-0000000000000001-8a528de0-e274-11ec-b069-0a580a800213
    volumeAttributes:
      clusterID: rook
      fsName: rook-cephfilesystem
      storage.kubernetes.io/csiProvisionerIdentity: 1654174264855-8081-rook.cephfs.csi.ceph.com
      subvolumeName: csi-vol-8a528de0-e274-11ec-b069-0a580a800213
      subvolumePath: >-
        /volumes/csi/csi-vol-8a528de0-e274-11ec-b069-0a580a800213/da98fb83-fff3-485a-a0a9-57c227cb67ec
    nodeStageSecretRef:
      name: rook-csi-cephfs-node
      namespace: rook
    controllerExpandSecretRef:
      name: rook-csi-cephfs-provisioner
      namespace: rook
  accessModes:
    - ReadWriteMany
  claimRef:
    kind: PersistentVolumeClaim
    namespace: first-namespace
    name: base-pvc
    apiVersion: v1
    resourceVersion: "49728"
  persistentVolumeReclaimPolicy: Retain
  storageClassName: rook-cephfs
  volumeMode: Filesystem
```

3. 将其按下面的例子修改，注意删除原来导出的 yaml 中不必要的配置：

```yaml
kind: PersistentVolume
apiVersion: v1
metadata:
  # name 为原pv name+<你要用的新namespace>，其实没有规定，此处为官网建议，示例的新namespace为“newnamespace”
  name: pvc-a02dd277-cb26-4c1e-9434-478ebc321e22-newnamespace
spec:
  capacity:
    # 容量和原pv保持一致
    storage: 100Gi
  csi:
    driver: rook.cephfs.csi.ceph.com
    # 原volumeHanle后+<你要用的新namespace>
    volumeHandle: 0001-0011-rook-0000000000000001-8a528de0-e274-11ec-b069-0a580a800213-newnamespace
    volumeAttributes:
      clusterID: rook
      fsName: rook-cephfilesystem
      storage.kubernetes.io/csiProvisionerIdentity: 1654174264855-8081-rook.cephfs.csi.ceph.com
      subvolumeName: csi-vol-8a528de0-e274-11ec-b069-0a580a800213
      subvolumePath: /volumes/csi/csi-vol-8a528de0-e274-11ec-b069-0a580a800213/da98fb83-fff3-485a-a0a9-57c227cb67ec
      # 添加 rootPath，值和subvolumePath一样
      rootPath: /volumes/csi/csi-vol-8a528de0-e274-11ec-b069-0a580a800213/da98fb83-fff3-485a-a0a9-57c227cb67ec
      # 添加staticVolume
      staticVolume: "true"
    nodeStageSecretRef:
      name: rook-csi-cephfs-node
      namespace: rook
  accessModes:
    - ReadWriteMany
  # persistentVolumeReclaimPolicy改为Retain
  persistentVolumeReclaimPolicy: Retain
  storageClassName: rook-cephfs
  volumeMode: Filesystem
```

4. 有了新的 pv，就可以在其他命名空间中创建一个新的 pvc，让这个 pvc 使用我们制备的 pv：

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: second-pvc # 新pvc的名字
  namespace: newnamespace # 新pvc所在的命名空间
  finalizers:
    - kubernetes.io/pvc-protection
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 100Gi # 注意容量应小于等于制备的pv
  volumeName: pvc-a02dd277-cb26-4c1e-9434-478ebc321e22-newnamespace # 我们制备的新pv的名字
  storageClassName: rook-cephfs
  volumeMode: Filesystem
```

5. 使用新的 pv，尝试使用[busybox-pod]测试新的 pvc：

```yaml
# busybox.yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox
  namespace: default # 新pvc所在的命名空间
spec:
  containers:
    - image: busybox
      command:
        - sleep
        - "3600"
      imagePullPolicy: IfNotPresent
      name: busybox
      volumeMounts:
        - name: test-volume
          mountPath: /test # 把volume挂载的路径
  volumes:
    - name: test-volume # volume的名字
      persistentVolumeClaim:
        claimName: second-pvc # 我们制备的pvc的名字
        readOnly: false
  restartPolicy: Always
```

随后使用`kubectl apply -f <filename>`启动这个 pod，使用`kubectl -n <newnamespace> exec -it <podname> -- sh` 进入这个 pod。

可以使用`dd if=/dev/zero of=./testfile bs=1M count=1024`快速生成一个 1G 大小的文件。然后在 rook-toolbox 里使用`ceph fs volume info myfs`或`ceph status`查看存储使用的情况。
