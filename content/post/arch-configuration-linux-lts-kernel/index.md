+++

title = "Arch Linux 配置 -- 安装linux-lts备用内核"
date = 2023-07-09T14:10:06+08:00
slug = "arch-linux-configuration-linux-lts-kernel"
description = "Arch Linux 滚动更新的特性，让我们总能够体验最新的内核和软件，但有时候也会由于更新导致系统出现故障。本文记录了如何安装linux-lts备用内核，并配置GRUB和rEFInd从而方便地选择不同内核"
tags = ["Linux"]
categories = ["Notes"]
image = "/p/arch-linux-configuration-driver-and-software/arch.webp"

+++

由于滚动更新的机制，Arch Linux采用的linux内核更新较频繁，难免会碰到由于内核更新无法进入系统。为了解决由于内核更新导致的系统故障，我们可以同时安装更新周期更长的linux-lts内核，如果linux内核更新后无法进入系统，此时就可以选择加载linux-lts内核进入系统，回退linux内核或者等待修复。

## 安装linux-lts内核

首先安装linux-lts内核

```bash
sudo pacman linux-lts linux-lts-headers
```

此时重启如果使用GRUB引导系统的话，会出现“Advanced Options for Arch Linxu”，选择这一项就会进入选择启动linux内核或者linux-lts内核的自选单。

如果使用rEFInd引导系统的话，此时选择进入Arch Linux会默认加载最后安装的内核即linux-lts，选中Arch Linux图标在按F2键的话就会进入选择加载不同内核的选单。

为了更加方便地在Boot Loader界面选择想要加载的内核，我们可以分别对GRUB和rEFInd进行一些配置。

## 配置GRUB

我们想要的是在进入GRUB界面后，直接列出了加载不同内核的选项，而不需要先进入子选单再选择内核。编辑`/etc/default/grub`，取消注释

```bash
GRUB_DISABLE_SUBMENU=y
```

如果使用的文件系统不是btrfs，还可以配置GRUB自动选中上一次选择加载的选项，这样可以免除每次重新选择。同样编辑`/etc/default/grub`，对下面几项进行更改

```bash
GRUB_DEFAULT=saved
GRUB_SAVEDEFAULT=true
```

修改完成后重新更新引导配置即可

```bash
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

## 配置rEFInd

rEFInd每次会默认加载最后安装的内核，所以如果不按F2进入子选单选择linux内核就会自动加载linux-lts内核。

通过更改rEFInd配置文件，我们可以实现默认加载进入linux内核，然后通过子选单选择加载linux-lts内核。同时我们也可以选择附加内核参数，使得每次加载内核的同时加载微码，防止出现一些意外奔溃。

为了让rEFInd支持Btrfs子卷，需要为rEFInd安装驱动

```bash
# 我的ESP分区挂载在/efi下
sudo cp /usr/share/refind/drivers_x64/btrfs_x64.efi /efi/EFI/refind/drivers_x64/btrfs_x64.efi
```

编辑rEFInd配置文件`/efi/EFI/refind/refind.conf`，我的ESP分区挂载在`/efi`，如果ESP分区挂载在/boot下的话配置文件路径为`/boot/efi/EFI/refind/refind.conf`。

让rEFInd能够匹配Arch Linux的不同内核

```bash
extra_kernel_version_strings linux-hardened,linux-zen,linux-lts,linux
```

让rEFInd能够在Btrfs子卷下自动扫描内核

```bash
also_scan_dirs boot,ESP2:EFI/linux/kernels,@/boot
```

为了实现rEFInd默认加载linux内核，修改配置文件`/efi/EFI/refind/refind.conf`，手动添加启动项

```bash
# 我使用的是Btrfs文件系统，请更加自己的文件系统和内核位置更改路径选项
menuentry "Arch Linux" {
    icon     /EFI/refind/themes/refind-theme-regular/icons/128-48/os_arch.png
    volume   "arch"  # Arch 文件系统所在卷对应的标签（Label）
    loader   /@/boot/vmlinuz-linux  # 默认加载linux内核
    initrd   /@/boot/initramfs-linux.img  # linux内核对应的initramfs
    # options选项需要更据GRUB配置文件中的内核参数更改
		# options的最后一项设定加载Intel微码，需要确定微码的路径
    options  "root=UUID=d26ed334-68d4-481d-894f-838783fa4f88 rw rootflags=subvol=@ loglevel=5 nowatchdog initrd=@\boot\intel-ucode.img"
		# 子选单定义，用于加载备用initramfs和linux-lts内核
    submenuentry "Boot using fallback initramfs" {
        initrd /@/boot/initramfs-linux-fallback.img  # linux内核的备用initramfs
    }
    submenuentry "Boot to linux-lts" {
    	  loader /@/boot/vmlinuz-linux-lts  # 加载linux-lts内核
        initrd /@/boot/initramfs-linux-lts.img  # linux-lts内核对应的initramfs
    }
    submenuentry "Boot to linux-lts using fallback initramfs" {
    	  loader /@/boot/vmlinuz-linux-lts
        initrd /@/boot/initramfs-linux-lts-fallback.img  # linux-lts的备用initramfs
    }
    submenuentry "Boot to terminal" {
        add_options "systemd.unit=multi-user.target"  # 进入终端
    }
    # disabled  # 注释掉disable才会生效
}
```

设定启动选项时需要在`volume`指定内核和initramfs的所处的子卷，`volume`的值可以设定成`/`或`/boot`所处的子卷标签或者GUID。

查看子卷标签

```bash
❯ sudo lsblk -o name,mountpoint,label,size,uuid
NAME          MOUNTP LABEL         SIZE UUID
nvme0n1                            476.9G
├─nvme0n1p1        SYSTEM_DRV    260M C2B6-5B79
├─nvme0n1p2                      16M
├─nvme0n1p3        Windows-SSD   175.7G 9292B82A92B81527
├─nvme0n1p4        Windows       50G B88EF24F8EF20622
├─nvme0n1p5        WINRE_DRV     1000M D800BC7600BC5CE6
├─nvme0n1p6 /home  arch          239.5G d26ed334-68d4-481d-894f-838783fa4f88
├─nvme0n1p7 [SWAP]               10G d5792b35-68e7-421d-8dde-365e39e6a92b
└─nvme0n1p8 /efi                 512M 6CB0-E267
```

系统启动时会通过Boot Loader加载内核和ramfs，一般还要指定加载微码。内核文件vmlinuz-linux、内核对应的initramfs-linux.img以及微码intel-ucode.img默认位于`/boot`文件夹，在rEFInd配置时需要确定自己系统中这些文件实际存在的路径。

Arch Linux启动过程参考[Arch boot process](https://wiki.archlinux.org/title/Arch_boot_process)。

保存对`/efi/EFI/refind/refind.conf`更改，此时重启进入rEFInd界面，会多出一个我们刚才定义的启动项，直接选择此项启动会直接加载linux内核，按F2进入此启动项的子选单会出现我们刚才定义的加载fallback initramfs和加载linux-lts内核等选项。选中原先的Arch Linux启动项按Delete隐藏即可。
