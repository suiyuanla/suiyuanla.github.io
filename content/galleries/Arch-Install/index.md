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

## 安装 archlinux 基本环境

参考: [Arch Wiki 如何安装](https://wiki.archlinuxcn.org/wiki/%E5%AE%89%E8%A3%85%E6%8C%87%E5%8D%97)

 <!--more-->

### 一、获取系统镜像

1. 从[阿里云镜像站](https://mirrors.aliyun.com/archlinux/iso/)下载镜像
2. (可选)在有 gpg 环境的系统中通过一下命令验证下载的镜像

   ```bash
   gpg --keyserver-options auto-key-retrieve --verify archlinux-version-x86_64.iso.sig
   ```

### 二、安装最小环境

1. 验证引导模式

   ```bash
   # 若有输出则为EFI引导
   ls /sys/firmware/efi/efivars
   ```

2. 验证网络

   ```bash
   ip a            # 查看网络连接
   ping archlinux.org   # 查看是否连上网
   ```

3. 更新系统时间

   ```bash
   timedatectl     # 更新系统时间
   ```

4. 创建硬盘分区

   文件系统和分区可以参考[Arch Wiki 文件系统](https://wiki.archlinuxcn.org/wiki/%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F)以及[Arch Wiki 分区](https://wiki.archlinuxcn.org/wiki/%E5%88%86%E5%8C%BA)。此处给出两种工具的使用教程，`cfdisk`相对于`fdisk`更加易用。

   使用 cfdisk 进行分区（推荐）：

   ```bash
   # 使用lsblk或fdisk查看所有物理设备
   lsblk -p # 或fdisk -l
   # 根据自己设备的容量等信息确定对应分区的路径，nvme是固态硬盘
   cfdisk /path/to/your/device
   # 在命令行图形界面分别建立两个分区
   # 1. 一个EFI分区，大小可以按自己需求设置，我通常设置为512M
   # 2. 一个Linux Filesystem分区，占据设备剩下所有的空间
   # 注意：如果是在已经安装好windows的双系统上安装Archlinux，
   # 则不需要建立EFI分区，同时需要注意不要删除你的windows分区。
   # 创建完成后，记得在cfdisk上write再退出，否则修改不会生效
   ```

   使用 fdisk 进行分区：

   ```bash
   fdisk -l        # 查看块设备，结果中以 rom、loop 或者 airoot 结尾的设备可以被忽略

   # 此设置会占满一整个物理设备

   fdisk /dev/the_disk_to_be_partitioned #（要被分区的磁盘）如：/dev/sda
   Command (m for help): g     # 输入g设置分区表为gpt格式

   # EFI分区
   Command (m for help): n     # 创建一个新分区
   # 回车、回车
   Last sector **省略** : +512M # 设置EFI分区大小，单系统设置为512M，要双系统则设置为1G

   # 交换区（可跳过），不建议在这一步创建，建议在装好系统后使用swap文件，更灵活
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

   ```

   格式化对应分区：

   > 如果使用 Btrfs 文件系统，请参考[Arch Wiki Btrfs](https://wiki.archlinuxcn.org/wiki/Btrfs)。EFI 分区挂载到不同地方的区别参考[Arch Wiki EFI 分区](https://wiki.archlinuxcn.org/wiki/EFI_%E7%B3%BB%E7%BB%9F%E5%88%86%E5%8C%BA)的第 4.1 小节：典型挂载点

   ```bash
   # 如果你是新创建的EFI分区（按照上面步骤来的），则格式化
   mkfs.fat -F 32 /dev/efi_system_partition（EFI 系统分区）

   # 如果创建了交换区，也需要格式化
   mkswap /dev/swap_partition（交换空间分区）

   # 格式化根分区为自己想要的文件系统
   # 格式化根分区，此处使用的是ext4文件系统（建议）
   mkfs.ext4 /dev/root_partition
   # 格式化根分区，此处使用的是btrfs
   mkfs.btrfs -L 自定义标签 /dev/分区名

   # 挂载对应分区
   mount /dev/root_partition（根分区） /mnt
   # 可以将EFI分区挂载到不同的地方，以下示例了两种
   # 如果之前装windows已经有了EFI分区，建议将Windows创建的EFI分区直接挂载到/mnt/efi
   mount --mkdir /dev/efi_system_partition（EFI 系统分区） /mnt/boot
   mount --mkdir /dev/efi_system_partition（EFI 系统分区） /mnt/efi
   # 如果前面创建了swap分区则使用此命令开启swap
   swapon /dev/swap_partition（交换空间分区）
   ```

5. 安装系统

   ```bash
   # 使用reflector更新系统软件源
   reflector --latest 5 --country China --protocol http,https --sort rate --save /etc/pacman.d/mirrorlist

   # 安装系统，intel cpu则多安装一个intel-ucode包，在命令最后加上空格和"intel-ucode"，amd则是"amd-ucode"
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

   # 安装必备软件，此处可以按需调整安装的文本编辑器，本例选用的neovim，还需要安装sudo、openssh
   pacman -S neovim sudo openssh

   # 建立vi、vim->到nvim的软链接（个人习惯）
   ln -s /usr/bin/neovim /usr/bin/vi
   ln -s /usr/bin/neovim /usr/bin/vi

   systemctl enable sshd   # 设置openssh开机自启
   ssh-keygen -A           # 生成本机的ssh key

   # 安装字体
   pacman -S noto-fonts noto-fonts-cjk

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

   # 将以下内容写入 /etc/hosts
   127.0.0.1        localhost
   ::1              localhost
   127.0.1.1        主机名.localdomain        主机名

   # 启用前面安装的networkmanager服务
   systemctl enable NetworkManager

   # 忽略下面这一行
   # systemctl enable systemd-networkd
   ```

8. 用户、ArchlinuxCN 源、引导设置

   ```bash
   # 设置root密码
   passwd

   # 创建非特权用户
   useradd -m -G wheel <your username>
   passwd <your username>   # 设置pathfinder密码
   groups <your username>
   # 使用visudo 更改wheel组权限，允许使用sudo
   visudo

   # 配置archlinuxcn源
   sudo vi /etc/pacman.conf
   # 添加
   [archlinuxcn]
   Server = https://mirrors.ustc.edu.cn/archlinuxcn/$arch
   # 记得将multilib也取消注释

   # 安装archlinuxcn-keyring
   sudo pacman -Sy && sudo pacman -S archlinuxcn-keyring
   # 安装paru，一个aur helper
   sudo pacman -S paru

   # 设置grub引导，注意--efi-directory要根据前面挂载的是/efi还是/boot调整
   # 安装grub包
   pacman -S grub efibootmgr os-prober
   # 安装grub引导
   grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
   # 生成grub配置
   grub-mkconfig -o /boot/grub/grub.cfg
   # 此处就装好了arch，可以重启了
   reboot # 重启
   ```

## 双系统的部分配置

双系统可以参考 Arch Wiki 的[部分内容](https://wiki.archlinuxcn.org/wiki/Arch_%2B_Windows_%E5%8F%8C%E7%B3%BB%E7%BB%9F)。

如果你在前面没有将 windows 系统的 EFI 直接挂载到`/efi`下，而是自己创建了另一个 EFI 分区，为了能让 GRUB 引导 Windows，你需要将 Winodws 的 EFI 挂载到 linux 下：

```bash
mkdir /mnt/winEFI
mount --mkdir /dev/path/to/winefi /mnt/winEFI # 挂载windows系统
```

修改 grub 的配置，并且确认自己安装`os-prober`包，随后重新生成 grub 的配置即可。

```bash
sudo vi /etc/default/grub
# 在里面设置GRUB_DISABLE_OS_PROBER=false    # 探测win系统
# GRUB_CMDLINE_LINUX_DEFAULT里添加nvidia_drm.modeset=1  # 设置nvidia drm内核参数
grub-mkconfig -o /boot/grub/grub.cfg    # 生成grub配置
```

完成后即可在 Grub 引导界面看到 windows 系统。

## 桌面环境的配置

个人喜欢使用 KDE 桌面，具体的细节与问题可以参考[Arch Wiki KDE](https://wiki.archlinuxcn.org/wiki/KDE)

安装 KDE 桌面，同时需要安装一个终端软件，以便在桌面环境打开终端，而不是使用`ctrl+alt+F[1-12]`切换 TTY。个人喜欢使用 [konsole-osc52](https://aur.archlinux.org/packages/konsole-osc52)。

```bash
paru -S plasma-meta konsole-osc52
# 启用sddm服务
systemctl enable --now sddm.service
```

在安装好后，可以安装`firefox`等软件，具体请参阅 wiki 对应界面。可以在设置->语言与地区，更改语言为简体中文，记得在字体里调整为有中文字符的字体，如`Noto Sans Cjk SC`。具体美化或许会再出一篇博客。

在 KDE 里设置语言后，SDDM 界面显示的时间格式与 KDE 锁屏不同，原因是 SDDM 使用的系统的 locale，要更改的话需要进行以下操作：

```bash
sudo systemctl edit display-manager.service

# 在里面添加以下内容
[Service]
Environment="LANG=zh_CN.UTF-8"
```

由于使用的是 KDE Wayland 桌面，因此对于某些软件需要特殊的设置，如 Electron 系应用的原生 Wayland 支持，需要在`~/.config/<应用>-flags.conf`添加以下设置：

```bash
# 注意不同软件的配置文件名不同，如
--enable-features=WaylandWindowDecorations
--ozone-platform-hint=auto
# 此行是fcitx5对electron系软件的输入法支持
--enable-wayland-ime
```

## 输入法配置：

个人使用 Fcitx5，对应 Arch Wiki 页面为[Fcitx5](https://wiki.archlinuxcn.org/wiki/Fcitx5)。

> 注意：本人使用 KDE Waylan 桌面环境，其他桌面环境以及 X11 图形协议并不适用，请参考官方 wiki。

需要安装以下包：

```bash
paru -S fcitx5-im fcitx5-chinese-addons
```

安装好后，在 KDE 系统设置：系统设置 > 输入设备 > 虚拟键盘，选择 Fcitx 5

设置环境变量：

```bash
XMODIFIERS=@im=fcitx
```

对于 Electron 系软件(QQ，Microsoft Edge，VS code)，打字过快漏编码的问题的解决方法：

```bash
# 在 ~/.config/gtk-3.0/settings.ini 中添加下面这行
gtk-im-module=fcitx
```

## Shell 配置和美化：

此处完全是个人习惯，字体喜欢使用`JetBrainsMono Nerd Font`，终端喜欢使用`zsh`，zsh 插件框架喜欢使用[`zimfw`](https://github.com/zimfw/zimfw)，主题喜欢使用[`powerlevel10k`](https://github.com/romkatv/powerlevel10k)

安装框架：

```bash
# 安装一个Nerd fonts
paru -S ttf-jetbrains-mono-nerd
# 安装zsh，并设为默认
paru -S zsh
chsh -s /usr/bin/zsh
# 安装zimfw，参考官网
curl -fsSL https://raw.githubusercontent.com/zimfw/install/master/install.zsh | zsh
```

安装`powerlevel10k`主题：

```bash
# 在 ~/.zimrc 添加下面一行
zmodule romkatv/powerlevel10k --use degit

# 终端运行命令
zimfw install
```
