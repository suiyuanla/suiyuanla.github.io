---
title: "Ceph-0: install and use"
date: "2024-08-26T10:10:22+08:00"
draft: true
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

## 普通安装方式（非 K8s 集群）

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

## 集成到 K8s 集群中（使用 Rook）
