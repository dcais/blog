---
title: unbutu 关闭 GUI
date: 2019-04-23 09:48:49
tags: unbutu bash 运维
---

# 关闭

```bash
sudo systemctl set-default multi-user.target
```

# 开启

```bash
sudo systemctl set-default graphical.target
```
