# 1、Horizontal Pod Autoscaling (HPA)

* HPA can change the number of replicas of a pod, scale pods to add or remove pod container collections as needed. 
* HPA achieves its goal by checking various metrics to see whether preset thresholds have been met and reacting accordingly. 
* HPA takes care of scaling and distributing pods over the entire available nodes.

## Scaling is done based on metrics. There are three kinds of metrics:

1⃣️ Default per pod resource metrics: 
    
    Like CPU, Memory, these metrics are fetched from the resource metrics API for each
2⃣️ Custom per pod metrics: 
    
    These are such metrics which are not under default category, but scaling is required based on same. it works with raw values, not utilization values.
3⃣️ Object metrics and external metrics: 
      
    a single metric is fetched, which describes Pod. This metric is then compared to the target or threshold value.
    
⚠️ Horizontal Pod Autoscaler is an API resource in the Kubernetes autoscaling API group. The current stable version, which only includes support for CPU autoscaling, can be found in the autoscaling/v1 API version. When you create a Horizontal Pod Autoscaler API object, make sure the name specified is a valid DNS subdomain name.

## Setting Up HPA

1、制作YAML文件

2、部署hpa

3、增加pod负载

4、查看pod set状态
