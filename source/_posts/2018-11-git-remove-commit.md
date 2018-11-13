---
title: git删除错误的提交
date: 2018-11-07 20:14:48
tags: [git]
categories: [git]
---

# 方法一

```bash
git reset --hard f9782944b2f80433ead80de6dbe517bd7f9f7974
git push origin HEAD --force
```

# 方法二

回退到上一个版本

```bash
git reset --hard HEAD~1
git push origin HEAD --force
```
