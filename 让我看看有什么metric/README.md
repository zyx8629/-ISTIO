#  Metric Requirements

## Kubelet

 Node-level usage metrics for Filesystems, CPU, and Memory

 Pod-level usage metrics for Filesystems and Memory

## Metrics Server (outlined in Monitoring Architecture), which exposes the Resource Metrics API to the following system components:

Scheduler

Node-level usage metrics for Filesystems, CPU, and Memory

Pod-level usage metrics for Filesystems, CPU, and Memory

## Vertical-Pod-Autoscaler

Node-level usage metrics for Filesystems, CPU, and Memory

Pod-level usage metrics for Filesystems, CPU, and Memory

Container-level usage metrics for Filesystems, CPU, and Memory

## Horizontal-Pod-Autoscaler

Node-level usage metrics for CPU and Memory

Pod-level usage metrics for CPU and Memory

## Cluster Federation

Node-level usage metrics for Filesystems, CPU, and Memory

## kubectl top and Kubernetes Dashboard

Node-level usage metrics for Filesystems, CPU, and Memory

Pod-level usage metrics for Filesystems, CPU, and Memory

Container-level usage metrics for Filesystems, CPU, and Memory

# 1 、 gRPC metric

* request inbound rate  #请求入站速率

    sum(rate(grpc_server_started_total{job="foo"}[1m])) by (grpc_service)
 
* unary request error rate #一元请求错误率

    sum(rate(grpc_server_handled_total{job="foo",grpc_type="unary",grpc_code!="OK"}[1m])) by (grpc_service)

* unary request error percentage。#请求错误百分比

    sum(rate(grpc_server_handled_total{job="foo",grpc_type="unary",grpc_code!="OK"}[1m])) by (grpc_service)
     / 
    sum(rate(grpc_server_started_total{job="foo",grpc_type="unary"}[1m])) by (grpc_service)
     * 100.0

* average response stream size #平均响应流大小

    sum(rate(grpc_server_msg_sent_total{job="foo",grpc_type="server_stream"}[10m])) by (grpc_service)
     /
    sum(rate(grpc_server_started_total{job="foo",grpc_type="server_stream"}[10m])) by (grpc_service)

* * 99%-tile latency of unary requests  #99%响应时间的延迟

     histogram_quantile(0.99, 
      sum(rate(grpc_server_handling_seconds_bucket{job="foo",grpc_type="unary"}[5m])) by (grpc_service,le)
    )

percentage of slow unary queries (>250ms) #慢速查询百分比（时间>250ms）

    100.0 - (
    sum(rate(grpc_server_handling_seconds_bucket{job="foo",grpc_type="unary",le="0.25"}[5m])) by (grpc_service)
     / 
    sum(rate(grpc_server_handling_seconds_count{job="foo",grpc_type="unary"}[5m])) by (grpc_service)
    ) * 100.0


# 2 、kube-state-metrics [ https://github.com/kubernetes/kube-state-metrics/tree/master/docs ]

## Network Policy Metrics

|              Metric name              | Metric type |                         Labels/tags                          |    Status    |
| :-----------------------------------: | :---------: | :----------------------------------------------------------: | :----------: |
|      kube_networkpolicy_created       |    Gauge    | namespace=<namespace name\>                                             networkpolicy=<networkpolicy name\> | EXPERIMENTAL |
|       kube_networkpolicy_labels       |    Gauge    | namespace=<namespace name\>                                             networkpolicy=<networkpolicy name\> | EXPERIMENTAL |
| kube_networkpolicy_spec_egress_rules  |    Gauge    | namespace=<namespace name\>                                             networkpolicy=<networkpolicy name\> | EXPERIMENTAL |
| kube_networkpolicy_spec_ingress_rules |    Gauge    | namespace=<namespace name\>                                             networkpolicy=<networkpolicy name\> | EXPERIMENTAL |


## Pod Metrics

* To get the list of pods that are in the Unknown state #查未知状态的pod

    sum(kube_pod_status_phase{phase="Unknown"}) by (namespace, pod) or (count(kube_pod_deletion_timestamp) by (namespace, pod) * sum(kube_pod_status_reason{reason="NodeLost"}) by(namespace, pod))

* For Pods in Terminating state #查找正在被销毁的pod

    count(kube_pod_deletion_timestamp) by (namespace, pod) * count(kube_pod_status_reason{reason="NodeLost"} == 0) by (namespace, pod)

