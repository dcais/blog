---
title: es清除过期的log索引bash脚本
date: 2020-12-28 18:18:56
tags: tips es bash
---

```bash
#!/bin/bash
ES_IP=127.0.0.1
ES_PORT=9200

###################################
#删除早于十天的ES集群的索引
###################################
function delete_indices() {
    comp_date=`date -d "5 day ago" +"%Y-%m-%d"`
    date1="$1 00:00:00"
    date2="$comp_date 00:00:00"

    t1=`date -d "$date1" +%s`
    t2=`date -d "$date2" +%s`

    if [ $t1 -le $t2 ]; then
        echo "$1时间早于$comp_date，进行索引删除"
        #转换一下格式，将类似2017-10-01格式转化为2017.10.01
        format_date=`echo $1| sed 's/-/\./g'`
        curl -XDELETE http://${ES_IP}:${ES_PORT}/*$format_date
    fi
}

curl -XGET http://${ES_IP}:${ES_PORT}/_cat/indices | awk -F" " '{print $3}' | awk -F"-" '{print $NF}' | egrep "[0-9]*\.[0-9]*\.[0-9]*" | sort | uniq  | sed 's/\./-/g' | while read LINE
do
    #调用索引删除函数
    delete_indices $LINE
done
```
