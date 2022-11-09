---
title: 'innoDB 死锁问题一例--大事务'
date: 2022-09-08 10:56:29
updated: 2022-11-10 00:25:12
draft: true
categories: 
- review
tags: 
- innodb
description: 大事务导致数据库死锁
---

## 背景
调低 delete 任务的频率后，报错大幅减少，
但，还是有 lock 相关报警：
凌晨 3 点。

![](https://yzp-note.4everland.store/img/2022/09/lock-timeout-202209081057197.png)


## 定位
获取锁超时，
`show db engine status`没有对应的错误。
没细看日志，以为还是之前的死锁。

每天都是 3 点这个时间点，应该是有大事务锁住了这个表的资源。

检查一下 定时任务。
发现，还是之前那个：

### 分析一下代码：
- 一个大事务
  - 查询所有的 token
  - 查询所有的 cpc 计划
  - 清除每个计划 60 天之前的快照数据

```java

/**  
 * 清理快照数据  
 *  
 * @param minDatetime 指定时间之前  
 */  
@Transactional(rollbackFor = Exception.class)  
public void clearSnapshotByLaunch(LocalDateTime minDatetime) {  
    
    findAllValidTokenList(); 
     
    for (token : tokens) {  
        List<CpcLaunchConf> confList = findAllByTokenId(token.getId());  
        ...
        for (CpcLaunchConf conf : confList) {  
            Long launchId = conf.getLaunchId();  
            deleteByLaunchIdAndDateTimeBefore(launchId, minDatetime);  
        }  
    }  
}
```
### 原理
这个大事务，会锁住 `所有涉及到的 cpc 计划` 的 `快照数据` 的 `大量记录`。
因为并没有这两个的联合索引：`launchId,dateTime`
已有的索引为：
- `dateTime`
- `launchId,date`

之前的死锁日志中，可以看懂，这里的删除是锁住了 `supremum` 的。
也就是说，会用 `nextkey lock` 锁住大量数据，阻塞读写。
by `launchId` ？
by `dateTime` ? #TODO 

## 解决
思路：
- 缩减事务规模
  - 将`读操作`移出事务
  - 将`写操作事务`拆分成多个`小事务`
- 具体到这个问题，删除任务是每天执行的，且删除范围都是 `60天之前的所有数据`（即：后面的任务范围是覆盖的前面的事务的），所以某一天定时任务失败也不会影响多少，后面一次成功，会将这些数据删除。

```java
@Transactional(rollbackFor = Exception.class)  
public void clearSnapshotByLaunch(LocalDateTime minDatetime) {  
    
    findAllValidTokenList(); 
     
    for (token : tokens) {  
        List<CpcLaunchConf> confList = findAllByTokenId(token.getId());  
        ...
        for (CpcLaunchConf conf : confList) {  
            Long launchId = conf.getLaunchId();  
            cpcRtDataSnapshotRepository.deleteByLaunchIdAndDateTimeBefore(launchId, minDatetime);
        }  
    }  
}

// 修改后
public void clearSnapshotByLaunch(LocalDateTime minDatetime) {  
    
    findAllValidTokenList(); 
     
    for (token : tokens) {  
        List<CpcLaunchConf> confList = findAllByTokenId(token.getId());  
        ...
        for (CpcLaunchConf conf : confList) {  
            Long launchId = conf.getLaunchId();  
            clearCpcTempService.doClear(minDatetime, launchId);
        }  
    }  
}

// 小事务的 trx 注解，也删掉。
// @Transactional(rollbackFor = Exception.class)  
public int doClear(LocalDateTime minDatetime, Long launchId) {  
    return cpcRtDataSnapshotRepository.deleteByLaunchIdAndDateTimeBefore(launchId, minDatetime);  
}

```


终于没有 lock 的告警了：
![](https://yzp-note.4everland.store/img/2022/09/luffy-lock-timeout-202209091011168.png)

