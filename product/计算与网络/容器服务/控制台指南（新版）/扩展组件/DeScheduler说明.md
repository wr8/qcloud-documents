## 简介
### 组件介绍

DeScheduler 是容器服务 TKE 基于 Kubernetes 原生社区 [DeScheduler](https://github.com/kubernetes-sigs/descheduler)  实现的一个基于 Node 真实负载进行重调度的插件。在 TKE 集群中安装该插件后，该插件会和 Kube-scheduler 协同生效，实时监控集群中高负载节点并驱逐低优先级 Pod。 
该插件依赖 Prometheus 监控组件以及相关规则配置，建议您安装插件之前仔细阅读 [依赖部署](#DeScheduler)，以免插件无法正常工作。
建议您搭配 TKE [Dynamic Scheduler（动态调度器扩展组件）](https://cloud.tencent.com/document/product/457/50843) 一起使用，多维度保障集群负载均衡。


### 在集群内部署 Kubernetes 对象

| Kubernetes 对象名称  | 类型               |                   请求资源                   | 所属 Namespace |
| :----------------- | :----------------- | :------------------------------------------: | ------------- |
| descheduler        | Deployment         | 每个实例 CPU:200m，Memory:200Mi，共1个实例 | kube-system   |
| descheduler        | ClusterRole        |                      -                       | kube-system   |
| descheduler        | ClusterRoleBinding |                      -                       | kube-system   |
| descheduler        | ServiceAccount     |                      -                       | kube-system   |
| descheduler-policy | ConfigMap          |                      -                       | kube-system   |
| probe-prometheus   | ConfigMap          |                      -                       | kube-system   |


## 使用场景

DeScheduler 通过重调度来解决集群现有节点上不合理的运行方式。社区版本 DeScheduler 中提出的策略基于 APIServer 中的数据实现，并未基于节点真实负载。因此可以增加对于节点的监控，基于真实负载进行重调度调整。

容器服务 TKE 自研的 ReduceHighLoadNode 策略依赖 Prometheus 和 node_exporter 监控数据，根据节点 CPU 利用率、内存利用率、网络 IO、system loadavg 等指标进行 Pod 驱逐重调度，防止出现节点极端负载的情况。DeScheduler 的 ReduceHighLoadNode 与 TKE 自研的 Dynamic Scheduler 基于节点真实负载进行调度的策略需配合使用。


## 限制条件

Kubernetes 版本 ≥ v1.10.x



## 依赖部署[](id:DeScheduler)

DeScheduler 组件依赖于 Node 当前和过去一段时间的真实负载情况来进行调度决策，需要通过 Prometheus 等监控组件获取系统 Node 真实负载信息。在使用 DeScheduler 组件之前，您可以采用自建 Prometheus 监控或采用 TKE 云原生监控，以下将详细介绍：

- [自建 Prometheus 监控服务](#Prometheus1)
- [云原生监控 Prometheus](#Prometheus2)


### 自建 Prometheus 监控服务[](id:Prometheus1)

#### 部署 node-exporter 和 Prometheus

通过 node-exporter 实现对于 Node 指标的监控，您可按需部署 node-exporter 和 Prometheus。

#### 聚合规则配置[](id:rules)

在 node-exporter 获取节点监控数据后，需要通过 Prometheus 对原始的 node-exporter 中采集数据进行聚合计算。为获取 DeScheduler 所需要的 `cpu_usage_avg_5m`、`mem_usage_avg_5m` 等指标，需要在 Prometheus 的 rules 规则中进行配置。示例如下：

```
groups:
      - name: cpu_mem_usage_active
        interval: 30s
        rules:
        - record: mem_usage_active
          expr: 100*(1-node_memory_MemAvailable_bytes/node_memory_MemTotal_bytes)
      - name: cpu-usage-1m
        interval: 1m
        rules:
        - record: cpu_usage_avg_5m
          expr: 100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
      - name: mem-usage-1m
        interval: 1m
        rules:
        - record: mem_usage_avg_5m
          expr: avg_over_time(mem_usage_active[5m])
```

#### Prometheus 文件配置


1. 上述定义了 DeScheduler 所需要的指标计算的 rules，需要将 rules 配置到 Prometheus 中，参考一般的 Prometheus 配置文件。示例如下：
```
global:
      evaluation_interval: 30s
      scrape_interval: 30s
      external_labels:
rule_files:
- /etc/prometheus/rules/*.yml # /etc/prometheus/rules/*.yml 是定义的 rules 文件
```
2. 将 rules 配置复制到一个文件（例如 de-scheduler.yaml），文件放到上述 Prometheus 容器的 `/etc/prometheus/rules/` 下。
3. 重新加载 Prometheus server，即可从 Prometheus 中获取到动态调度器需要的指标。
 
>?通常情况下，上述 Prometheus 配置文件和 rules 配置文件都是通过 configmap 存储，再挂载到 Prometheus server 容器，因此修改相应的 configmap 即可。


### 云原生监控 Prometheus[](id:Prometheus2)

1. 登录容器服务控制台，在左侧菜单栏中选择【[云原生监控](https://console.cloud.tencent.com/tke2/prometheus)】，进入“云原生监控”页面。
2. 创建与 Cluster 处于同一 VPC 下的 [云原生监控 Prometheus 实例](https://cloud.tencent.com/document/product/457/49889#.E5.88.9B.E5.BB.BA.E7.9B.91.E6.8E.A7.E5.AE.9E.E4.BE.8B)，并 [关联用户集群](https://cloud.tencent.com/document/product/457/49890)。
   ![](https://main.qcloudimg.com/raw/bafb027663fbb3f2a5063531743c2e97.jpg)
2. 与原生托管集群关联后，可以在用户集群查看到每个节点都已安装 node-exporter。 
   ![](https://main.qcloudimg.com/raw/e35d4af7eeba15f6d9da62ce79176904.png)
3. 设置 Prometheus 聚合规则，具体规则内容与上述 [规则](#rules) 相同。规则保存后立即生效，无需重新加载 server。


## 组件原理

DeScheduler  基于 [社区版本 Descheduler](https://github.com/kubernetes-sigs/descheduler) 的重调度思想，定期扫描各个节点上的运行 Pod，发现不符合策略条件的进行驱逐以进行重调度。社区版本 DeScheduler  已提供部分策略，策略基于 APIServer 中的数据，例如 `LowNodeUtilization` 策略依赖的是 Pod 的 request 和 limit 数据，这类数据能够有效均衡集群资源分配、防止出现资源碎片。但社区策略缺少节点真实资源占用的支持，例如节点 A 和 B 分配出去的资源一致，由于 Pod 实际运行情况，CPU 消耗型和内存消耗型不同，峰谷期不同造成两个节点的负载差别巨大。

因此，腾讯云 TKE 推出 DeScheduler，底层依赖对节点真实负载的监控进行重调度。具体实现上，通过 Prometheus 拿到集群 Node 的负载统计信息，根据用户设置的负载阈值，定期执行策略里面的检查规则，驱逐高负载节点上的 Pod 。

![](https://main.qcloudimg.com/raw/9a31a5d0995c40f3540a83da3b037323.png)


### 查找高负载节点

![](https://main.qcloudimg.com/raw/ac5285d3fc10fad645239507570a3e39.png)

### 筛选可驱逐 Pod

![](https://main.qcloudimg.com/raw/00c60959cb1956e1a1cfa9d683f1f542.png)

可迁移标记是 TKE 指定的 annotation，设置为 `"descheduler.alpha.kubernetes.io/evictable": true`，注入到 workload 中。


### 根据 Pod 驱逐顺序进行驱逐

当节点 CPU 或者内存超过阈值时，对节点进行 Pod 驱逐的顺序基于以下规则排序，例如有两个 Pod，A 与 B。

>? 当节点 CPU 和内存均超过阈值时，DeScheduler 将先按照降低内存到目标水位的策略去驱逐 Pod，因为内存是不可压缩资源，且会同步将驱逐的 Pod 对节点 CPU 的降低值更新到节点负载中，最后再按照降低 CPU 到目标水位的策略去驱逐 Pod。

1. priority 值低的 Pod 优先驱逐。
2. QosClass 低的（besteffort < burstable < guaruanteed）优先驱逐。
3. 如果 A 与 B 的 priority 与 QosClass 都相同，则比较二者的 CPU 和内存利用率，利用率高的优先驱逐（为了快速降低负载）。


## 组件参数说明[](id:parameter)

### Prometheus 数据查询地址


>!为确保组件可以拉取到所需的监控数据、调度策略生效，请按照【[依赖部署](#DeScheduler)】>【[Prometheus 规则配置](#Prometheus1)】步骤配置监控数据采集规则。

- 如果使用自建 Prometheus，直接填入数据查询 URL（HTTPS/HTTPS）即可。
- 如果使用托管 Prometheus，选择托管实例 ID 即可，系统会自动解析实例对应的数据查询 URL。

### 利用率阈值和目标利用率

>! 负载阈值参数已设置默认值，如您无额外需求，可直接采用。

过去5分钟内，节点的 CPU 平均利用率或者内存平均使用率超过设定阈值，Descheduler 会判断节点为高负载节点，执行 Pod 驱逐逻辑（不可驱逐 Pod 筛选以及驱逐顺序请参考组件说明），并尽量通过 Pod 重调度使节点负载降到目标利用率以下。


## 风险控制

1. 该组件已对接容器服务 TKE 的监控告警体系。
2. 推荐您为集群开启事件持久化，以便更好的监控组件异常以及故障定位。
3. 为避免 DeScheduler 驱逐关键的 Pod，设计的算法默认不驱逐 Pod，对于可以驱逐的 Pod，用户需要显示给判断 Pod 所属 workload。例如，statefulset、deployment 等对象设置可驱逐 annotation。
4. 驱逐太多 Pod，导致服务不可用。
   Kubernetes 原生提供 PDB 对象用于防止驱逐接口造成的 workload 不可用 Pod 过多，但需要用户创建该 PDB 配置。容器服务 TKE 自研的 DeScheduler 组件加入了兜底措施，即调用驱逐接口前，判断 workload 准备的 Pod 数是否大于副本数一半，否则不调用驱逐接口。

## 操作步骤

1. 登录 [容器服务](https://console.cloud.tencent.com/tke2/cluster) 控制台。
2. 按照 [依赖部署](#DeScheduler) 部署 Prometheus、Node-Exporter，并配置好 Prometheus Rule。
3. 单击左侧导航栏中的【集群】，进入集群管理界面。
4. 单击需新建组件的集群 ID，进入集群详情页，在该页面左侧栏中选择【组件管理】。
5. 单击【新建】，进入“新建组件”页面。
6. 在组件列表勾选【Decheduler（重调度器）】，单击【参数配置】。
7. 按照 [参数说明](#parameter) 填写组件所需参数。
8. 单击【完成】，组件安装成功后 DeScheduler 即可正常运行，无需进行额外配置。
9. 对于用户认为可以驱逐的 workload（例如 statefulset、deployment 等对象），可以设置 Annotation 如下：
```plaintext
descheduler.alpha.kubernetes.io/evictable: 'true'
```

