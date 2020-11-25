# 1 、 基于 prometheus 的gRPC metric

* request inbound rate  #请求入站速率
	
   	 	sum(rate(grpc_server_started_total{job="foo"}[1m])) by (grpc_service)
 
* unary request error rate #一元请求错误率

   	 	sum(rate(grpc_server_handled_total{job="foo",grpc_type="unary",grpc_code!="OK"}[1m])) by (grpc_service)

* unary request error percentage。#请求错误百分比

	    sum(rate(grpc_server_handled_total{job="foo",grpc_type="unary",grpc_code!="OK"}[1m])) by (grpc_service)
	     / 
	    sum(rate(grpc_server_started_total{job="foo",grpc_type="unary"}[1m])) by (grpc_service) * 100.0

* average response stream size #平均响应流大小

	    sum(rate(grpc_server_msg_sent_total{job="foo",grpc_type="server_stream"}[10m])) by (grpc_service)
	     /
	    sum(rate(grpc_server_started_total{job="foo",grpc_type="server_stream"}[10m])) by (grpc_service)

* 99%-tile latency of unary requests  # 99%响应时间的延迟

    	 histogram_quantile(0.99, 
      sum(rate(grpc_server_handling_seconds_bucket{job="foo",grpc_type="unary"}[5m])) by (grpc_service,le))

* percentage of slow unary queries (>250ms) #慢速查询百分比（时间>250ms）

	    100.0 - (sum(rate(grpc_server_handling_seconds_bucket{job="foo",grpc_type="unary",le="0.25"}[5m])) by (grpc_service)/ sum(rate(grpc_server_handling_seconds_count{job="foo",grpc_type="unary"}[5m])) by (grpc_service)) * 100.0


# 2 、kube-state-metrics  [ https://github.com/kubernetes/kube-state-metrics/tree/master/docs ]

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
      
# 4、Horizontal Pod Autoscaler metrics [在这里发现了定义多维度metric的样例 https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/]

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
       
# k8s 提供的 metrics-server

主要是cpu，memory使用率指标

   
# 基于 prometheus 的 cadvisor [https://github.com/google/cadvisor/blob/master/metrics/prometheus.go]

