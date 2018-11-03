---
title: 几种用SSH执行远程命令的方法（译）
date: 2018-11-03 13:45:34
tags: [bash,ssh,remote] 
categories: [运维]
---

这篇文章将罗列几种使用SSH远程执行命令的方法。
假设 HOST 参数已经配置好了你的测试服务器信息。

# 单行命令

执行一个单行命令：

```bash
ssh $HOST ls
```

执行多个用;分割的内联命令 (inlined, separated with ;)

```bash
ssh $HOST ls; pwd; cat /path/to/remote/file
```

使用sudo权限执行命令

```bash
ssh $HOST sudo ls /root
sudo: no tty present and no askpass program specified
```

sudo 需要与shell交互, 需要用 -t 参数开启

```bash
ssh -t $HOST sudo ls /root
[sudo] password for zaiste:
```

# 简单的多行命令

```bash
VAR1="Variable 1"
ssh $HOST '
ls
pwd
if true; then
    echo "True"
    echo $VAR1      # <-- it won't work
else
    echo "False"
fi
'
```

shell 变量$VAR1将不会传递到远程命令中

# 可以带变量的多行远程命令

为了能够传递变量，我们使用bash -c 命令

```bash
VAR1="Variable 1"
ssh $HOST bash -c "'
ls
pwd
if true; then
    echo $VAR1
else
    echo "False"
fi
'"
```

# 在远程机器上执行本地脚本

可以简单的用stdin重定向实现

```output
cat script.sh
ls
pwd
hostname
```

```bash
ssh $HOST < script.sh
```

# 使用Heredoc远程执行多行命令

使用**heredoc**可能是最方便的远程执行多行命令的方式了。
而且支持代码块外的变量传递。

```bash
VAR1="boo"
ssh -T $HOST << EOSSH
ls
pwd
if true; then
  echo $VAR1
else
  echo "False"
fi
EOSSH
```

如果需要在heredoc代码块内定义变量，那就在heredoc开始的标记上打上单引号

```bash
ssh -T $HOST <<'EOSSH'
VAR1=`pwd`
echo $VAR1

VAR2=$(uname -a)
echo $VAR2

EOSSH
```

如果出现以下的警告信息

```output
Pseudo-terminal will not be allocated because stdin is not a terminal.
```

可以执行ssh命令的时候加上 -T 参数消除这个警告

# 原文链接

[here](https://zaiste.net/a_few_ways_to_execute_commands_remotely_using_ssh/)