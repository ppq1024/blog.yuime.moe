---
date: 2026-03-22
update: 2026-05-28
name: reinstall-arch
title: 记一次重装 Archlinux 的经历
draft: false
enableKatex: true
enableGitalk: true
tags:
    - Archlinux
---

## 前言

事情的起因是在 Windows 下尝试 [WinBtrfs](https://github.com/maharmstone/btrfs)，看起来一切正常，而且 btrfs 分区可以正常写入。 嗯～～～不对！Archlinux 不是还在 S4 休眠吗，这怎么能写呢？火速重启换系统，Archlinux 可以正常唤醒，似乎一切还在掌握之中，关机换系统继续操作。

不过就在敲下 reboot 后我发现了不对劲，这怎么卡住了呢（事后分析是 grub 尝试读取根分区失败），进入 liveusb 紧急救援发现跟分区超级块没了，一番折腾后备份全盘镜像后重装系统吧。

{{< notice note >}}
这是一个悲惨的故事，告诉我们一定要做好充足的备份。
{{< /notice >}}

{{< notice note >}}
由于本文是事后总结的，可能会存在记忆不准确的地方。
{{< /notice >}}

{{< notice warning >}}
这不是一份 Archlinux 安装教程，流程也不完整，所有命令不建议照抄。
{{< /notice >}}

## 安装 Archlinux

这个其实照着 [Archlinux Wiki](https://wiki.archlinuxcn.org/wiki/安装指南) 安装就行，虽然我是从自己做的 liveusb 安装的，不过基本流程和官方镜像没什么区别。

分区这一步可以直接跳过，之前安装的时候已经分好了，只需要重新格式化主分区，懒得搞太复杂直接 btrfs 默认参数。这里给一下 lsblk 和 mount 的输出作为参考，考虑到大多数用户浏览器没有 nerd font 部分信息就省了（反正也没什么用）：
``` bash
❯ lsblk                                    
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
nvme0n1     259:0    0 953.9G  0 disk 
├─nvme0n1p1 259:1    0   260M  0 part /efi
├─nvme0n1p2 259:2    0    16M  0 part 
├─nvme0n1p3 259:3    0 551.9G  0 part /windows
├─nvme0n1p4 259:4    0   1.5G  0 part 
├─nvme0n1p5 259:5    0   260M  0 part 
├─nvme0n1p6 259:6    0     1G  0 part /boot
└─nvme0n1p7 259:7    0   399G  0 part /home
                                      /snapshots
                                      /
nvme1n1     259:8    0   1.8T  0 disk 
├─nvme1n1p1 259:9    0    16M  0 part 
├─nvme1n1p2 259:10   0   512G  0 part 
├─nvme1n1p3 259:11   0   1.2T  0 part 
├─nvme1n1p4 259:12   0    64G  0 part /home/ppq/Share
└─nvme1n1p5 259:13   0    64G  0 part [SWAP]

❯ mount | grep /dev/nvme
/dev/nvme0n1p7 on / type btrfs (rw,relatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvolid=256,subvol=/@)
/dev/nvme0n1p7 on /snapshots type btrfs (rw,relatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvolid=258,subvol=/@snapshots)
/dev/nvme0n1p7 on /home type btrfs (rw,relatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvolid=257,subvol=/@home)
/dev/nvme0n1p6 on /boot type ext4 (rw,relatime)
/dev/nvme0n1p1 on /efi type vfat (rw,relatime,fmask=0022,dmask=0022,codepage=437,iocharset=ascii,shortname=mixed,utf8,errors=remount-ro)
/dev/nvme1n1p4 on /home/ppq/Share type udf (ro,relatime,uid=1000,gid=1000,umask=26,iocharset=utf8)
/dev/nvme0n1p3 on /windows type fuseblk (ro,nosuid,nodev,relatime,user_id=0,group_id=0,allow_other,blksize=4096)
```

以上是目前的分区情况，挂载 Windows 系统分区主要是为了复制 ssh 密钥和配置，boot 分区是 ext4（懒得改了），Share 主要放一些媒体文件，CD 音源体积还是挺大的，两边各存一份感觉有点浪费（虽然现在是压成 opus 128kbps 了，镜像用的 flac 单独备份），至于交换分区为什么搞那么大，物理内存 40G 总得塞得下吧，毕竟需要 S4 休眠。

这里贴一下安装时额外的软件包：
{{< notice warning >}}
再次警告，这不是一份 Archlinux 安装教程，流程也不完整，所有命令不建议照抄。
{{< /notice >}}
```bash
pacstrap -K ./Arch base linux linux-firmware NetworkManager vim base-devel git man-db man-pages texinfo intel-ucode btrfs-progs grub efibootmgr
```
{{< notice note >}}
命令中的 `./Arch` 为实际的根分区挂载点，对应的绝对路径为 `/root/Arch`，wiki 上的 `/mnt` 我用于挂载 btrfs 根子卷而不是根分区（`@` 子卷）。
{{< /notice >}}

安装配置完成后重启进入系统，后续工作均在系统内进行。

## 安装图形界面

*Linux 为什么需要图形界面呢？当然是可以开很多终端了。*

这里用图形界面而不是桌面环境主要是还包括了窗口管理器（比如我现在用的 Hyprland）。其实我用过的图形界面还是比较少的，之前也就用过 Gnome 和 KDE。Gnome 还是因为早期用 Ubuntu，换成 Archlinux 后一开始用的 KDE，后来才换的 Hyprland（其实中间还浅尝了一些老旧的平铺式 WM，但我忘记叫什么了）。

因为之前的 Hyprland 是从 KDE 转过去的，导致系统中存在大量安装和配置残留，这次正好从头干净安装。需要的包大概就这些：
```bash
❯ pacman -Qt | grep hypr
hypridle 0.1.7-7
hyprland 0.54.2-2
hyprlauncher 0.1.5-2
hyprlock 0.9.2-9
hyprpaper 0.8.3-3
hyprpolkitagent 0.1.3-5
hyprshot 1.3.0-4
```
以及一些额外组件：
- kitty
- waybar
- mako
- sddm
- xdg-desktop-portal
- xdg-desktop-portal-hyprland
- xdg-desktop-portal-gtk

主题和 UI 库相关的包：
- qt5ct
- qt5-wayland
- adwaita-qt5
- qt6ct
- qt6-wayland
- adwaita-qt6
- gtk3
- gnome-theme-extra

{{< notice note >}}
adwaita-qt5 和 adwaita-qt6 在 AUR 中。
{{< /notice >}}

至于那一堆复杂的配置，找时间单独写一篇吧~~咕~~。

前端时间 hyprland 配置还迁移到了 lua，确实值得单独写一下（Hyprland 这更新是真频繁啊，还经常导致旧配置失效）。
