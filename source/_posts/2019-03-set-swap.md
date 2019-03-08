---
title: linux设置虚拟内存
date: 2019-03-08 09:53:50
tags: [bash,ssh,swap] 
categories: [运维]
---

1. 创建交换文件。

```bash
dd if=/dev/zero of=/opt/swap bs=4096 count=2048000
```

2. 使用以下命令来设置交换文件：

```bash
mkswap /opt/swap
```

3. 设置文件权限

```bash
chmod 0600 /opt/swap
```

4. 启用交换分区文件：
要立即启用交换文件而不是在引导时自动启用，使用以下命令：

```bash
swapon /opt/swap
```

5. 下次启动启用swap。

```bash
vim /etc/fstab
/opt/swap     swap      swap defaults 0 0
```
