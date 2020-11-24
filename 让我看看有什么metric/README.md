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

# gRPC metric

request inbound rate  #请求入站速率

    sum(rate(grpc_server_started_total{job="foo"}[1m])) by (grpc_service)
 
unary request error rate #一元请求错误率

    sum(rate(grpc_server_handled_total{job="foo",grpc_type="unary",grpc_code!="OK"}[1m])) by (grpc_service)

unary request error percentage。#请求错误百分比

    sum(rate(grpc_server_handled_total{job="foo",grpc_type="unary",grpc_code!="OK"}[1m])) by (grpc_service)
     / 
    sum(rate(grpc_server_started_total{job="foo",grpc_type="unary"}[1m])) by (grpc_service)
     * 100.0

average response stream size #平均响应流大小

    sum(rate(grpc_server_msg_sent_total{job="foo",grpc_type="server_stream"}[10m])) by (grpc_service)
     /
    sum(rate(grpc_server_started_total{job="foo",grpc_type="server_stream"}[10m])) by (grpc_service)

99%-tile latency of unary requests  #99%响应时间的延迟

     histogram_quantile(0.99, 
      sum(rate(grpc_server_handling_seconds_bucket{job="foo",grpc_type="unary"}[5m])) by (grpc_service,le)
    )

percentage of slow unary queries (>250ms) #慢速一元查询百分比（>250ms）

    100.0 - (
    sum(rate(grpc_server_handling_seconds_bucket{job="foo",grpc_type="unary",le="0.25"}[5m])) by (grpc_service)
     / 
    sum(rate(grpc_server_handling_seconds_count{job="foo",grpc_type="unary"}[5m])) by (grpc_service)
    ) * 100.0

# HPA metric 

# kube-state-metrics
