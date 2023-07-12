+++

title = "Arch Linux 配置 — Btrfs与系统备份"
date = 2023-07-11T14:54:53+08:00
slug = "arch-linux-configuration-btrfs-and-system-backup"
description = "Btrfs的一些高级功能，极大地方便了系统使用和维护，本文介绍了利用Btrfs写时复制、子卷和快照的特性，通过Snapper配置的系统自动备份方案"
tags = ["Linux"]
categories = ["Tech"]
image = "/p/arch-linux-configuration-driver-and-software/arch.webp"

+++

## Btrfs

更据官方文档的说明

> BTRFS is a modern copy on write (COW) filesystem for Linux aimed at implementing advanced features while also focusing on fault tolerance, repair and easy administration.
> 

Btrfs拥有较多传统文件系统如Ext3/4不具有的高级功能，为我们日常文件系统使用和备份带来了便利。

### 写时复制（Copy-on-Write，CoW）

Btrfs默认采用[CoW](https://en.wikipedia.org/wiki/copy-on-write)机制，将文件指针存放在metadata文件中。在对文件进行更改时，Btrfs会将新的数据写在新的空闲数据块中，数据写入完成后，更新metadata增加将文件指向新的内存块的指针。

### 子卷（subvolume）

一个子卷可以看作是Btrfs中的一个文件夹，但是子卷也可以像分区一样被挂载。例如在`subvolume-test`中创建一个子卷`subvolume`，就相当于创建了一个对应的文件夹。

```bash
❯ mkdir subvolume-test && cd subvolume-test
❯ sudo btrfs subvolume create subvolume  # 创建子卷
Create subvolume './subvolume'
❯ ls -l
总计 0
drwxr-xr-x 1 root root 0  7月10日 17:19 subvolume  # 当前目录出现对应的文件夹
❯ sudo btrfs subvolume list -o .  # 查看当前目录下存在的子卷
ID 263 gen 6134 top level 257 path @home/xu/subvolume-test/subvolume
```

如果把子卷挂载到某一目录中，那么该目录内的内容就和该子卷相同了。

```bash
❯ mkdir mount-test  # 创建挂载文件夹
❯ sudo mount -o subvol=@home/xu/subvolume-test/subvolume /dev/nvme0n1p6 mount-test
❯ sudo touch subvolume/test  # 在子卷内新建文件
❯ tree .
.
├── mount-test
│   └── test  # 挂载文件夹内容和子卷相同
└── subvolume
    └── test
3 directories, 2 files
```

Btrfs文件系统中默认会存在一个ID为5的根子卷，用户创建的子卷都是根子卷下的嵌套子卷。

```bash
❯ mkdir rootsub
❯ sudo mount -o subvolid=5 /dev/nvme0n1p6 rootsub  # 挂载根子卷
❯ ls rootsub
@  @home  # 根子卷下包含的子卷
❯ df
文件系统           1K的块      已用      可用 已用% 挂载点
dev               3929540         0   3929540    0% /dev
run               3941308      1480   3939828    1% /run
/dev/nvme0n1p6  251133952  38761288 210874440   16% /
tmpfs             3941308    157696   3783612    5% /dev/shm
/dev/nvme0n1p6  251133952  38761288 210874440   16% /home
tmpfs             3941312     19428   3921884    1% /tmp
/dev/nvme0n1p8     523248      6172    517076    2% /efi
tmpfs              788260        76    788184    1% /run/user/1000
onedriver      1131413504 200908800 930504704   18% /home/xu/OneDrive
/dev/nvme0n1p6  251133952  38761288 210874440   16% /home/xu/rootsub
```

可以看到我的根子卷下创建了`@`子卷和`@home`子卷分别挂载在`/`和`/home`文件夹下。每一个子卷都可以达到文件系统的总容量，使用子卷挂载`/`和`/home`就免除了某一个分区空间不足的烦恼。

### 快照（snapshot）

快照可以看作是一个子卷，对一个子卷创建快照就是创建了一个新的子卷，这个新的子卷内容和原本子卷完全相同，但是快照不会包含子卷内的嵌套子卷。

```bash
❯ mkdir snapshot-test && cd snapshot-test
❯ sudo btrfs subvolume create demo  # 创建子卷demo
Create subvolume './demo'
❯ sudo touch demo/a demo/b
❯ sudo btrfs subvolume snapshot demo demo-snapshot  # 创建子卷的快照
Create a snapshot of 'demo' in './demo-snapshot'
❯ tree
.
├── demo
│   ├── a
│   └── b
└── demo-snapshot  # 快照和子卷具有相同的内容
    ├── a
    └── b

3 directories, 4 files
```

得益于CoW的特性，Btrfs创建快照时并不会将原子卷中的内容复制到快照子卷中，而是在快照中创建了指向原子卷内容的指针。如果对子卷内的文件进行更改，原本内容在数据块中不会改变，只是增加了指向新数据块的指针。

## 使用快照备份系统

得益于Btrfs的快照特性，我们可以快速方便地备份和回滚系统，而且系统快照只会占用很小的储存空间，在备份方案上，我选择使用Snapper创建和管理系统快照。

### Snapper自动备份

首先安装Snapper

```bash
sudo pacman -S snapper
```

为挂载在`/`目录下的子卷创建命名为root的快照配置文件

```bash
sudo snapper -c root create-config /
```

此时Snapper会在`/etc/snapper/configs`生成配置文件，并且在根目录`/`挂载的子卷即`@`下创建`.snapshots`子卷，所有快照都会存储在`.snapshots`子卷下

```bash
❯ sudo btrfs subvolume list /
ID 256 gen 6803 top level 5 path @
ID 257 gen 6803 top level 5 path @home
ID 258 gen 17 top level 256 path var/lib/portables
ID 259 gen 18 top level 256 path var/lib/machines
ID 266 gen 6800 top level 256 path .snapshots  # 挂载在@子卷下
```

Snapper可以自动创建和清除快照，默认会每小时创建一个快照，每天清理一次快照，在清理时会保存10个每小时快照、10个每日快照、10个每月快照、10个每年快照，但是对于日常使用更改较频繁的子卷我们不需要这么频繁的快照。

修改`/lib/systemd/system/snapper-timeline.timer`，设置每6小时创建一次快照

```bash
[Unit]
Description=Timeline of Snapper Snapshots
Documentation=man:snapper(8) man:snapper-configs(5)

[Timer]
OnCalendar=00/3:00  # 每3小时创建一次快照

[Install]
WantedBy=timers.target
```

修改配置文件`/etc/snapper/configs/root`中的变量，改变清理是保留的快照数量

```bash
TIMELINE_MIN_AGE="1800"
TIMELINE_LIMIT_HOURLY="3"  # 保留3个小时快照
TIMELINE_LIMIT_DAILY="7"  # 保留7个每日快照
TIMELINE_LIMIT_WEEKLY="0"
TIMELINE_LIMIT_MONTHLY="0"
TIMELINE_LIMIT_YEARLY="0"
```

开启Snapper自动快照和自动清理

```bash
sudo systemctl enable --now snapper-timeline.timer
sudo systemctl enable --now snapper-cleanup.timer
```

### 改变快照存储位置

Snapper默认把快照存储在`@`子卷内的`.snapshots`分卷下，如果`@`子卷损坏`.snapshots`也可能受影响，所以更好的子卷结构是把快照存储在`@snapshots`子卷下，然后把`@snapshots`挂载在根子卷下，这样`@`和`@snapshots`互不影响。

删除默认的`.snapshots`子卷

```bash
❯ sudo btrfs subvolume list /
ID 256 gen 7144 top level 5 path @
ID 257 gen 7144 top level 5 path @home
ID 258 gen 17 top level 256 path var/lib/portables
ID 259 gen 18 top level 256 path var/lib/machines
ID 266 gen 7115 top level 256 path .snapshots
ID 267 gen 6963 top level 266 path .snapshots/1/snapshot
ID 272 gen 6995 top level 266 path .snapshots/2/snapshot
# 如果已经有快照，需要先删除.snapshots里面的快照子卷
❯ sudo btrfs subvolume delete .snapshots/1/snapshot
Delete subvolume (no-commit): '/.snapshots/1/snapshot'
❯ sudo btrfs subvolume delete .snapshots/2/snapshot
Delete subvolume (no-commit): '/.snapshots/2/snapshot'
❯ sudo btrfs subvolume delete .snapshots
Delete subvolume (no-commit): '//.snapshots'
```

在根子卷下并列`@`子卷创建一个`@snapshots`子卷

```bash
sudo mount -o subvolid=5 /dev/nvme0n1p6 ~/rootsub  # 挂载根子卷
sudo btrfs subvolume create ~/rootsub/@snapshots  # 在根子卷下创建子卷
```

编辑`/etc/fstab`，将`@subvolume`子卷挂载到`./snapshots`

```bash
# 仿照@和@home子卷的挂载更改
UUID=d26ed334-68d4-481d-894f-838783fa4f88  /.snapshots  btrfs  rw,relatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvolid=275,subvol=/@snapshots  0 0
```

挂载@snapshots子卷

```bash
sudo mkdir /.snapshots
sudo mount -a
sudo chmod 750 /.snapshots  # 阻止非root用户访问
```

Btrfs的快照不会备份其他子卷的内容，所以我们也可以把`/var/cache/pacman/pkg`、`/var/abs`、`/var/tmp`和`/srv`等经常读写但有没必要备份的文件夹挂载成子卷。

### 从快照中恢复

假设我们已经为`/`创建了快照，如果想要从快照中恢复系统，可以先进入Arch Linux live USB，然后挂载Btrfs分区

```bash
sudo mount /dev/nvme0n1p6 /mnt
cd /mnt
```

移除旧的`@`子卷

```bash
sudo mv /mnt/@ /mnt/@.broken
```

列出所有快照和对应的时间信息

```bash
$ sudo grep -r '<date>' /mnt/@snapshots/*/info.xml
/.snapshots/1/info.xml:  <date>2023-07-11 06:21:53</date>  # Snapper使用UTC时间记录快照创建时间
/.snapshots/2/info.xml:  <date>2022-07-11 06:22:39</date>
```

由于Snapper创建的快照是只读子卷，所以需要从快照中创建可读写快照作为`@`子卷

```bash
sudo btrfs subvolume snapshot /mnt/@snapshots/{number}/snapshot /mnt/@
```

然后就可以删除旧的`@`子卷

```bash
sudo btrfs subvolume delete /mnt/@.broken
```

## Reference

1. [ArchWiki-Btrfs](https://wiki.archlinux.org/title/Btrfs)
2. [Working with Btrfs – General Concepts](https://fedoramagazine.org/working-with-btrfs-general-concepts/)
3. [Working with Btrfs – Subvolumes](https://fedoramagazine.org/working-with-btrfs-subvolumes/)
4. [Working with Btrfs – Snapshots](https://fedoramagazine.org/working-with-btrfs-snapshots/)
5. [ArchWiki-Snapper](https://wiki.archlinux.org/title/snapper)
6. [BTRFS snapshots and system rollbacks on Arch Linux](https://www.dwarmstrong.org/btrfs-snapshots-rollbacks/)