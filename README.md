# -ISTIO

# 环境：

Kubernetes1.18

centos7

# 一、安装Istio

   1、官网下载安装包并解压
   
    mkdir istio   
    cd istio
    wget https://github.com/istio/istio/releases/download/1.7.4/istio-1.7.4-linux-amd64.tar.gz
    tar -xzvf istio-1.7.4-linux-amd64.tar.gz
    cd istio-1.7.4
    [官网说可浏览到]
      install/kubernetes 目录下，有 Kubernetes 相关的 YAML 安装文件   【我没找见这个！！！！】
      samples/ 目录下，有示例应用程序
      bin/ 目录下，包含 istioctl 的客户端文件。istioctl 工具用于手动注入 Envoy sidecar 代理。
      
   2、将 istioctl 客户端路径增加到 path 环境变量中
    
      vim etc/profile
      export PATH=$PATH:/root/istio/istio-1.7.4/bin
      source /etc/profile
      istioctl version
      [提示]
         no running Istio pods in "istio-system"1.7.4
   
   3、安装 demo 配置
      
      istioctl manifest install --set profile=demo
      #需要等一阵
      [提示]
         Detected that your cluster does not support third party JWT authentication. Falling back to less secure first party JWT. See https://istio.io/docs/ops/best-practices/security/#configure-third-party-service-account-tokens for details.
      ✔ Istio core installed                                                                                                                 
      ✔ Istiod installed                                                                                                                     
      ✔ Egress gateways installed                                                                                                            
      ✔ Ingress gateways installed                                                                                                           
      ✔ Installation complete 
   
   4、查看一下 kubectl get svc -n istio-system
   
       *发现 LoadBalancer 的状态是 <pending>，想修改一下配置文件 NodePort 【并不是这样解决看来，因为install/kubernetes 目录没了】
       
       kubectl get pods -n istio-system  【没啥问题，先往下做吧】
       istio-egressgateway-8d84f88b8-zbxfw    1/1     Running   0          98m
       istio-ingressgateway-bd4fdbd5f-55p4z   1/1     Running   0          98m
       istiod-74844f57b-mmxz2                 1/1     Running   0          100m

