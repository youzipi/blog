---
title: 'pulsar broker load balance'
date: 2022-12-27 20:05:05
updated: 2022-12-27 20:05:05
categories: 
- MQ
tags: 
- pulsar
- broker
- load-balance
description: Pulsar broker load balance 是如何设计的
draft: false
---

> Pulsar broker load balance 是如何设计的
<!--more-->


## 原理

### bundle

将 所有的 topic（partitioned topic) 分到 bundle 中，作为负载的单元，进行管理（split, unload-load）。

类比：mysql extent

#### consistent hash

![](https://yzp-note.4everland.store/img/pulsar/lb/lb/bundle-hash-ring.png)

![](https://yzp-note.4everland.store/img/pulsar/lb/lb/bundle-level.png)

### lookup

图中，描述了 client 启动时，为本地 topic 对象 绑定 broker url 的过程：
- 如果有 topic-bundle-broker 的关系，就返回。
- 如果没有，leader broker 从所有 broker 中挑选出一个，assign 目标 bundle，然后返回这个 broker。

![](https://yzp-note.4everland.store/img/2022/12/202212051958626.png)


### shedding strategies

#### OverloadShedder
![](https://pulsar.apache.org/assets/images/OverloadShedder-b981b25c664212a44e650597158c6ced.png)
> 每个节点的负载不超过 指定阈值

缺点：
如果 阈值=90，broker 当前负载情况= `(80,0,0)`。此时，不会进行负载均衡。

#### ThresholdShedder
![](https://pulsar.apache.org/assets/images/ThresholdShedder-6973b45c5703a3d00b78330729e24757.png)
为了解决 `OverloadShedder` 的缺点，出现了 ThresholdShedder
> 每个节点的负载不超过 平均负载 + 指定阈值
- 计算平均负载
- 比较 `current/historical` 和 `avg+loadBalancerBrokerThresholdShedderPercentage`
- offload 流量 = `cur - avg - threshold + 5%`

#### UniformLoadShedder
![](https://pulsar.apache.org/assets/images/UniformLoadShedder-78dfc2001f20dbbb600cdf6baaebb09b.png)
> 每个节点的负载不超过 `(max-min)/min - 指定阈值`，
> 如果超过，unload `(max-min)*0.2` 的流量

前面两种配置 基于 `系统资源使用量`，
UniformLoadShedder 基于 `msgRate, msgThrought`。

```js
maxUnloadPercentage = 0.2
// rate: max 超过 min 50%
// throughput: max 达到 min 的 4 倍
// 两个条件，满足任何一个，触发 unload
loadBalancerMsgRateDifferenceShedderThreshold=50
loadBalancerMsgThroughputMultiplierDifferenceShedderThreshold=4

最小 unload 阈值 = 1 MB | 1000 条
minUnloadMessageThroughput
minUnloadMessage

最大 unload 阈值 = 20%
minUnloadMessageThroughput
```


### 如何定义，收集 broker 的负载？
![](https://yzp-note.4everland.store/img/2022/12/rRazUSwzfI.jpg)

#### 负载上报 配置
```conf
# Percentage of change to trigger load report update
loadBalancerReportUpdateThresholdPercentage=10
# minimum interval to update load report
loadBalancerReportUpdateMinIntervalMillis=5000
# maximum interval to update load report
loadBalancerReportUpdateMaxIntervalMinutes=15
# Frequency of report to collect
loadBalancerHostUsageCheckIntervalMinutes=1
# Interval to flush dynamic resource quota to ZooKeeper
loadBalancerResourceQuotaUpdateIntervalMinutes=15
```

## 演示
### pulsar admin
https://docs.streamnative.io/pulsarctl/v2.9.2.3/#namespaces

```bash
# 看一下 3 个节点的 bundle 分布

# 查看 指定 broker 的 bundle 分布
pulsarctl brokers namespaces zaihui-pulsar --url ${ip}:8080 | jq
> 
{
  "public/default/0x20000000_0x30000000": {
    "broker_assignment": "shared",
    "is_controlled": false,
    "is_active": true
  },
  ...
}
# 查看 指定 namespaces 的 bundle 信息
pulsarctl namespaces policies public/default | jq
> 
{
  "bundles": {
    "boundaries": [
      "0x00000000",
      "0x10000000",
      "0x20000000",
      "0x30000000",
      "0x40000000",
      "0x50000000",
      "0x60000000",
      "0x70000000",
      "0x80000000",
      "0x90000000",
      "0xa0000000",
      "0xb0000000",
      "0xc0000000",
      "0xd0000000",
      "0xe0000000",
      "0xf0000000",
      "0xffffffff"
    ],
    "numBundles": 16
  },
...
}
# 查看 指定 topic 的 bundle
pulsarctl topics bundle-range smart.normal.download.topic.prod
>
The bundle range of the topic persistent://public/default/smart.normal.download.topic.prod is: 0xd0000000_0xe0000000
```

### zkCli
查看 broker，bundle 的负载数据
```bash
>>> ls /loadbalance
[resource-quota, bundle-data, leader, brokers, broker-time-average]
```

```java
public enum BrokerAssignment {
    primary, 
    secondary, 
    shared
}
```

pulsar dashboard 
也可以看到 bundle 的分配情况：
![](https://yzp-note.4everland.store/img/2022/12/202212021729956.png)

## 配置
```sh

默认 一个 namspace 切成 4 个 bundle

# conf/broker.conf

# When a namespace is created without specifying the number of bundle, this
# value will be used as the default
defaultNumberOfNamespaceBundles=4

# 创建 namespace 时，可以指定 budle 数
bin/pulsar-admin namespaces create my-tenant/my-namespace --clusters us-west --bundles 16

```

## metrics
https://pulsar.apache.org/docs/next/reference-metrics#loadbalancing-metrics

```bash
pulsar_lb_bandwidth_in_usage
pulsar_lb_bandwidth_out_usage
pulsar_lb_cpu_usage
pulsar_lb_directMemory_usage
pulsar_lb_memory_usage

pulsar_lb_unload_broker_total
pulsar_lb_unload_bundle_total
pulsar_lb_bundles_split_total

pulsar_bundle_msg_rate_in
pulsar_bundle_msg_rate_out
pulsar_bundle_msg_throughput_in
pulsar_bundle_msg_throughput_out

pulsar_bundle_topics_count
pulsar_bundle_consumer_count
pulsar_bundle_producer_count
```

![](https://yzp-note.4everland.store/img/2022/12/202212021801775.png)

## 如何实现一个 load balancer?

- 定义 负载均衡的粒度
  - topic (partitioned topic; physical topic) ?
  - bundle ( batch of topics) ?
  - msg ?
  - 因为 broker 是有一点状态的，粒度达到 msg，cache 就失去效果了。
    - (为了提高读取速度，在 broker 加了缓存，引入了状态。是一种“反模式”。)
    - tailing read , catch-up read
    - ![apache-pulsar-in-action](https://yzp-note.4everland.store/img/2022/12/202212051128543.png)
    - ![apache-pulsar-in-action](https://yzp-note.4everland.store/img/2022/12/202212091721510.png)
    - ![](https://yzp-note.4everland.store/img/2022/12/202212091747549.png)
  - 网关服务，可以做到更细的粒度。
    - 集中式 session -> 分布式 session -> JWT
  - pulsar 为什么要做 cache
    - LSM Tree vs B+Tree
  - 其他状态：topic - bookie
    - 存算分离
- 如何 水平扩展？
  - 一致性 hash
- 如何 计算 节点的负载？
  - 不同指标 加权求和
    - 类比：雨热同期
  - 移动平均数
- 如何 避免 频繁的节点切换？
  - 慢启动
  - 
- 如何 避免 流量的反复横跳？
  - 设置 最小负载迁移阈值
  - 保证 load 和 unload 的计算逻辑`一致`
- 如何 尽可能地 降低 节点变更 导致的 不可用时间？
  - 设置 最大负载迁移阈值
  - 先算出目标节点，再 unload 流量
- 一致性保证
  - read your write

### 一致性保证
类比 db:
```sql
pub:
insert into table_1

sub fetch:
select from table_1 where state=un_consumed limit 100
update table_1 set state=picked where id in (...)

sub ack:
update table_1 set state=consumed where id in (...)
```

![](https://yzp-note.4everland.store/img/2022/12/202212051107718.png)

![](https://yzp-note.4everland.store/img/2022/12/202212051104944.png)



### best practice
[Apache Pulsar 在微信大流量实时推荐场景下实践](https://juejin.cn/post/7164321406564450340#heading-4)
load 和 unload 逻辑不一致 导致 流量在 broker 之间反复横跳，
unload 根据 机器负载，
load 根据 消息量。

[Apache Pulsar 在 BIGO 的性能调优实战（上）](https://segmentfault.com/a/1190000023706815#:~:text=ZooKeeper%20%E5%A4%B1%E8%81%94%E3%80%82-,%E5%85%B3%E9%97%AD%20auto%20bundle%20split%EF%BC%8C%E4%BF%9D%E8%AF%81%E7%B3%BB%E7%BB%9F%E7%A8%B3%E5%AE%9A,-Pulsar%20bundle%20split)
unload 很慢。
推荐：
- 生产环境 关掉自动 split
- 预先规划，分配好 bundle
- 流量低的时候，手动 unload

## 代码实现

```java
- org.apache.pulsar.broker.PulsarService#start
- org.apache.pulsar.broker.PulsarService#startLoadManagementService
- org.apache.pulsar.broker.loadbalance.LoadReportUpdaterTask#run
- org.apache.pulsar.broker.loadbalance.LoadManager#writeLoadReportOnZookeeper()
```

- LoadReportUpdaterTask
- LoadResourceQuotaUpdaterTask
- LoadSheddingTask

统计 broker 的负载，
写入 zk
```java
broker 的负载：
org.apache.pulsar.policies.data.loadbalancer.SystemResourceUsage


// org.apache.pulsar.broker.loadbalance.impl.ModularLoadManagerImpl#writeBrokerDataOnZooKeeper(boolean)

public void writeBrokerDataOnZooKeeper(boolean force) {
  ...
  brokerDataLock.updateValue(localData).join();
  ...
}

// dive:
store.put(path, payload, Optional.of(version), EnumSet.of(CreateOption.Ephemeral))
```


```java
从 zk 读取 负载信息
// org.apache.pulsar.broker.loadbalance.impl.ModularLoadManagerImpl#updateAll
    public void updateAll() {
        if (log.isDebugEnabled()) {
            log.debug("Updating broker and bundle data for loadreport");
        }
        cleanupDeadBrokersData();
        updateAllBrokerData();
        updateBundleData();
        // broker has latest load-report: check if any bundle requires split
        checkNamespaceBundleSplit();
    }


// bundle Data
BrokerData.{LocalBrokerData}.{Map<String, NamespaceBundleStats>}


    public void checkNamespaceBundleSplit() {
...
        synchronized (bundleSplitStrategy) {
            final Set<String> bundlesToBeSplit = bundleSplitStrategy.findBundlesToSplit(loadData, pulsar);
            ...
            for (String bundleName : bundlesToBeSplit) {
                ...
                    pulsar.getAdminClient().namespaces().splitNamespaceBundle(namespaceName, bundleRange,
                        unloadSplitBundles, null);
                ...
            }

        }

    }

```

read /loadbalance/brokers
update broker data
update bundle data
write /loadbalance/broker-time-average

## refer
- https://pulsar.apache.org/docs/next/administration-load-balance/
- https://streamnative.io/blog/engineering/2022-07-21-achieving-broker-load-balancing-with-apache-pulsar/
- [Apache Pulsar 在 BIGO 的性能调优实战（上）](https://segmentfault.com/a/1190000023706815)
- [Consistency Models of NoSQL Databases](https://www.mdpi.com/1999-5903/11/2/43/htm)
- https://livebook.manning.com/book/apache-pulsar-in-action/chapter-2/v-2/88
- https://www.youtube.com/watch?v=ShjDrvV3MNE&ab_channel=TheApacheFoundation
- [Pulsar Broker Load Balance Improvement Areas](https://docs.google.com/document/d/1nXaiSK39E10awqinnDUX5Sial79FSTczFCmMdc9q8o8/edit)
  - Distribute the load balance decisions to local brokers
  - Improve availability of topic while unloading
  - ...
- [PIP-192: New Pulsar Broker Load Balancer](https://github.com/apache/pulsar/issues/16691)
- [分布式系统中的“无状态”和“有状态”详解_tencent_cloud](https://cloud.tencent.com/developer/article/1620559)
