---
date: 2026-04-03
update: 2026-05-28
name: hibernate
title: Archlinux 配置 S4 休眠
draft: false
enableKatex: true
enableGitalk: true
tags:
    - Archlinux
---

前段时间因为意外重装了一下系统，由于 S4 休眠需要额外的配置，大部分发行版默认是不提供的，正好就这次重新配置记录一下。

## 前期准备

{{< notice note >}}
这部分内容在实际配置的过程中并没有执行，而是直接基于旧系统的遗留，复制命令时请注意甄别，如发现有误欢迎通知作者修改。
{{< /notice >}}

S4 休眠是深度最高的一种休眠，会将内存数据写入硬盘并关闭电源，因此休眠过程中不会产生电量消耗。最重要的是，由于操作系统交出了所有硬件的控制权限，在休眠过程中是可以切换系统的，作为双系统玩家，基本是我日常必用的工具。

不同于 Windows 下严格区分休眠文件和交换文件（由于历史原因命名和 Linux 下存在一定差异，分别对应 `hiberfil.sys` 和 `pagefile.sys`），Linux 统一使用交换空间（swap space），且支持独立的交换分区。S4 休眠需要足够的交换空间，一般建议大于物理内存，对于小于物理内存的情况未进行过测试。

```bash
❯ free -h
                总计        已用        空闲        共享      缓冲/缓存        可用
内存：          38Gi       5.9Gi        27Gi       1.0Gi       6.2Gi        32Gi
交换：          64Gi          0B        64Gi
❯ swapon --show
NAME           TYPE      SIZE USED PRIO
/dev/nvme0n1p5 partition  64G   0B   -1
```
因为上次根文件系统炸掉的时候没有影响到交换分区就直接用了，一般情况下需要重新创建或调整大小，以下两种任选其一即可。

**创建交换分区**

首先使用分区工具创建新分区，以我目前环境为例为 `/dev/nvme0n1p5`，然后对其进行格式化并挂载。分区创建以我常用的 `fdisk` 为例：

{{< notice note >}}
具体操作其实我也记不住，但查看帮助信息是一个在linux下的好习惯，这里也贴一下。
{{< /notice >}}

{{< notice warning >}}
尽量不要在有系统休眠时修改分区表。
{{< /notice >}}

```bash
❯ sudo fdisk /dev/nvme0n1

欢迎使用 fdisk (util-linux 2.42.1)。
更改将停留在内存中，直到您决定将更改写入磁盘。
使用写入命令前请三思。

该磁盘目前正在使用 - 不建议您重新分区。
推荐您卸载此磁盘上所有的文件系统，并关闭（swapoff）上面的交换分区。


命令(输入 m 获取帮助)：m

帮助：

  GPT
   M   进入 保护/混合 MBR

  常规
   d   删除分区
   F   列出未分区的空闲区
   l   列出已知分区类型
   n   添加新分区
   p   打印分区表
   t   更改分区类型
   v   检查分区表
   i   打印某个分区的相关信息
   e   调整分区大小
   T   discard (trim) sectors

  杂项
   m   打印此菜单
   x   更多功能(仅限专业人员)

  脚本
   I   从 sfdisk 脚本文件加载磁盘布局
   O   将磁盘布局转储为 sfdisk 脚本文件

  保存并退出
   w   将分区表写入磁盘并退出
   q   退出而不保存更改

  新建空磁盘标签
   g   新建一份 GPT 分区表
   G   新建一份空 GPT (IRIX) 分区表
   o   新建一份的空 DOS（MBR）分区表
   s   新建一份空 Sun 分区表
```

首先确保有足够的剩余空间（为了示例我给旧 Swap 分区删了，~~不过没写入磁盘~~），没有的话可能需要先压缩旧分区。
```bash
命令(输入 m 获取帮助)：F
未分区的空间 /dev/nvme0n1：64 GiB，68722695680 个字节，134224015 个扇区
单元：扇区 / 1 * 512 = 512 字节
扇区大小(逻辑/物理)：512 字节 / 512 字节

      起点       末尾      扇区  大小
3638589440 3638591487      2048    1M
3772807168 3907029134 134221967   64G
```

根据帮助信息使用 `n` 新建分区，参数什么看提示就好。
```bash
命令(输入 m 获取帮助)：n
分区号 (5-128, 默认  5): 
第一个扇区 (3638589440-3907029134, 默认 3772807168): 
最后一个扇区，+/-sectors 或 +size{K,M,G,T,P} (3772807168-3907029134, 默认 3907028991): +64G

创建了一个新分区 5，类型为“Linux filesystem”，大小为 64 GiB。
```

然后修改分区类型为 `Linux swap`，这里直接使用别名，所有类型信息可以通过 `l` 查看（太长就不贴了）。
```bash
命令(输入 m 获取帮助)：t
分区号 (1-5, 默认  5): 
分区类型或别名（输入 L 列出所有类型）：swap

已将分区“Linux filesystem”的类型更改为“Linux swap”。
```

然后执行 `w` 写入分区表，我这边仅示例就 `q` 退出了。

格式化和挂载：
```bash
sudo mkswap /dev/nvme0n1p5
sudo swapon /dev/nvme0n1p5
```

{{< notice note >}}
大部分博客习惯使用 `/dev/sd*` 作为硬盘设备文件，除更简短外也存在一部分历史因素，然而现代计算机中 `/dev/sd*` 多为 SATA 设备，而 NVMe 设备则为 `/dev/nvme*`，交换空间建议置于 SSD 中。
{{< /notice >}}

