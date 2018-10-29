---
title: 利用git credential免密认证git http仓库
date: 2018-10-29 14:32:30
tags: [git,command,node]
categories: [git]
---

# 1. 命令

在git仓库目录下执行

```bash
git config user.name xxx
git config user.email xxx@xxx.com
git config credential.helper store
```

# 2. 缺点

密码将会以明文形式存储在~/.git-credentials文件中，不安全。