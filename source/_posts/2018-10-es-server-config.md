---
title: ElasticSearch服务器配置
date: 2018-10-23 11:14:25
tags: [elasticsearch]
categories: [运维]
---

# 1. 修改/etc/security/limits.conf

/etc/security/limits.conf

```bash
[username] soft nofile 102400
[username] hard nofile 102400
[username] soft nproc 2048
[username] hard nproc 2048
[username] soft memlock unlimited
[username] hard memlock unlimited
```

重新登录后

```bash
ulimit -a 检测
```

# 2. 修改max_map_count

```bash
vim /etc/sysctl.conf
vm.max_map_count=262144
```

执行命令刷新

``` bash
sysctl -p
```