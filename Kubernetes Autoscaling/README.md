# 1、Horizontal Pod Autoscaling (HPA)

* HPA can change the number of replicas of a pod, scale pods to add or remove pod container collections as needed. 
* HPA achieves its goal by checking various metrics to see whether preset thresholds have been met and reacting accordingly. 
* HPA takes care of scaling and distributing pods over the entire available nodes.

Scaling is done based on metrics. There are three kinds of metrics:

Default per pod resource metrics: 
    
    Like CPU, Memory, these metrics are fetched from the resource metrics API for each
Custom per pod metrics: 
    
    These are such metrics which are not under default category, but scaling is required based on same. it works with raw values, not utilization values.
Object metrics and external metrics: 
      
    a single metric is fetched, which describes Pod. This metric is then compared to the target or threshold value.
    
## Setting Up Kubernetes Autoscaling
