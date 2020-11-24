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
|      kube_networkpolicy_created       |    Gauge    | namespace=<namespace name>                                             networkpolicy=<networkpolicy name> | EXPERIMENTAL |
|       kube_networkpolicy_labels       |    Gauge    | namespace=<namespace name>                                             networkpolicy=<networkpolicy name> | EXPERIMENTAL |
| kube_networkpolicy_spec_egress_rules  |    Gauge    | namespace=<namespace name>                                             networkpolicy=<networkpolicy name> | EXPERIMENTAL |
| kube_networkpolicy_spec_ingress_rules |    Gauge    | namespace=<namespace name>                                             networkpolicy=<networkpolicy name> | EXPERIMENTAL |


## Pod Metrics

* To get the list of pods that are in the Unknown state #查未知状态的pod

    sum(kube_pod_status_phase{phase="Unknown"}) by (namespace, pod) or (count(kube_pod_deletion_timestamp) by (namespace, pod) * sum(kube_pod_status_reason{reason="NodeLost"}) by(namespace, pod))

* For Pods in Terminating state #查找正在被销毁的pod

    count(kube_pod_deletion_timestamp) by (namespace, pod) * count(kube_pod_status_reason{reason="NodeLost"} == 0) by (namespace, pod)

## Node Metrics

|        Metric name         | Metric type |                         Labels/tags                          | Status |
| :------------------------: | :---------: | :----------------------------------------------------------: | :----: |
| kube_node_status_capacity  |    Gauge    | `node`=<node-address><br/>`resource`=<resource-name><br/>`unit`=<resource-unit> | STABLE |
| kube_node_status_condition |    Gauge    | `node`=<node-address><br/>`condition`=<node-condition><br/>`status`=<true\|false\|unknown> | STABLE |