# 二、Bookinfo应用

   1、进入 istio 安装目录，注入 sidecar，命名空间打标签
   
       kubectl label namespace default istio-injection=enabled
   
   2、kubectl 部署 Bookinfo 应用
   
       kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
   
   3、查看一下服务和 pods 们都 ready 了没
      
       kubectl get services
         NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
         details       ClusterIP   10.96.88.156    <none>        9080/TCP   70s
         kubernetes    ClusterIP   10.96.0.1       <none>        443/TCP    10d
         productpage   ClusterIP   10.110.88.135   <none>        9080/TCP   69s
         ratings       ClusterIP   10.109.170.97   <none>        9080/TCP   70s
         reviews       ClusterIP   10.108.96.26    <none>        9080/TCP   69s
         
       kubectl get pods       #慢，需等一会儿，在此查看全部 pods 都 ready 才行  【注意到以前启动服务 pod 都是一个，但现在是两个】
         NAME                              READY   STATUS            RESTARTS   AGE
         details-v1-558b8b4b76-6jrx4       0/2     PodInitializing   0          84s
         productpage-v1-6987489c74-q2t4h   0/2     PodInitializing   0          82s
         ratings-v1-7dc98c7588-k29qh       0/2     PodInitializing   0          83s
         reviews-v1-7f99cc4496-kjqk9       0/2     PodInitializing   0          82s
         reviews-v2-7d79d5bd5d-nk8rw       0/2     PodInitializing   0          83s
         reviews-v3-7dbcdcbc56-wttk8       0/2     PodInitializing   0          83s
   
   4、要确认 Bookinfo 应用是否正在运行，请在某个 Pod 中用 curl 命令对应用发送请求，例如 ratings：

        kubectl exec -it $(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}') -c ratings -- curl productpage:9080/productpage | grep -o "<title>.*</title>"
        【出现红色的】
      <title>Simple Bookstore App</title>
        【就好了～】
   
   5、为 ingfess 配置IP和端口号，利用端口，实现外部访问
         
       kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml 【真是执行了个寂寞】
       kubectl get gateway
  
   6、查看 INGRESS_HOST 和 INGRESS_PORT 变量，暴露 GATEWAY_URL
       
       export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT  # 此时鲁莽执行并没有什么用，浏览器不可以访问，故查看 istio-ingressgateway 服务的外部IP
       
       kubectl get svc istio-ingressgateway -n istio-system  
       【输出】
       NAME                   TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)                                                                      AGE
       istio-ingressgateway   LoadBalancer   10.96.231.38   <pending>     15021:31880/TCP,80:31966/TCP,443:32273/TCP,31400:31340/TCP,15443:30333/TCP   16h
       【果然还是 LoadBalancer 状态的问题】
       
   7、由于这个版本的istio不知道怎么永久修改 ingressgateway的 type ，只能先默认 LB 状态。因此，查看开发手册https://istio.io/latest/zh/docs/tasks/traffic-management/ingress/ingress-control/ ，需先使用外部load balancer确定 ingress ip 和 port的方法
       
       export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
       export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}')
       
       【此时可以echo输出查看端口号】
       echo $INGRESS_PORT
       31966
       echo $SECURE_INGRESS_PORT
       32273
       
   8、使用Istio Gateway 配置 ingress
   
   a.创建 Istio Gateway：
   
      kubectl apply -f - <<EOF
      apiVersion: networking.istio.io/v1alpha3
      kind: Gateway
      metadata:
        name: bookinfo-gateway
      spec:
        selector:
          istio: ingressgateway # use Istio default gateway implementation
        servers:
        - port:
            number: 80
            name: http
            protocol: HTTP
          hosts:
          - "bookinfo.example.com"
      EOF
      
   b.为通过 Gateway 的入口流量配置路由:
   
      kubectl apply -f - <<EOF
      apiVersion: networking.istio.io/v1alpha3
      kind: VirtualService
      metadata:
        name: bookinfo
      spec:
        hosts:
        - "bookinfo.example.com"
        gateways:
        - bookinfo-gateway
        http:
        - match:
          - uri:
              exact: /productpage
          - uri:
              prefix: /static
          - uri:
              exact: /login
          - uri:
              exact: /logout
          - uri:
              prefix: /api/v1/products
          route:
          - destination:
              host: productpage
              port:
                number: 9080
      EOF
    
   c.使用 curl 访问 bookinfo 服务
    
     curl -I -HHost:bookinfo.example.com http://$INGRESS_HOST:$INGRESS_PORT/api/v1/products/200
     
     【出现】
      HTTP/1.1 200 OK
      content-type: application/json
      content-length: 197
      server: istio-envoy
      date: Tue, 03 Nov 2020 05:56:26 GMT
      x-envoy-upstream-service-time: 70
     【NICE】
     
   9、现在来试试通过浏览器访问 ingress 服务
   
       kubectl apply -f -<<EOF
      > apiVersion: networking.istio.io/v1alpha3
      > kind: Gateway
      > metadata:
      >   name: bookinfo-gateway
      > spec:
      >   selector:
      >     istio: ingressgateway # use istio default controller
      >   servers:
      >   - port:
      >       number: 80
      >       name: http
      >       protocol: HTTP
      >     hosts:
      >     - "*"
      > ---
      > apiVersion: networking.istio.io/v1alpha3
      > kind: VirtualService
      > metadata:
      >   name: bookinfo
      > spec:
      >   hosts:
      >   - "*"
      >   gateways:
      >   - bookinfo-gateway
      >   http:
      >   - match:
      >     - uri:
      >         exact: /productpage
      >     - uri:
      >         prefix: /static
      >     - uri:
      >         exact: /login
      >     - uri:
      >         exact: /logout
      >     - uri:
      >         prefix: /api/v1/products
      >     route:
      >     - destination:
      >         host: productpage
      >         port:
      >           number: 9080
      > EOF

      然后输出 INGRESS_HOST 和 INGRESS_PORT
      echo INGRESS_HOST=$INGRESS_HOST, INGRESS_PORT=$INGRESS_PORT
      
      INGRESS_HOST=192.168.3.13, INGRESS_PORT=31966
      
      浏览器打开
      http://$INGRESS_HOST:$INGRESS_PORT/productpage
      
      【可以打开首页就成功了～】
       可以注意到刷新几次应用的页面，就会看到 productpage 页面中会随机展示 reviews 服务的不同版本的效果（红色、黑色的星形或者没有显示）。
       reviews 服务出现这种情况是因为我们还没有使用 Istio 来控制版本的路由。
       
 ![image](https://github.com/zyx8629/-ISTIO/blob/main/images/bookinfo.png)
      
   10、应用默认目标规则（未开启TLS）
   
        kubectl apply -f samples/bookinfo/networking/destination-rule-all.yaml
        kubectl get destinationrules -o yaml
        
# 三、利用Jaeger对Bookinfo进行分布式追踪

   1、给istio安装jaeger
   
   a.官网下载安装YAML文件
   
      mkdir jaeger
      cd jaeger
      wget https://raw.githubusercontent.com/jaegertracing/jaeger-kubernetes/master/all-in-one/jaeger-all-in-one-template.yml
      
   b.在istio-system命名空间上安装jaeger
   
      kubectl create -n istio-system -f jaeger-all-in-one-template.yml
   
   c.查看一下 istio-system 命名空间下的服务
      
     【出现了jaeger的三个服务】
     jaeger-agent           ClusterIP      None             <none>        5775/UDP,6831/UDP,6832/UDP,5778/TCP                                          2m52s
     jaeger-collector       ClusterIP      10.104.142.103   <none>        14267/TCP,14268/TCP,9411/TCP                                                 2m52s
     jaeger-query           LoadBalancer   10.97.140.81     <pending>     80:31455/TCP                                                                 2m52s
     
 【注】service type 目前有两种，如果使用 gce 的 kubernetes，可以直接使用 LoadBalancer 类型，gce 会自动帮忙生成一个外部ip，做负载均衡；
 
 如果不是在 gce 平台，可以选择使用NodePort的类型，这样会在 node 里面添加一个对外的端口号，可以通过 nodeIP:nodePORT 来访问。
 
 所以目前我的服务器并非支持外部负载均衡器的，因此，不能通过LB状态下的端口直接访问。

   
   2、访问仪jaeger控制面板
   
    kubectl port-forward -n istio-system $(kubectl get pod -n istio-system -l app=jaeger -o jsonpath='{.items[0].metadata.name}') 16686:16686 &
    
    ！！！报错！！！！
    error: error executing jsonpath "{.items[0].metadata.name}": Error executing template: array index out of bounds: index 0, length 0. Printing more information for debugging the template:
	template was:
		{.items[0].metadata.name}
	object given to jsonpath engine was:
		map[string]interface {}{"apiVersion":"v1", "items":[]interface {}{}, "kind":"List", "metadata":map[string]interface {}{"resourceVersion":"", "selfLink":""}}
      error: TYPE/NAME and list of ports are required for port-forward
      See 'kubectl port-forward -h' for help and examples
      
   查看kubectl源码
 
      func (o *PortForwardOptions) Complete(f cmdutil.Factory, cmd *cobra.Command, args []string) error {
         var err error
         if len(args) < 2 {   【发现是参数小于2个报错】
            return cmdutil.UsageErrorf(cmd, "TYPE/NAME and list of ports are required for port-forward")
         }
         
   所以这官网命令行似乎有点问题，我修改了jaeger安装YAML文件中的参数，让 jeager-query 是 NodePort 的状态
   
      jaeger-query           NodePort       10.97.140.81     <none>        80:31455/TCP                                                                 60m
   
   但是浏览器访问 http://localhost:31455 并无界面出现
   
   得知可以直接修改tracing服务的类型，但查找istio中全部服务没发现tracing的痕迹
   
   	kubectl get all -n istio-system
	NAME                                       READY   STATUS    RESTARTS   AGE
	pod/istio-egressgateway-8d84f88b8-mp7sp    1/1     Running   1          2d2h
	pod/istio-ingressgateway-bd4fdbd5f-55p4z   1/1     Running   2          2d16h
	pod/istiod-74844f57b-mmxz2                 1/1     Running   2          2d16h

	NAME                           TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                                                                      AGE
	service/istio-egressgateway    ClusterIP      10.105.145.146   <none>        80/TCP,443/TCP,15443/TCP                                                     2d16h
	service/istio-ingressgateway   LoadBalancer   10.96.231.38     <pending>     15021:31880/TCP,80:31966/TCP,443:32273/TCP,31400:31340/TCP,15443:30333/TCP   2d16h
	service/istiod                 ClusterIP      10.100.251.114   <none>        15010/TCP,15012/TCP,443/TCP,15014/TCP,853/TCP                                2d16h
	service/jaeger-agent           ClusterIP      None             <none>        5775/UDP,6831/UDP,6832/UDP,5778/TCP                                          38h
	service/jaeger-collector       ClusterIP      10.101.142.126   <none>        14267/TCP,14268/TCP,9411/TCP                                                 38h
	service/jaeger-query           LoadBalancer   10.111.135.253   <pending>     80:30860/TCP                                                                 38h
	service/zipkin                 ClusterIP      None             <none>        9411/TCP                                                                     38h

	NAME                                   READY   UP-TO-DATE   AVAILABLE   AGE
	deployment.apps/istio-egressgateway    1/1     1            1           2d16h
	deployment.apps/istio-ingressgateway   1/1     1            1           2d16h
	deployment.apps/istiod                 1/1     1            1           2d16h

	NAME                                             DESIRED   CURRENT   READY   AGE
	replicaset.apps/istio-egressgateway-8d84f88b8    1         1         1       2d16h
	replicaset.apps/istio-ingressgateway-bd4fdbd5f   1         1         1       2d16h
	replicaset.apps/istiod-74844f57b                 1         1         1       2d16h

现在想一定是有什么别的问题，查看了英语版本的开发手册，恍然大悟，https://istio.io/latest/docs/tasks/observability/distributed-tracing/jaeger/ ，为什么和中文版本不一样！！！删掉了刚才安装的jaeger服务，重新来过

	cd /root/istio/istio-1.7.4/samples/addons
	kubectl apply -f jaeger.yaml
	
再次查看服务，出现了tracing
	
	kubectl get svc -n istio-system
	NAME                   TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                                                                      AGE
	istio-egressgateway    ClusterIP      10.105.145.146   <none>        80/TCP,443/TCP,15443/TCP                                                     2d17h
	istio-ingressgateway   LoadBalancer   10.96.231.38     <pending>     15021:31880/TCP,80:31966/TCP,443:32273/TCP,31400:31340/TCP,15443:30333/TCP   2d17h
	istiod                 ClusterIP      10.100.251.114   <none>        15010/TCP,15012/TCP,443/TCP,15014/TCP,853/TCP                                2d17h
	jaeger-agent           ClusterIP      None             <none>        5775/UDP,6831/UDP,6832/UDP,5778/TCP                                          39h
	jaeger-collector       ClusterIP      10.101.142.126   <none>        14267/TCP,14268/TCP,9411/TCP                                                 39h
	jaeger-query           LoadBalancer   10.111.135.253   <pending>     80:30860/TCP                                                                 39h
	tracing                ClusterIP      10.97.28.187     <none>        80/TCP                                                                       67s
	zipkin                 ClusterIP      None             <none>        9411/TCP                                                                     39h

然后修改一下他的TYPE
	
	kubectl get svc -n istio-system
	NAME                   TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                                                                      AGE
	tracing                NodePort       10.99.56.112     <none>        80:30111/TCP                                                                 6s

访问tracing端口号 http://192.168.3.11:30111/jaeger/search 就可以了～

![image](https://github.com/zyx8629/-ISTIO/blob/main/images/jaeger.png)

3、利用 jeager dashboard 监测 bookinfo

不是，什么情况😭谁动了我的网！！

![image](https://github.com/zyx8629/-ISTIO/blob/main/images/404.png)

删除了服务，又执行了一下～

简单执行了一下访问主页的追踪

4、尝试了解每个图和参数都是什么意思

a.监控板可以看见基本信息

![image](https://github.com/zyx8629/-ISTIO/blob/main/images/baseinfo.png)

b.总用时17.19s；该过程一共启用五个服务；产生6词调用(其中小于1s的调用不被下面结构图记录)；8个span

![image](https://github.com/zyx8629/-ISTIO/blob/main/images/timeline%E8%A7%86%E5%9B%BE.png)

c.这里不知道为啥istio-ingressgateway的服务没显示

![image](https://github.com/zyx8629/-ISTIO/blob/main/images/trace%20graph.png)

【知道了，因为 ingressgateway 并不是服务，是一种智能路由】

# 四、将sidecar 注入pod

   1、执行 kube-inject ，并启动 sleep
   
   	istioctl kube-inject -f samples/sleep/sleep.yaml | kubectl apply -f -

   2、 手动注入
   
   	kubectl -n istio-system get configmap istio-sidecar-injector -o=jsonpath='{.data.config}' > inject-config.yaml
	kubectl -n istio-system get configmap istio-sidecar-injector -o=jsonpath='{.data.values}' > inject-values.yaml
 	kubectl -n istio-system get configmap istio -o=jsonpath='{.data.mesh}' > mesh-config.yaml
	
   3、执行
   
   	istioctl kube-inject \
	>     --injectConfigFile inject-config.yaml \
	>     --meshConfigFile mesh-config.yaml \
	>     --valuesFile inject-values.yaml \
	>     --filename samples/sleep/sleep.yaml \
	>     | kubectl apply -f -

   4、检查 pod 状态
   
   	 kubectl get pod  -l app=sleep
	 【提示】
	 NAME                     READY   STATUS             RESTARTS   AGE
	 sleep-7cf44d4ddd-6gtst   1/2     CrashLoopBackOff   9          23m
	
	CrashLoopBackOff ！！！报错！！！ 大事不妙 🤔
	
	【检查 1】kubectl describe pod  -l app=sleep （这个 pod 中有两个 containers）
	
	 问题出现在 ->istio-proxy<-
	    Container ID:  docker://7aaf51cf8aa0a0ba3cb341ce403f74a8257bbf3d2f4fb9e0f757990bee4a241c
	    Image:         docker.io/istio/proxyv2:1.7.4
	    Image ID:      docker-pullable://istio/proxyv2@sha256:17faf9ddc1254ad98cc70fb11fa74043ce2705f3272eace3fa7011a29576c8f1
	    Port:          15090/TCP
	    Host Port:     0/TCP
	    Args:
	      proxy
	      sidecar
	      --domain
	      $(POD_NAMESPACE).svc.cluster.local
	      --serviceCluster
	      sleep.$(POD_NAMESPACE)
	      --proxyLogLevel=warning
	      --proxyComponentLogLevel=misc:error
	      --trust-domain=cluster.local
	      --concurrency
	      2
	    State:          Waiting
	      Reason:       CrashLoopBackOff
	    Last State:     Terminated
	      Reason:       Error
	      Exit Code:    255
	      Started:      Sat, 07 Nov 2020 17:37:24 +0800
	      Finished:     Sat, 07 Nov 2020 17:37:25 +0800
	    Ready:          False
	 
	 【检查 2】kubectl logs sleep-7cf44d4ddd-6gtst -c istio-proxy
	 
	 Error: invalid prometheus scrape configuration: application port is the same as agent port, which may lead to a recursive loop. Ensure pod does not have prometheus.io/port=15020 label, or that injection is not happening multiple times
	 
	 参考： https://github.com/istio/istio/issues/27675 
	 
	 解决：在执行bookinfo实例的时候，把默认命名空间修改为自动注入 sidecar 状态，导致了二次注入，删除sleep服务，直接apply就行了
	 
	kubectl get po
	NAME                              READY   STATUS    RESTARTS   AGE
	details-v1-558b8b4b76-6jrx4       2/2     Running   18         4d9h
	productpage-v1-6987489c74-q2t4h   2/2     Running   18         4d9h
	ratings-v1-7dc98c7588-k29qh       2/2     Running   18         4d9h
	reviews-v1-7f99cc4496-kjqk9       2/2     Running   18         4d9h
	reviews-v2-7d79d5bd5d-nk8rw       2/2     Running   18         4d9h
	reviews-v3-7dbcdcbc56-wttk8       2/2     Running   18         4d9h
	sleep-8f795f47d-nb42k             2/2     Running   0          58s


	 

	




       
       
