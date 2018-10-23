---
title: 利用bash制作守护进程的脚本
date: 2018-10-23 11:33:40
tags:
---

# 运用场景

在linux deploy服务时，为了保证服务crash以后能够自动重启，经常需要制作守护进程的脚本。

## 1. 记录进程的PID

为了得到准确的进程PID，我们经常在启动脚本中输出一个xxx.pid文件，其中记录了需要守护的进程的PID

我们可以利用bash变量\!\$ 获取Shell最后运行的后台Process的PID

Example:

```bash
#!/bin/bash
java -jar myapp.jar & echo $! > ./pid.file &
```

这样我们便得到了 myapp.jar的进程PID,并写入了 ./pid.file文件中

## 2. 监测进程是否在运行

利用 kill -0 检测进程是否存在
kill -0 \$pid中的-0表示不发送任何信号给PID对应的进程，但是仍会对变量值PID对应的进程是否存在进行检查，如果\$pid对应的进程存在，则返回0，不存在返回1。

```bash
PID=$(cat ./pid.file) > /dev/null 2>&1
kill -0 ${SAUNA_PID} > /dev/null 2>&1
IS_RUNNING=$?
```

## 3. 完整脚本

start.sh

```bash
#!/bin/bash
PID_FILE=${HOME}/pids/app.pid
WATCH_PID_FILE=${HOME}/pids/watch.pid

#进程PID输出到文件
java -jar myapp.jar & echo $! > ${PID_FILE} 2>/dev/null &
sleep 5

sh watch.sh > watch.log 2>&1 &

echo $! > ${PID_FILE}


```

watch.sh

```bash
#!/bin/bash
PID_FILE=${HOME}/pids/app.pid

if [ -f "$PID_FILE" ]
then
#如果PID文件存在
  PID=$(cat ${SAUNA_PID_FILE}) > /dev/null 2>&1
  kill -0 ${PID} > /dev/null 2>&1
  IS_RUNNING=$?
else
  PID="0000"
  IS_RUNNING=1
fi

#检测循环
while true
do
  # 5 秒检测一次
  sleep 5
  if [ ${IS_RUNNING} -ne 0 ] ; then
    echo "service is dead. restarting...";
    sh start.sh
    exit 0;
  fi
  kill -0 ${PID} > /dev/null 2>&1
  IS_RUNNING=$?
done

```

**注意** stop脚本里要kill掉watch.sh ，不然会重复启动。