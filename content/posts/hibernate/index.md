---
date: 2026-04-03
update: 2026-04-03
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

首先使用分区工具创建新分区，以我目前环境为例为 `/dev/nvme0n1p5`，然后对其进行格式化，然后挂载：
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
```bash
# ~/.config/hypr/hypridle.conf
general {
    lock_cmd = pidof hyprlock || hyprlock
    before_sleep_cmd = loginctl lock-session
}
```
Hyprlock 配置直接使用 [官方示例](https://github.com/hyprwm/hyprlock/blob/main/assets/example.conf)。





