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

1、部署 Kubernetes Metrics Server.由于没有科学上网，导致metrics-server镜像没下载成，弄了一下午😭

尝试解决：在本地 pull 镜像，同时也给部署文件增加设置 nodeSelector 将该服务固定在 master 上，但没有啥用呢

2、制作YAML文件 

nginx-hpa.yaml

      1 apiVersion: autoscaling/v2beta2
      2 kind: HorizontalPodAutoscaler
      3 metadata:
      4   name: nginx
      5 spec:
      6   scaleTargetRef:
      7     apiVersion: apps/v1
      8     kind: Deployment
      9     name: nginx
     10   minReplicas: 1
     11   maxReplicas: 10
     12   metrics:
     13   - type: Resource
     14     resource:
     15       name: cpu
     16       target:
     17         type: Utilization
     18         averageUtilization: 50                               

ngnix.yaml

      1  apiVersion: apps/v1
      2 kind: Deployment
      3 metadata:
      4   name: nginx
      5   namespace: zyx-hpa
      6 spec:
      7   replicas: 1
      8   selector:
      9     matchLabels:
     10       app: nginx
     11   template:
     12     metadata:
     13       labels:
     14         app: nginx
     15     spec:
     16       containers:
     17       - name: nginx
     18         image: nginx:1.7.9
     19         ports:
     20         - containerPort: 80
     21         resources:
     22           limits:
     23             cpu: 500m
     24           requests:
     25             cpu: 300m
     26 ---
     27 apiVersion: v1
     28 kind: Service
     29 metadata:
     30   name: nginx
     31   labels:
     32     app: nginx
     33 spec:
     34   type: NodePort
     35   ports:
     36   - port: 80
     37     protocol: TCP
     38     targetPort: 80
     39     name: http
     40     nodePort: 30088
     41   selector:
     42     app: nginx               

is-client.yaml  客户端，用了前面实验的文件

3、部署hpa

kubectl apply -f nginx-hpa.yaml -n zyx-hpa

kubectl get hpa -n zyx-hpa

    NAME    REFERENCE          TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
    nginx   Deployment/nginx   <unknown>/50%   1         10        1          6h28m
    
 ⚠️<unknown>就是因为我的metrics-server服务没起来，所以监测不到，但我今天怎么也没给他部署上
   
4、增加pod负载

进入client内部，向nginx的pod发起连续访问

    kubectl -n zyx-hpa exec -it client-c565c4f7-htthk -- sh
    / # while sleep 0.01; do wget -q -O- http://192.168.3.11:30088; done    

5、查看pod、deployment、hpa状态

应该是要看见pod的动态伸缩的，可惜监测服务有部上

