---
title: linux硬盘分区格式化以及挂载
date: 2018-10-23 10:02:43
tags: [harddisk,linux,mount,format]
categories: 运维
---

# 1. 分区

## 1.1 使用fdisk

1）先查看下是否有磁盘没有分区

```bash
fdisk -l
```

2） 分区

```bash
fdisk /dev/sdb
```

3）根据提示操作

### 1.2 使用 parted

## 2. 格式化新硬盘

```bash
mkfs.ext4 /dev/sdb1  
```

## 3. 挂载

1) 创建/data目录（/data目录为硬盘将挂载的地方）：

```bash
mkdir /data  
```

2）挂载分区：

```bash
mount /dev/sdb1 /data  
```

## 4. 查看磁盘分区的UUID

```bash
1. $ sudo blkid  
```

效果如下：

```bash
/dev/sda1: UUID="8048997a-16c9-447b-a209-82e4d380326e" TYPE="ext4"
/dev/sda5: UUID="0c5f073a-ad3f-414f-85c2-4af83f6a437f" TYPE="swap"
/dev/sdb1: UUID="11263962-9715-473f-9421-0b604e895aaa" TYPE="ext4"
/dev/sr0: LABEL="Join Me" TYPE="iso9660" 
```

## 5. 配置开机自动挂载：
因为mount命令会在重启服务器后失效，所以要将分区信息写到/etc/fstab文件中让它永久挂载：

```bash
sudo vim /etc/fstab  
```

加入：

```bash
UUID=11263962-9715-473f-9421-0b604e895aaa /data ext4 defaults 0 1
```

```bash
注：
<fs spec> <fs file> <fs vfstype> <fs mntops> <fs freq> <fs passno>
具体说明，以挂载/dev/sdb1为例:
<fs spec> :
分区定位，可以给UUID或LABEL，例如：UUID=6E9ADAC29ADA85CD或LABEL=software
<fs file> : 具体挂载点的位置，例如：/data
<fs vfstype> : 挂载磁盘类型，linux分区一般为ext4，windows分区一般为ntfs
<fs mntops> : 挂载参数，一般为defaults
<fs freq> : 磁盘检查，默认为0
<fs passno> : 磁盘检查，默认为0,不需要检查 
```

修改完/etc/fstab文件后，运行sudo mount -a命令验证一下配置是否正确。如果配置不正确可能会导致系统无法正常启动。