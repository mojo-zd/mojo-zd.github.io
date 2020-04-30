---
title: 实用的shell命令
date: 2018-04-20 15:38:30
tags: shell
---

### 从输出table找到指定列
这里以查找k8s下指定的ns为例
 
```
# /bin/bash
for n in $(kubectl get ns | grep ns-team | awk '{print $1}'); do
    echo $n
done

output:
ns-team-1-env-10
ns-team-1-env-11
ns-team-2-env-10
ns-team-2-env-11

说明:
1. kubectl get ns | grep ns-team // 获取所有以ns-team打头的ns列表
2. awk '{print $1}') // 获取前一个命令结果的第一列数据
```