## Node Metrics

|        Metric name         | Metric type |                         Labels/tags                          | Status |
| :------------------------: | :---------: | :----------------------------------------------------------: | :----: |
| kube_node_status_capacity  |    Gauge    | `node`=<node-address\><br/>`resource`=<resource-name\><br/>`unit`=<resource-unit\> | STABLE |
| kube_node_status_condition |    Gauge    | `node`=<node-address\><br/>`condition`=<node-condition\><br/>`status`=<true\|false\|unknown> | STABLE |

# 3、Kube-state-metrics self metrics

* kube-state-metrics exposes list and watch success and error metrics

      kube_state_metrics_list_total{resource="*v1.Node",result="success"} 1
      kube_state_metrics_list_total{resource="*v1.Node",result="error"} 52
      kube_state_metrics_watch_total{resource="*v1beta1.Ingress",result="success"} 1

* kube-state-metrics exposes some http request metrics

      http_request_duration_seconds_bucket{handler="metrics",method="get",le="2.5"} 30
      http_request_duration_seconds_bucket{handler="metrics",method="get",le="5"} 30
      http_request_duration_seconds_bucket{handler="metrics",method="get",le="10"} 30
      http_request_duration_seconds_bucket{handler="metrics",method="get",le="+Inf"} 30
      http_request_duration_seconds_sum{handler="metrics",method="get"} 0.021113919999999998
      http_request_duration_seconds_count{handler="metrics",method="get"} 30
      
# 4、Horizontal Pod Autoscaler metrics 【在这里发现了定义多维度metric的样例 https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/ 】

* 定义多维指标
  
      apiVersion: autoscaling/v2beta2
      kind: HorizontalPodAutoscaler
      metadata:
        name: php-apache    #对这个pod做垂直伸缩
      spec:
        scaleTargetRef:
          apiVersion: apps/v1
          kind: Deployment
          name: php-apache
        minReplicas: 1     #最小有1个副本
        maxReplicas: 10    #最多有十个副本
        metrics:           #开始自定义度量标准
        - type: Resource
          resource:
            name: cpu     #cpu平均利用率50%
            target:
              type: Utilization
              averageUtilization: 50
        - type: Pods
          pods:         #pod每秒接受的数据达到1k
            metric:
              name: packets-per-second
            target:
              type: AverageValue
              averageValue: 1k
        - type: Object
          object:
            metric:
              name: requests-per-second
            describedObject:
              apiVersion: networking.k8s.io/v1beta1
              kind: Ingress
              name: main-route
            target:
              type: Value
              value: 10k
      status:
        observedGeneration: 1
        lastScaleTime: <some-time>
        currentReplicas: 1
        desiredReplicas: 1
        currentMetrics:
        - type: Resource
          resource:
            name: cpu
          current:
            averageUtilization: 0
            averageValue: 0
        - type: Object
          object:
            metric:
              name: requests-per-second
            describedObject:
              apiVersion: networking.k8s.io/v1beta1
              kind: Ingress
              name: main-route
            current:
              value: 10k

* HAP指标类型

      Object类型：Object类型是用于描述k8s内置对象的指标，例如ingress对象中hits-per-second指标
      Pods类型：pods类型描述当前扩容目标中每个pod的指标（例如，每秒处理的事务数）。在与目标值进行比较之前，将对值进行平均。
      Resource类型：Resource是Kubernetes已知的资源指标,如request和limit中所指定的，描述当前扩容目标的每个pod（例如CPU或内存）。该指标将会在与目标值对比前进行平均，被此类指标内置于Kubernetes中,且使用"pods"源在正常的每个pod度量标准之上提供特殊的扩展选项。
      External 类型：ExternalMetricSource指示如何扩展与任何Kubernetes对象无关的指标（例如，云消息传递服务中的队列长度，或集群外部运行的负载均衡器的QPS）。
    
* HAP指标来源

1. 用于hpa 中Resource类型数据来源.

       metric-server
       heapster
       
2. 用于hpa 中object/pods类型的数据来源，需要自己实现适配器

       Prometheus Adapter
       Microsoft Azure Adapter
       Google Stackdriver
       
3. 用于 hpa 中external类型的数据来源，需要云厂商或平台自己实现适配器（custom metrics adapters）

       Stackdriver

   
