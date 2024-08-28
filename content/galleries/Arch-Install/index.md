---
title: "archlinux安装流程"
date: 2023-09-11T09:09:19+08:00
draft: false
categories:
  - Development
  - linux
tags:
  - archlinux
  - coding
series:
  - "arch从零开始"
---

## 安装archlinux基本环境

参考: [Arch Wiki 如何安装](https://wiki.archlinuxcn.org/wiki/%E5%AE%89%E8%A3%85%E6%8C%87%E5%8D%97)

 <!--more-->

### 一、获取系统镜像

1. 从[阿里云镜像站](http://mirrors.aliyun.com/archlinux/iso/2023.08.01/)下载镜像
2. (可选)在有gpg环境的系统中通过一下命令验证下载的镜像

   ```bash
   gpg --keyserver-options auto-key-retrieve --verify archlinux-version-x86_64.iso.sig
   ```

### 二、安装最小环境

1. 验证引导模式

   ```bash
   ls /sys/firmware/efi/efivars
   ```

2. 验证网络

   ```bash
   ip a            # 查看网络连接
   ping arch.org   # 查看是否连上网
   ```

3. 更新系统时间

   ```bash
   timedatectl     # 更新系统时间
   ```

4. 创建硬盘分区

   ```bash
   fdisk -l        # 查看块设备，结果中以 rom、loop 或者 airoot 结尾的设备可以被忽略

   # 此设置会占满一整个物理设备

   fdisk /dev/the_disk_to_be_partitioned #（要被分区的磁盘）如：/dev/sda
   Command (m for help): g     # 输入g设置分区表为gpt格式

   # EFI分区
   Command (m for help): n     # 创建一个新分区
   # 回车、回车
   Last sector **省略** : +512M # 设置EFI分区大小，单系统设置为512M，要双系统则设置为1G

   # 交换区（可跳过），只在休眠时有用，会一定程度损伤硬盘寿命
   Command (m for help): n     # 创建一个新分区
   # 回车、回车
   Last sector **省略** : +512M # 设置swap分区大小，建议设置为和内存一样大，内存过大则设置为16G较为合适

   # 根分区
   Command (m for help): n     # 创建一个新分区
   # 回车、回车、回车，分配大小也默认全部占满

   # 设置EFI分区格式
   Command (m for help): t     # 更改某个分区格式
   # 输入1回车，输入1回车

   Command (m for help): p     # 检查当前设置分区信息
   Command (m for help): w     # 保存更改

   # 如果你是新创建的EFI分区（按照上面步骤来的），则格式化
   mkfs.fat -F 32 /dev/efi_system_partition（EFI 系统分区）

   # 如果创建了交换区，也需要格式化
   mkswap /dev/swap_partition（交换空间分区）

   # 格式化根分区
   mkfs.ext4 /dev/root_partition（根分区）

   # 挂载对应分区
   mount /dev/root_partition（根分区） /mnt
   mount --mkdir /dev/efi_system_partition（EFI 系统分区） /mnt/boot
   swapon /dev/swap_partition（交换空间分区）
   ```

5. 安装系统

   ```bash
   # 使用reflector更新系统软件源
   reflector --latest 5 --country China --protocol http,https --sort rate --save /etc/pacman.d/mirrorlist

   # 安装系统，intel cpu则多安装一个intel-ucode包，在命令最后加上空格和"intel-ucode"，amd则是amd-ucode
   pacstrap -K /mnt base linux linux-firmware
   # 可在上述命令中追加 vim(文本编辑器)、networkmanager(网络管理)、man-db包，man-pages包 和 texinfo包（帮助文档）。
   ```

6. 配置系统

   ```bash
   # 生成 fstab 文件
   genfstab -U /mnt >> /mnt/etc/fstab
   cat /mnt/etc/fstab  # 检查fstab是否正确

   # chroot 到新安装的系统
   arch-chroot /mnt

   # 设置时区为上海
   ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime

   # 运行hwclock以生成 /etc/adjtime
   hwclock --systohc

   # 区域和本地化设置
   # 编辑 /etc/locale.gen，然后取消掉 en_US.UTF-8 UTF-8 和其他需要的区域（zh_CN UTF-8 UTF-8）设置前的注释（#）
   # 执行 locale-gen 以生成 locale 信息
   locale-gen
   # 创建 locale.conf(5) 文件，并 编辑设定 LANG 变量，比如：
   ehco "LANG=en_US.UTF-8" > /etc/locale.conf
   ```

7. 网络配置

   ```bash
   echo "自定义主机名：如(myarch)" > /etc/hostname
   # 将一下内容写入 /etc/hosts
   127.0.0.1        localhost
   ::1              localhost
   127.0.1.1        主机名.localdomain        主机名
   # 启动前面安装的networkmanager服务
   systemctl enable NetworkManager
   systemctl enable systemd-networkd
   ```

8. 其他工作

   ```bash
   # 设置root密码
   passwd
   # 设置grub引导
   pacman -S grub efibootmgr os-prober   # 安装grub包
   grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB #安装grub引导

   grub-mkconfig -o /boot/grub/grub.cfg    # 生成grub配置

   # 如果之前没有安装文本编辑器，可以安装文本编辑器，本例选用的neovim，还需要安装sudo、openssh
   pacman -S neovim sudo openssh

   # 建立vi、vim->到nvim的软链接（个人习惯）
   ln -s /usr/bin/neovim /usr/bin/vi
   ln -s /usr/bin/neovim /usr/bin/vi

   # 创建非特权用户
   useradd -m -G wheel pathfinder
   passwd pathfinder   # 设置pathfinder密码
   groups pathfinder
   visudo  # 使用visudo 更改wheel组权限，允许使用sudo

   systemctl enable sshd   # 设置openssh开机自启
   ssh-keygen -A           # 生成ssh key
   reboot
   ```

### 三、安装桌面环境

1. 准备工作

   ```bash
   # 配置aur
   sudo vi /etc/pacman.conf
   # 添加
   [archlinuxcn]
   Server = https://mirrors.hit.edu.cn/archlinuxcn/$arch
   # 记得将multilib也取消注释

   # 安装archlinuxcn-keyring
   sudo pacman -Sy && sudo pacman -S archlinuxcn-keyring

   # 安装paru
   sudo pacman -S paru

   # 安装zsh，并设为默认
   paru -S zsh
   chsh -s /usr/bin/zsh

   # 安装英伟达驱动，配置drm
   paru -S nvidia

   # 双系统配置
   mount --mkdir /dev/nvme0n1p1 /mnt/winEFI # 挂载windows系统
   sudo vi /etc/default/grub

   # 在里面设置GRUB_DISABLE_OS_PROBER=false    # 探测win系统
   # GRUB_CMDLINE_LINUX_DEFAULT里添加nvidia_drm.modeset=1  # 设置nvidia drm内核参数

   grub-mkconfig -o /boot/grub/grub.cfg    # 生成grub配置

   # 安装字体
   pacman -S wqy-microhei wqy-zenhei
   ```
