kubernetes集群事件采集展示
----

# 特点

1. 适配clickhouse
2. grafana面板:
    1. 重要事件, 时间线, 统计, 查询
    2. 支持搜索

# 原理

kubernetes-event-exporter --> kafka --> clickhouse --> grafana

* 采集组件: [kubernetes-event-exporter](https://github.com/resmoio/kubernetes-event-exporter)
* MQ: kafka
* 数据存储: clickhouse
* 展示: grafana

# 前提

你首先需要需要以下资源, 不管是自行搭建还是向公司团队申请:

* kubernetes集群
* kafka
* clickhouse

# 步骤

## 1. 建立clickhouse表

```text
CREATE TABLE events_table
(
    env String,
    iDc String,
    city String,
    cluster String,
    apiVersion String,
    kind String,
    metadata_name String,
    metadata_namespace String,
    metadata_uid String,
    metadata_creationTimestamp DateTime('Asia/Shanghai'),
    involvedObject_kind String,
    involvedObject_name String,
    involvedObject_namespace String,
    involvedObject_uid String,
    involvedObject_apiVersion String,
    reason String,
    message String,
    source_component String,
    source_host String,
    firstTimestamp DateTime('Asia/Shanghai'),
    lastTimestamp DateTime('Asia/Shanghai'),
    count UInt32,
    type String,
    eventTime DateTime('Asia/Shanghai'),
    reserved1 String,
    reserved2 String,
    reserved3 String,
    reserved4 String
)
ENGINE = MergeTree()
ORDER BY eventTime
```

## 2. kafka

以下步骤略, 自行搭建

### kafka建立topic

### 打通kafka和clickhouse

## 3. 安装kubernetes-event-exporter

### config.yaml

```yaml
logLevel: info
logFormat: json
metricsNamePrefix: "event_exporter_"
maxEventAgeSeconds: 120
kubeQPS: 100
kubeBurst: 500
route:
  routes:
    - match:
        - receiver: "clickhouse_events"
receivers:
  - name: "clickhouse_events"
    kafka:

      clientId: "CLUSTER_NAME"
      topic: "TOPIC_NAME"
      brokers:
        - "IP1:PORT1"
        - "IP2:PORT2"
        - "IP3:PORT3"
      compressionCodec: "snappy"
      layout:
        "env": ""
        "iDc": ""
        "city": ""
        "cluster": ""
        "reserved1": ""
        "reserved2": ""
        "reserved3": ""
        "reserved4": ""

        "apiVersion": "{{ .APIVersion }}"
        "kind": "{{ .Kind }}"
        "metadata_name": "{{ .Name }}"
        "metadata_namespace": "{{ .Namespace }}"
        "metadata_uid": "{{ .UID }}"
        "involvedObject_kind": "{{ .InvolvedObject.Kind }}"
        "involvedObject_name": "{{ .InvolvedObject.Name }}"
        "involvedObject_namespace": "{{ .InvolvedObject.Namespace }}"
        "involvedObject_uid": "{{ .InvolvedObject.UID }}"
        "involvedObject_apiVersion": "{{ .InvolvedObject.APIVersion }}"
        "reason": "{{ .Reason }}"
        "message": "{{ .Message }}"
        "source_component": "{{ .Source.Component }}"
        "source_host": "{{ .Source.Host }}"
        "firstTimestamp": "{{ dateInZone \"2006-01-02 15:04:05\" .FirstTimestamp.Time \"Asia/Shanghai\" }}"
        "lastTimestamp": "{{ dateInZone \"2006-01-02 15:04:05\" .LastTimestamp.Time \"Asia/Shanghai\" }}"
        "count": "{{ .Count }}"
        "type": "{{ .Type }}"
        "eventTime": "{{ dateInZone \"2006-01-02 15:04:05\" .GetTimestampCK \"Asia/Shanghai\" }}"
```

因为我们一般会采集很多个集群, 以上信息需要自行修改:

```
clientId: # 一般使用集群唯一标识, 如集群名称CLUSTER_NAME
topic: # kafka topic
brokers: # kafka broker地址
layout:
    "env": "" # 环境信息
    "iDc": "" # 机房信息
    "city": "" # 城市信息
    "cluster": "" # 集群名称CLUSTER_NAME
    "reservedN" : "" # 保留字段1-4, 自行配置, 可以用在grafana中做展示
```

#### 注意点说明

kubernetes的event结构体里的时间字段使用两种类型`metav1.Time`, `metav1.MicroTime`

我们在clickhouse表中, 使用`DateTime`作为时间字段类型, 格式为`2006-01-02 15:04:05`

以上kubernetes event默认转成json格式, 发送到kafka中, 和clickhouse的时间格式不一致, 会导致数据导入失败, 因此需要进行转换.

查看kubernetes-event-exporter源码, 其使用了[sprig](https://github.com/Masterminds/sprig)作为模板引擎,
所以我们可以使用`dateInZone`函数来转换.

这里一定要注意必须统一时区, 否则会出现在grafana中选择时间范围后, 数据没有显示或者显示时区错乱数据, 问题很严重.

关于`GetTimestampCK`:

kubernetes-event-exporter里对于`EnhancedEvent`扩展了几个GetTime*
方法, [地址](https://github.com/resmoio/kubernetes-event-exporter/blob/master/pkg/kube/event.go)

我们使用eventTime作为事件发生的时间, 需要首先取FirstTimestamp, 如果没有则取EventTime, 因此我添加了GetTimestampCK方法,
自定打包了镜像

```text
func (e *EnhancedEvent) GetTimestampCK() time.Time {
	timestamp := e.FirstTimestamp.Time
	if timestamp.IsZero() {
		timestamp = e.EventTime.Time
	}
	return timestamp
}
```

### deploy

自行修改配置后, 直接部署即可

[deployment.yaml](deployment.yaml)

`kubectl apply -f deployment.yaml`

### check

可以通过以下方式检查数据是否导入成功:

1. check kafka topic
2. check clickhouse table data

## 4. 安装node problem detector

非必要, 但是强烈建议.

Kubernetes中关于Node的事件不多，对于节点上更多偏向底层的状态（如内核死锁、容器运行时无响应等）并不能通过事件的方式通知出来。
Node Problem Detector作为一个很好的补充，它可以将node上更细节的事件以NodeCondition和Event方式上报给Kubernetes。

安装参考`https://github.com/kubernetes/node-problem-detector`

## 5. 配置grafana dashboard

说明: 本人使用版本为Grafana v11.2.0

### 安装插件

Altinity plugin for ClickHouse

### 配置数据源

略

### 配置dashboard

import [grafana_base_clickhouse.json](grafana_base_clickhouse.json)

需要更改:

1. 数据源名称

#### 说明

变量filter, 用于配置全局筛选条件

## 效果

![img.png](imgs%2Fimg.png)

![img_1.png](imgs%2Fimg_1.png)

![img_2.png](imgs%2Fimg_2.png)