这是一个容器监控工具，基于 prometheus 参数，也是找到了他的代码，看到了一些函数在计算 metric 参数

	1、	c.containerMetrics: []containerMetric{
			{
				name:      "container_last_seen",
				help:      "Last time a container was seen by the exporter",
				valueType: prometheus.GaugeValue,
				getValues: func(s *info.ContainerStats) metricValues {
					return metricValues{{
						value:     float64(now.Now().Unix()),
						timestamp: now.Now(),
					}}
				},
			},
		},
	2、	c.containerMetrics = append(c.containerMetrics, []containerMetric{
			{
				name:      "container_cpu_user_seconds_total",
				help:      "Cumulative user cpu time consumed in seconds.",
				valueType: prometheus.CounterValue,
				getValues: func(s *info.ContainerStats) metricValues {
					return metricValues{
						{
							value:     float64(s.Cpu.Usage.User) / float64(time.Second),
							timestamp: s.Timestamp,
						},
					}
				},
			}, {
				name:      "container_cpu_system_seconds_total",
				help:      "Cumulative system cpu time consumed in seconds.",
				valueType: prometheus.CounterValue,
				getValues: func(s *info.ContainerStats) metricValues {
					return metricValues{
						{
							value:     float64(s.Cpu.Usage.System) / float64(time.Second),
							timestamp: s.Timestamp,
						},
					}
				},
			}, {
			
			。。。。。。
		
# 基于 prometheus 的 haproxy_exporter [https://github.com/prometheus/haproxy_exporter/blob/master/haproxy_exporter.go]

这是一个负载均衡器的监测器，找到实现 metric 监测的代码，主要是依靠提取 prometheus 的监测参数，在代码最前面有引入 prometheus 相关工具包，还定三种类型metric的计算函数

可参考它的 metric 参数进行后续数据收集

* serverMetric

                 newServerMetric("current_queue", "Current number of queued requests assigned to this server.", prometheus.GaugeValue, nil),
		 newServerMetric("max_queue", "Maximum observed number of queued requests assigned to this server.", prometheus.GaugeValue, nil),
		 newServerMetric("current_sessions", "Current number of active sessions.", prometheus.GaugeValue, nil),
		 newServerMetric("max_sessions", "Maximum observed number of active sessions.", prometheus.GaugeValue, nil),
		 newServerMetric("limit_sessions", "Configured session limit.", prometheus.GaugeValue, nil),
	         newServerMetric("sessions_total", "Total number of sessions.", prometheus.CounterValue, nil),
		 newServerMetric("bytes_in_total", "Current total of incoming bytes.", prometheus.CounterValue, nil),
		 newServerMetric("bytes_out_total", "Current total of outgoing bytes.", prometheus.CounterValue, nil),
		 newServerMetric("connection_errors_total", "Total of connection errors.", prometheus.CounterValue, nil),
		 newServerMetric("response_errors_total", "Total of response errors.", prometheus.CounterValue, nil),
		 newServerMetric("retry_warnings_total", "Total of retry warnings.", prometheus.CounterValue, nil),
		 newServerMetric("redispatch_warnings_total", "Total of redispatch warnings.", prometheus.CounterValue, nil),
		 newServerMetric("up", "Current health status of the server (1 = UP, 0 = DOWN).", prometheus.GaugeValue, nil),
		 newServerMetric("weight", "Current weight of the server.", prometheus.GaugeValue, nil),
		 newServerMetric("check_failures_total", "Total number of failed health checks.", prometheus.CounterValue, nil),
		 newServerMetric("downtime_seconds_total", "Total downtime in seconds.", prometheus.CounterValue, nil),
		 newServerMetric("server_selected_total", "Total number of times a server was selected, either for new sessions, or when re-dispatching.", prometheus.CounterValue, nil),
		 newServerMetric("current_session_rate", "Current number of sessions per second over last elapsed second.", prometheus.GaugeValue, nil),
		 newServerMetric("max_session_rate", "Maximum observed number of sessions per second.", prometheus.GaugeValue, nil),
		 newServerMetric("check_duration_seconds", "Previously run health check duration, in seconds", prometheus.GaugeValue, nil),
		 newServerMetric("http_responses_total", "Total of HTTP responses.", prometheus.CounterValue, prometheus.Labels{"code": "1xx"}),
		 newServerMetric("http_responses_total", "Total of HTTP responses.", prometheus.CounterValue, prometheus.Labels{"code": "2xx"}),
		 newServerMetric("http_responses_total", "Total of HTTP responses.", prometheus.CounterValue, prometheus.Labels{"code": "3xx"}),
		 newServerMetric("http_responses_total", "Total of HTTP responses.", prometheus.CounterValue, prometheus.Labels{"code": "4xx"}),
		 newServerMetric("http_responses_total", "Total of HTTP responses.", prometheus.CounterValue, prometheus.Labels{"code": "5xx"}),
		 newServerMetric("http_responses_total", "Total of HTTP responses.", prometheus.CounterValue, prometheus.Labels{"code": "other"}),
		 newServerMetric("client_aborts_total", "Total number of data transfers aborted by the client.", prometheus.CounterValue, nil),
		 newServerMetric("server_aborts_total", "Total number of data transfers aborted by the server.", prometheus.CounterValue, nil),

* frontendMetrics

                newFrontendMetric("current_sessions", "Current number of active sessions.", prometheus.GaugeValue, nil),
	        newFrontendMetric("max_sessions", "Maximum observed number of active sessions.", prometheus.GaugeValue, nil),
		newFrontendMetric("limit_sessions", "Configured session limit.", prometheus.GaugeValue, nil),
		newFrontendMetric("sessions_total", "Total number of sessions.", prometheus.CounterValue, nil),
		newFrontendMetric("bytes_in_total", "Current total of incoming bytes.", prometheus.CounterValue, nil),
		newFrontendMetric("bytes_out_total", "Current total of outgoing bytes.", prometheus.CounterValue, nil),
	        newFrontendMetric("requests_denied_total", "Total of requests denied for security.", prometheus.CounterValue, nil),
		newFrontendMetric("request_errors_total", "Total of request errors.", prometheus.CounterValue, nil),
		newFrontendMetric("current_session_rate", "Current number of sessions per second over last elapsed second.", prometheus.GaugeValue, nil),
		newFrontendMetric("limit_session_rate", "Configured limit on new sessions per second.", prometheus.GaugeValue, nil),
	        newFrontendMetric("max_session_rate", "Maximum observed number of sessions per second.", prometheus.GaugeValue, nil),
		newFrontendMetric("http_responses_total", "Total of HTTP responses.", prometheus.CounterValue, prometheus.Labels{"code": "1xx"}),
		newFrontendMetric("http_responses_total", "Total of HTTP responses.", prometheus.CounterValue, prometheus.Labels{"code": "2xx"}),
		newFrontendMetric("http_responses_total", "Total of HTTP responses.", prometheus.CounterValue, prometheus.Labels{"code": "3xx"}),
		newFrontendMetric("http_responses_total", "Total of HTTP responses.", prometheus.CounterValue, prometheus.Labels{"code": "4xx"}),
		newFrontendMetric("http_responses_total", "Total of HTTP responses.", prometheus.CounterValue, prometheus.Labels{"code": "5xx"}),
		newFrontendMetric("http_responses_total", "Total of HTTP responses.", prometheus.CounterValue, prometheus.Labels{"code": "other"}),
		newFrontendMetric("http_requests_total", "Total HTTP requests.", prometheus.CounterValue, nil),
		newFrontendMetric("compressor_bytes_in_total", "Number of HTTP response bytes fed to the compressor", prometheus.CounterValue, nil),
		newFrontendMetric("compressor_bytes_out_total", "Number of HTTP response bytes emitted by the compressor", prometheus.CounterValue, nil),
		newFrontendMetric("compressor_bytes_bypassed_total", "Number of bytes that bypassed the HTTP compressor", prometheus.CounterValue, nil),
		newFrontendMetric("http_responses_compressed_total", "Number of HTTP responses that were compressed", prometheus.CounterValue, nil),
		newFrontendMetric("connections_total", "Total number of connections", prometheus.CounterValue, nil),
      
* backendMetrics   

                newBackendMetric("current_queue", "Current number of queued requests not assigned to any server.", prometheus.GaugeValue, nil),
	        newBackendMetric("max_queue", "Maximum observed number of queued requests not assigned to any server.", prometheus.GaugeValue, nil),
	        newBackendMetric("current_sessions", "Current number of active sessions.", prometheus.GaugeValue, nil),
		newBackendMetric("max_sessions", "Maximum observed number of active sessions.", prometheus.GaugeValue, nil),
		newBackendMetric("limit_sessions", "Configured session limit.", prometheus.GaugeValue, nil),
		newBackendMetric("sessions_total", "Total number of sessions.", prometheus.CounterValue, nil),
	        newBackendMetric("bytes_in_total", "Current total of incoming bytes.", prometheus.CounterValue, nil),
		newBackendMetric("bytes_out_total", "Current total of outgoing bytes.", prometheus.CounterValue, nil),
		newBackendMetric("connection_errors_total", "Total of connection errors.", prometheus.CounterValue, nil),
		newBackendMetric("response_errors_total", "Total of response errors.", prometheus.CounterValue, nil),
		newBackendMetric("retry_warnings_total", "Total of retry warnings.", prometheus.CounterValue, nil),
		newBackendMetric("redispatch_warnings_total", "Total of redispatch warnings.", prometheus.CounterValue, nil),
		newBackendMetric("up", "Current health status of the backend (1 = UP, 0 = DOWN).", prometheus.GaugeValue, nil),
		newBackendMetric("weight", "Total weight of the servers in the backend.", prometheus.GaugeValue, nil),
	        newBackendMetric("current_server", "Current number of active servers", prometheus.GaugeValue, nil),
		newBackendMetric("server_selected_total", "Total number of times a server was selected, either for new sessions, or when re-dispatching.", prometheus.CounterValue, nil),
		newBackendMetric("current_session_rate", "Current number of sessions per second over last elapsed second.", prometheus.GaugeValue, nil),
	        newBackendMetric("max_session_rate", "Maximum number of sessions per second.", prometheus.GaugeValue, nil),
		newBackendMetric("http_responses_total", "Total of HTTP responses.", prometheus.CounterValue, prometheus.Labels{"code": "1xx"}),
		newBackendMetric("http_responses_total", "Total of HTTP responses.", prometheus.CounterValue, prometheus.Labels{"code": "2xx"}),
		newBackendMetric("http_responses_total", "Total of HTTP responses.", prometheus.CounterValue, prometheus.Labels{"code": "3xx"}),
	        newBackendMetric("http_responses_total", "Total of HTTP responses.", prometheus.CounterValue, prometheus.Labels{"code": "4xx"}),
		newBackendMetric("http_responses_total", "Total of HTTP responses.", prometheus.CounterValue, prometheus.Labels{"code": "5xx"}),
		newBackendMetric("http_responses_total", "Total of HTTP responses.", prometheus.CounterValue, prometheus.Labels{"code": "other"}),
		newBackendMetric("client_aborts_total", "Total number of data transfers aborted by the client.", prometheus.CounterValue, nil),
		newBackendMetric("server_aborts_total", "Total number of data transfers aborted by the server.", prometheus.CounterValue, nil),
		newBackendMetric("compressor_bytes_in_total", "Number of HTTP response bytes fed to the compressor", prometheus.CounterValue, nil),
		newBackendMetric("compressor_bytes_out_total", "Number of HTTP response bytes emitted by the compressor", prometheus.CounterValue, nil),
		newBackendMetric("compressor_bytes_bypassed_total", "Number of bytes that bypassed the HTTP compressor", prometheus.CounterValue, nil),
		newBackendMetric("http_responses_compressed_total", "Number of HTTP responses that were compressed", prometheus.CounterValue, nil),
		newBackendMetric("http_queue_time_average_seconds", "Avg. HTTP queue time for last 1024 successful connections.", prometheus.GaugeValue, nil),
	        newBackendMetric("http_connect_time_average_seconds", "Avg. HTTP connect time for last 1024 successful connections.", prometheus.GaugeValue, nil),
	        newBackendMetric("http_response_time_average_seconds", "Avg. HTTP response time for last 1024 successful connections.", prometheus.GaugeValue, nil),
	        newBackendMetric("http_total_time_average_seconds", "Avg. HTTP total time for last 1024 successful connections.", prometheus.GaugeValue, nil),
	