**创建交换文件**

原则上交换文件可以置于任何位置，但一般直接放在根目录下，其本身作为环回设备，对文件名没有特殊要求，这里以 `/swap.img` 为例，注意按照实际需要调整大小。
```bash
sudo fallocate -l 64G /swap.img
sudo mkswap /swap.img
sudo swapon /swap.img
```

交换分区挂载建议写入 `/etc/fstab`，Archlinux 下也直接使用 `genfstab` 生成。

## 修改 grub 配置

这一步主要是为了告诉内核在启动时尝试从休眠景象中恢复，在 `/etc/default/grub` 中添加以下参数（以交换分区为例）：
```bash
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet resume=UUID=xxxx"
```
`xxxx` 更换为交换分区的 UUID, 可以使用 `blkid` 命令查看。

对于交换文件，需要额外指定文件在分区中的偏移量：
```bash
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet resume=UUID=xxxx resume_offset=xxxx"
```
交换文件在分区中的偏移量可以通过 `filefrag -v` 命令查看：
```bash
❯ filefrag -v temp                         
Filesystem type is: 9123683e
File size of temp is 1048576 (256 blocks of 4096 bytes)
 ext:     logical_offset:        physical_offset: length:   expected: flags:
   0:        0..     255:    4886416..   4886671:    256:             last,unwritten,eof
temp: 1 extent found
```
`physical_offset` 的两个值分别对应起始块和结束块的位置，启动参数中使用起始块的偏移量。

之后重新生成 grub 配置：
```bash
sudo grub-mkconfig -o /boot/grub/grub.cfg
```
进行休眠并重新唤醒确认是否生效：
```bash
sudo systemctl hibernate
```
*~~唤醒正常按开机键就好了（）~~*

## 后续配置

如果不出意外的话，KDE、GNOME 等环境到这里已经可以正常使用了，不过我用的 Hyprland 下还有一点小问题，唤醒后会直接进入已登陆的会话，需要对 Hypridle 及 Hyprlock 进行配置。

在 `~/.config/hypr/hyprland.conf` 中添加以下内容：
```bash
exec-once = hypridle
bind = $mainMod, H, exec, sudo systemctl hibernate
```
Hypridle 的配置文件如下:
```ini
# ~/.config/hypr/hypridle.conf
general {
    lock_cmd = pidof hyprlock || hyprlock
    before_sleep_cmd = loginctl lock-session
}
```
Hyprlock 配置直接使用 [官方示例](https://github.com/hyprwm/hyprlock/blob/main/assets/example.conf)。

### 2026.04.12 更新

前几天配开发环境的时候发现 N 卡驱动没有装，就给补了一下，结果今天唤醒的时候发现失败了，原因是 sddm 需要 nvidia 模块早期加载，但这会导致 resume 失败。折腾半天也没找到之前是怎么配的，然后我意识到一个问题，sddm 需求 nvidia 模块是因为用 N 卡渲染，但我又不在 Arch 上打游戏，能跑 CUDA 就行，渲染交给核显不就好。

配置参数很简单，在 `/etc/modprobe.d` 下创建相关配置文件：
```ini
# /etc/modprobe.d/nvidia.conf
options nvidia_drm modeset=0
```

确认一下显示器输出：
```bash
❯ lspci -vnn | grep VGA
0000:00:02.0 VGA compatible controller [0300]: Intel Corporation Meteor Lake-P [Intel Arc Graphics] [8086:7d55] (rev 08) (prog-if 00 [VGA controller])
0000:01:00.0 VGA compatible controller [0300]: NVIDIA Corporation AD107M [GeForce RTX 4060 Max-Q / Mobile] [10de:28a0] (rev a1) (prog-if 00 [VGA controller])

❯ la /sys/class/drm/card*
lrwxrwxrwx 1 root root 0  4月12日 21:31 /sys/class/drm/card0 -> ../../devices/pci0000:00/0000:00:01.0/0000:01:00.0/drm/card0
lrwxrwxrwx 1 root root 0  4月12日 21:31 /sys/class/drm/card1 -> ../../devices/pci0000:00/0000:00:02.0/drm/card1
lrwxrwxrwx 1 root root 0  4月12日 21:31 /sys/class/drm/card1-DP-1 -> ../../devices/pci0000:00/0000:00:02.0/drm/card1/card1-DP-1
lrwxrwxrwx 1 root root 0  4月12日 21:31 /sys/class/drm/card1-DP-2 -> ../../devices/pci0000:00/0000:00:02.0/drm/card1/card1-DP-2
lrwxrwxrwx 1 root root 0  4月12日 21:31 /sys/class/drm/card1-DP-3 -> ../../devices/pci0000:00/0000:00:02.0/drm/card1/card1-DP-3
lrwxrwxrwx 1 root root 0  4月12日 21:31 /sys/class/drm/card1-DP-4 -> ../../devices/pci0000:00/0000:00:02.0/drm/card1/card1-DP-4
```
所有显示器输出都指向 Intel 核显，与预期一致。

最后确认一下 CUDA 是否正常工作（更多是看 pytorch 是否正常）：
```bash
❯ python -c "import torch; print(f'{torch.cuda.is_available()=}')"
torch.cuda.is_available()=True
```
