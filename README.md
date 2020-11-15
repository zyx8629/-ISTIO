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
	
# 五、学习一下 ISTIO 的原理

![image](https://github.com/zyx8629/-ISTIO/blob/main/images/istio%E7%BB%93%E6%9E%84%E5%9B%BE.png)

（1）自动注入
	
	「在上一个学习过程中深有体会」
	它的原理是：kube-apiserver 调用管理面组建的 Sidecar-Injector 服务，自动修改应用程序的描述信息并注入 Sidecar。
	由此也可知，pod中的 Sidecar 和业务容器是同时建立的。
	一旦k8s资源被注入Sidecar，就生成一个服务网格。
	

（2）流量拦截【Enovy】

	iptables规则，拦截业务容器的进、出流量到Sidecar 中。
	ps:像个经纪人（Enovy）一样，按照经纪人委员会（控制面）的要求，安排艺人（服务）的全部事宜、办事规则😎

（3）服务发现【Sidecar->Pilot->service list】

	服务发起者的Envoy(Sidecar) 去找 Pilot要目标服务的实例列表。
	
（4）负载均衡【Sidecar->Pilot->service IP】
	
	服务发起者的Envoy(Sidecar) 根据负载均衡策略选择服务实例。
	
（5）流量治理【Pilot】

	Envoy 找 Pilot要流量规则，在拦截到的入、出流量中执行。

（6）访问安全🔐【Pilot下发安全配置，Citadel维护证书和密钥】

	双向认证和通道加密
	
（7）服务遥测【Mixer】  ps：我怎么觉得Mixer这么多余呢，直接传给Galley不好吗🙈

	通信双方向Mixer 上报访问数据，然后Mixer 再发给监控后端。

（8）策略执行【Mixer】

	通过Mixer 连接后端服务来控制服务间的访问。

（9）外部访问

	网格入口处有个网关。
	
# 六、istio 和 jaeger 间交互的实现【调用链跟踪，istio可视化特性的体现】

📝 一般，一个Trace 由多个span 组成，span记录开始时间、结束时间、操作名称及一组标签和日志。

🙆‍♂️ proxy 可以自动生成 span，需要传播相应的HTTP Header，将span正确的关联在一个Trace 中。

	x-request-id

	x-b3-traceid
	
	x-b3-spanid
	
	x-b3-parentspanid
	
	x-b3-sampled

	x-b3-flags
	
	x-ot-span-context 
	
🤩 在前面的Bookinfo 实例中，Istio 采用基于Envoy 的方式与后端跟踪系统Jaeger 整合。Jaeger的服务和pod名称为istio-tracing，采用的Jaeger 镜像是docker.io/jaegertracing/all-in-one， 它包含jaeger-agent、jaeger-collector和jaeger-query这三个组件。其中，Envoy 代理被配置为自动地将跟踪信息发送到jaeger-collector 服务，jaeger-collector 负责将跟踪数据写入存储卷。

🤓【小结】

💁🏻 实际测试过程中可考虑数据存储在永久数据库中，保障数据的存储。

# 七、istio 的流量管理【特指数据面的流量】

👻 流量管理的本质是采用合适的策略控制流量的方向和多少。

🎃 网格发送和接受度额所有流量都通过Envoy进行代理。过程如下：

![image](https://github.com/zyx8629/-ISTIO/blob/main/images/%E6%B5%81%E9%87%8F%E7%AE%A1%E7%90%86%E6%B5%81%E7%A8%8B%E5%9B%BE.jpg)

## 【实验 1】 简单的流量管理

场景描述：用户可以访问 web service，现在有两种版本的 web service （如 Tomcat 和 httpd），想控制80%用户访问前者，20%访问后者。大概实现流程如下：

![image](https://github.com/zyx8629/-ISTIO/blob/main/images/%E6%B5%81%E9%87%8F%E7%AE%A1%E7%90%86proj.jpg)

Step 1: 准备需要的YAML文件

	 touch is-client.yaml
	 touch is-deployment.yaml
	 touch is-vs.yaml
	 
	 a、 is-client.yaml
	 
	  1 apiVersion: apps/v1
	  2 kind: Deployment
	  3 metadata:
	  4   name: client
	  5 spec:
	  6   replicas: 1
	  7   selector:
	  8     matchLabels:
	  9       app: client
	 10   template:
	 11     metadata:
	 12       labels:
	 13         app: client
	 14     spec:
	 15       containers:
	 16       - name: busybox
	 17         image: busybox
	 18         imagePullPolicy: IfNotPresent
	 19         command: ["/bin/sh","-c","sleep 3600"]
	 
         b、is-deployment.yaml
	 
 	  1 apiVersion: apps/v1
	  2 kind: Deployment
	  3 metadata:
	  4   name: httpd
	  5   labels:
	  6     server: httpd
	  7     app: web
	  8 spec:
	  9   replicas: 1
	 10   selector:
	 11     matchLabels:
	 12       server: httpd
	 13       app: web
	 14   template:
	 15     metadata:
	 16       name: httpd
	 17       labels:
	 18         server: httpd
	 19         app: web
	 20     spec:
	 21       containers:
	 22       - name: busybox
	 23         image: busybox
	 24         imagePullPolicy: IfNotPresent
	 25         command: ["/bin/sh","-c","echo 'hello httpd' > /var/www/index.html; httpd -f -p 8080 -h /var/www" ]
	 26 ---
	 27 apiVersion: apps/v1
	 28 kind: Deployment
	 29 metadata:
	 30   name: tomcat
	 31   labels:
	 32     server: tomcat
	 33     app: web
	 34 spec:
	 35   replicas: 1
	 36   selector:
	 37     matchLabels:
	 38       server: tomcat
	 39       app: web
	 40   template:
	 41     metadata:
	 42       name: tomcat
	 43       labels:
	 44         server: tomcat
	 45         app: web
	 46     spec:
	 47       containers:
	 48       - name: tomcat
	 49         image: docker.io/kubeguide/tomcat-app:v1
	 50         imagePullPolicy: IfNotPresent
	 
         c、is-vs.yaml
	 
	  1 apiVersion: v1
	  2 kind: Service
	  3 metadata:
	  4   name: tomcat-svc
	  5 spec:
	  6   selector:
	  7     app: tomcat
	  8   ports:
	  9   - name: http
	 10     port: 8080
	 11     targetPort: 8080
	 12     protocol: TCP
	 13 ---
	 14 apiVersion: v1
	 15 kind: Service
	 16 metadata:
	 17   name: httpd-svc
	 18 spec:
	 19   selector:
	 20     app: httpd
	 21   ports:
	 22   - name: http
	 23     port: 8080
	 24     targetPort: 8080
	 25     protocol: TCP
	 26 ---
	 27 apiVersion: v1
	 28 kind: Service
	 29 metadata:
	 30   name: web-svc
	 31 spec:
	 32   selector:
	 33      app: web
	 34    ports:
	 35    - name: http
	 36      port: 8080
	 37      targetPort: 8080
	 38      protocol: TCP                     

Step 2: apply 全部YAML 文件，并为他们注入sidecar，并检查svc 和port 是否关联

	kubectl apply -f .
	【如果tomcat起不来可能是镜像pull失败，可事先手动拉取到本地】
	【由于bookinfo实验中，设置过默认命名空间自动注入sidecar，所以现在不需要注入了】
	
	【注】kubectl get po 查看是否注入成功
	NAME                              READY   STATUS    RESTARTS   AGE
	client-c565c4f7-spxnf             2/2     Running   7          16h
	httpd-6488b7f8b-5hcp8             2/2     Running   2          14h
	tomcat-75b9f6bf9b-gqkf6           2/2     Running   2          14h

	kubectl get endpoints
	【提示如下，就okk了】
	httpd-svc     10.244.3.27:8080
	tomcat-svc    10.244.1.179:8080 
	web-svc       10.244.1.179:8080,10.244.3.27:8080 
	
	【目前状态是实现了两个服务的轮询，各50%】
	
Step 3: 创建istio资源文件，is-virtualservice.yaml【是一种CRD资源】

	touch is-virtualservice.yaml
	vim is-virtualservice.yaml
	
	  1 apiVersion: networking.istio.io/v1alpha3
	  2 kind: VirtualService
	  3 metadata:
	  4   name: web-svc-vs
	  5 spec:
	  6   hosts:
	  7   - web-svc
	  8   http:
	  9   - route:
	 10     - destination:
	 11         host: tomcat-svc
	 12       weight: 80
	 13     - destination:
	 14         host: httpd-svc
	 15       weight: 20
	 
	kubectl apply -f is-virtualservice.yaml
	kubectl get virtualservices.networking.istio.io  （并不是k8s资源，只应用于istio）
	【提示】
	NAME         GATEWAYS             HOSTS       AGE
	bookinfo     [bookinfo-gateway]   [*]         5d10h
	web-svc-vs                        [web-svc]   84s

Step 4: 登陆client 端，查看成果
	
	kubectl exec -it client-c565c4f7-spxnf -- sh
	
	wget -q -O - http://web-svc:8080
	
	【发现不再执行轮询策略，Tomcat 服务启动的概率更大了】
	
	此时如果修改client服务，不再进行注入的话，再次访问，还是执行轮询的，因此证明协议或规则的实现也只能在同一个网格中执行。

🤓【小结】

💁🏻 virtual service = hosts field + routing rules

💁🏻 hosts field 是在k8s集群内，除了pod之外，可寻址的访问目标。一般该值可以是短域名、全域名、网关名（ingress）、‘*’

💁🏻 routing rules 是可支持http、TCP的，也可执行组合路由；一般由match（匹配条件）和destination（目的地）组成；match还存在优先级，写在前面的高；执行组合路由的状态下也是需要同时满足要求才能进入目的地。具体还很多参考：https://istio.io/latest/docs/reference/config/networking/virtual-service/

💁🏻 如果在集群外想访问集群中资源的时候，需要有Ingress

## 【实验 2】条件路由
	
修改【实验 1】中的路由条件如下所示：

![image](https://github.com/zyx8629/-ISTIO/blob/main/images/%E6%B5%81%E9%87%8F%E7%AE%A1%E7%90%86pj2.jpg)

Step 1： 关闭【实验 1】中的VS服务，写一个新的带有如上图所示匹配条件的VS文件,并执行

	a、is-virtualservice-with-condition.yaml
	
	  1 apiVersion: networking.istio.io/v1alpha3
	  2 kind: VirtualService
	  3 metadata:
	  4   name: web-svc-vs
	  5 spec:
	  6   hosts:
	  7   - web-svc
	  8   http:
	  9   - match:
	 10     - headers:
	 11         end-user:
	 12           exact: zyx
	 13     route:
	 14     - destination:
	 15         host: tomcat-svc
	 16   - route:
	 17     - destination:                        
	
	b、kubectl apply -f is-virtualservice-with-condition.yaml
	
Step 2: 先利用client 直接访问
	
	kubectl exec -it client-c565c4f7-spxnf -- sh
	Defaulting container name to busybox.
	Use 'kubectl describe pod/client-c565c4f7-spxnf -n default' to see all of the containers in this pod.
	/ # wget -q -O - http://web-svc:8080
	hello httpd
	/ # wget -q -O - http://web-svc:8080
	hello httpd
	/ # wget -q -O - http://web-svc:8080
	hello httpd
	/ # wget -q -O - http://web-svc:8080
	hello httpd
	/ # wget -q -O - http://web-svc:8080
	hello httpd
	······
	【发现只会访问 httpd 服务】
	
Step 3: 利用header 进行条件匹配，查看client 的访问

	/ # wget -q -O - http://web-svc:8080 --header 'end-user: zyx'
	
	【发先可以访问 tomcat 服务】
	
## 【实验 3】动态管理条件路由

Flagger是一个用于全自动化渐进式完成应用发布的Kubernetes operator，它通过分析Prometheus收集到的监控指标并通过Istio、App Mesh等流量管理技术或工具完成应用的渐进式发布。

描述：实现自动化灰度发布。

过程：用 Flagger对 ad服务的 v2版本进行灰度发布,自动调整流量比例,直到 v2版本全部接管流量,完成灰度发布。

【注】实验开始前已部署开源微服务系统 https://github.com/slzcc/cloud-native-istio

Step 1: 进入 cloud-native-istio/chapter-files/canary-release/ ，部署 ad服务

	kubectl apply -f ad-deployment.yaml  -n weather
	
Step 2: 创建Flagger 的 CRD资源，其中定义了自动化灰度发布的相关参数

	kubectl apply -f auto-canary.yaml  -n weather
	
	【若报错，需要手动修改一下，参考flagger的资源官方样例】
	
	💁🏻auto-canary.yaml

	apiVersion: flagger.app/v1beta1
	kind: Canary
	metadata:
	  name: ad
	  namespace: weather
	spec:
	  # deployment reference
	  targetRef:
	    apiVersion: apps/v1
	    kind: Deployment
	    name: ad
	  # the maximum time in seconds for the canary deployment
	  # to make progress before it is rollback (default 600s)
	  progressDeadlineSeconds: 60
	  service:
	    # container port
	    port: 3003
	    # Istio virtual service host names (optional)
	    hosts:
	    - ad
	  analysis:
	    # schedule interval (default 60s)
	    interval: 40s
	    # max number of failed metric checks before rollback
	    threshold: 3
	    # max traffic percentage routed to canary
	    # percentage (0-100)
	    maxWeight: 100
	    # canary increment step
	    # percentage (0-100)
	    stepWeight: 20
	    metrics:
	    - name: "request error rate"
	      threshold: 5
	      query: |
		100 - sum(rate(istio_requests_total{
		     reporter="destination",
		     destination_service_namespace="weather",
		     destination_workload="ad",
		     response_code="200"
		  }[30s]
		))/sum(rate(istio_requests_total{
		     reporter="destination",
		     destination_service_namespace="weather",
		     destination_workload="ad",
		  }[30s]
		)) * 100

Step 3: 进入容器[frontend-v1-fb4f47456-9vqg8]内部，对ad服务发起连续请求，时间间隔为1s：

	kubectl -n weather exec -it frontend-v1-fb4f47456-9vqg8 bash
	
	root@frontend-v1-fb4f47456-9vqg8:/app# for i in 'seg 1 1000'; do curl http://ad.weather:3003/ad --silent --w "Status: %{http_code}\n" -o /dev/null ;sleep 1;done 

Step 4: 执行更新 ad服务的 Deployment的镜像，触发对v2版本的灰度发布任务：
	
	kubectl -n weather set image deployment/ad ad=istioweather/advertisement:v2 

Sept 5: 查看结果

	kubectl describe canary ad -n weather 
	
	【提示】
	
	  Status:
	  Canary Weight:  20
	  Conditions:
	    Last Transition Time:  2020-11-15T03:09:01Z
	    Last Update Time:      2020-11-15T03:09:01Z
	    Message:               New revision detected, progressing canary analysis.
	    Reason:                Progressing
	    Status:                Unknown
	    Type:                  Promoted
	  Failed Checks:           1
	  Iterations:              0
	  Last Applied Spec:       7f7db87c66
	  Last Transition Time:    2020-11-15T03:10:21Z
	  Phase:                   Progressing
	  Tracked Configs:
	Events:
	  Type     Reason  Age                From     Message
	  ----     ------  ----               ----     -------
	  Warning  Synced  12m                flagger  ad-primary.weather not ready: waiting for rollout to finish: observed deployment generation less then desired generation
	  Normal   Synced  11m (x2 over 12m)  flagger  all the metrics providers are available!
	  Normal   Synced  11m                flagger  Initialization done! ad.weather
	  Normal   Synced  101s               flagger  New revision detected! Scaling up ad.weather
	  Normal   Synced  1m                 flagger  Starting canary analysis for ad.weather
	  Normal   Synced  1m                 flagger  Advance ad.weather canary weight 20
	  Normal   Synced  61s                flagger  Advance ad.weather canary weight 40
	  Normal   Synced  40s                flagger  Advance ad.weather canary weight 80
	  ...

如果检测到的 Metrics值始终低于设定的门限值, Flagger就会按照设定的步长(20%）逐步增加v2版本的流量比例。在达到100%后, Flagger会将 ad-primary的 Deployment的镜像改为v2,删掉临时的 Deployment,完成对v2版本的灰度发布。

# 八、istio非侵入流量治理

## 8.1 流量治理的原理

![image](https://github.com/zyx8629/-ISTIO/blob/main/images/%E6%B5%81%E9%87%8F%E6%B2%BB%E7%90%86%E6%B5%81%E7%A8%8B.png)

## 8.2 Istio路由规则配置：VirtualService

istio的VirtualService和DestinationRule，是用来进行流量控制的，但是istio用它们具体做什么事情呢？对envoy产生了那些影响呢？

VirtualService是用来修改route规则的，这样就可以将流量转发到不同的cluster。

DestinationRule是用来改写cluster的，不同的cluster有响应的endpoint，这样方便route。

## 8.3 Istio目标规则配置：DestinationRule

	1.TrafficPolicy
	2.LoadBalancerSettings  [考虑算法替换]
	3.ConnectiobPoolSettings
	4.OutlierDetection  [驱逐不健康实例]
	5.PortTrafficPolicy [端口上的流量策略会覆盖全局]
	6.subset [可规定连接数]
	
😈 典型应用：

	定义subset 
	服务熔断：如果对故障服务进行隔离，也可以有效进行负载优化
	负载均衡配置：端口或者subset都可配置
	TLS认证配置：双向认证

##8.4 Istio服务网关配置：Gateway

	Gateway 一般和 VirtualServics 配合使用，定以外部怎样访问。
	
😈 典型应用：
	
	1、将网格内部的HTTP 服务发布为HTTP 外部访问
	2、将网格内的HTTPS 服务发布为HTTPS 外部访问
	3、将网格内的HTTP 服务发布为HTTPS 外部访问
	4、将网格内的HTTP 服务发布为双向HTTPS 外部访问（身份校验）
	5、将网格内的HTTP服务发布为HTTPS外部访问和HTTPS内部访问
	
# 九、微服务故障处理 🚔

## 9.1 部署prometheus 和 grafana，实现服务指标可视化。

step 1: 启动 istio中的 prometheus 和 grafana 服务
	
	cd /root/istio/istio-1.7.4/samples/addons  【1.7版本的istio安装时默认不启动这两个服务，需要手动发布，安装文件在此目录下】
	kubectl apply -f prometheus.yaml
	kubectl apply -f grafana.yaml
	
	kubectl get svc -n istio-system
	[提示]
	NAME                   TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                                                                      AGE
	grafana                ClusterIP      10.103.81.71     <none>        3000/TCP                                                                     21s
	prometheus             ClusterIP      10.109.211.201   <none>        9090/TCP                                                                     12m
	
	配置文件里修改 TYPE为 NodePort 重新执行apply
	prometheus             NodePort       10.109.211.201   <none>        9090:30046/TCP      
	grafana                NodePort       10.103.81.71     <none>        3000:30190/TCP    
	
	执行：	
	istioctl dashboard prometheus
	istioctl dashboard grafana 
	
	此时访问 http://192.168.3.11:30046/graph 和 http://192.168.3.11:30190/ 即可显示Dashboard

Step 2: 利用 prometheus 进行查询
	 
prometheus存储的是时序数据，即按相同时序(相同名称和标签)，以时间维度存储连续的数据的集合。

![image](https://github.com/zyx8629/-ISTIO/blob/main/images/P.png)

其他查询：

请求 productpage 服务的总次数：
	
	istio_requests_total{destination_service="productpage.default.svc.cluster.local"}
	
请求 reviews 服务 V3 版本的总次数：

	istio_requests_total{destination_service="reviews.default.svc.cluster.local", destination_version="v3"}

过去 5 分钟 productpage 服务所有实例的请求频次：

	rate(istio_requests_total{destination_service=~"productpage.*", response_code="200"}[5m])
	
Step 3:利用Grafana 进行监测

![image](https://github.com/zyx8629/-ISTIO/blob/main/images/G.png)

## 9.2 使用「chaosblade」进行故障的注入

Chaosblade Operator 是混沌工程实验工具 ChaosBlade 下的一款面向云原生领域的混沌实验注入工具，可单独部署使用。通过定义 Kubernetes CRD 来管理混沌实验，每个实验都有非常明确的执行状态。工具具有部署简单、执行便捷、标准化实现、场景丰富等特点。将 ChaosBlade 混沌实验模型与 Kubernetes CRD 很好的结合在一起，可以实现基础资源、应用服务、容器等场景在 Kubernetes 平台上场景复用，方便了 Kubernetes 下资源场景的扩展，而且可通过 chaosblade cli 统一执行调用。

Step 1: 下载 chaosblade-operator ，利用 helm v3 方式安装

1、安装helm v3
	
	下载helm v3.0.2，地址https://get.helm.sh/helm-v3.0.2-linux-amd64.tar.gz。
	tar zxvf helm-v3.0.2-linux-amd64.tar.gz
	mv linux-amd64/helm /usr/local/bin/helm
	helm version
	
2、安装chaosblade-operator
		
	wget https://chaosblade.oss-cn-hangzhou.aliyuncs.com/agent/github/0.8.0/chaosblade-operator-0.8.0-v3.tgz
	helm install chaosblade-operator chaosblade-operator-0.8.0-v3.tgz --namespace kube-system
	
	kubectl get pod -l part-of=chaosblade -n kube-system
	NAME                                  READY   STATUS    RESTARTS   AGE
	chaosblade-operator-599d9684b-vkhtj   1/1     Running   0          9m53s

3、编辑故障文件loss-node-network-by-names.yaml，设置 bookinfo首页访问造成 60% 丢包率，并执行

	  1 apiVersion: chaosblade.io/v1alpha1
	  2 kind: ChaosBlade
	  3 metadata:
	  4   name: loss-node-network-by-names
	  5 spec:
	  6   experiments:
	  7   - scope: node
	  8     target: network
	  9     action: loss
	 10     desc: "node network loss"
	 11     matchers:
	 12     - name: names
	 13       value: ["cnu3.192.168.3.13"]
	 14     - name: namespace
	 15       value:
	 16       - "default"
	 17     - name: percent
	 18       value: ["60"]
	 19     - name: interface
	 20       value: ["eth0"]
	 21     - name: local-port
	 22       value: ["31966"]
	
	查看执行情况 
	
	kubectl get blade loss-node-network-by-names -o json  
	
	【提示不成功。。。。】
		{
	    "apiVersion": "chaosblade.io/v1alpha1",
	    "kind": "ChaosBlade",
	    "metadata": {
		"annotations": {
		    "preSpec": "{\"experiments\":[{\"scope\":\"node\",\"target\":\"network\",\"action\":\"loss\",\"desc\":\"node network loss\",\"matchers\":[{\"name\":\"names\",\"value\":[\"192.168.3.13\"]},{\"name\":\"namespace\",\"value\":[\"default\"]},{\"name\":\"percent\",\"value\":[\"60\"]},{\"name\":\"interface\",\"value\":[\"eth0\"]},{\"name\":\"local-port\",\"value\":[\"31966\"]}]}]}"
		},
		"creationTimestamp": "2020-11-12T08:51:33Z",
		"finalizers": [
		    "finalizer.chaosblade.io"
		],
		"generation": 4,
		"managedFields": [
		    {
			"apiVersion": "chaosblade.io/v1alpha1",
			"fieldsType": "FieldsV1",
			"fieldsV1": {
			    "f:metadata": {
				"f:annotations": {
				    "f:preSpec": {}
				},
				"f:finalizers": {
				    ".": {},
				    "v:\"finalizer.chaosblade.io\"": {}
				}
			    },
			    "f:status": {
				".": {},
				"f:expStatuses": {},
				"f:phase": {}
			    }
			},
			"manager": "chaosblade-operator",
			"operation": "Update",
			"time": "2020-11-12T10:19:43Z"
		    },
		    {
			"apiVersion": "chaosblade.io/v1alpha1",
			"fieldsType": "FieldsV1",
			"fieldsV1": {
			    "f:metadata": {
				"f:annotations": {}
			    },
			    "f:spec": {
				".": {},
				"f:experiments": {}
			    }
			},
			"manager": "kubectl-client-side-apply",
			"operation": "Update",
			"time": "2020-11-12T10:19:43Z"
		    }
		],
		"name": "loss-node-network-by-names",
		"resourceVersion": "1420877",
		"selfLink": "/apis/chaosblade.io/v1alpha1/chaosblades/loss-node-network-by-names",
		"uid": "455367df-ed88-4497-8d20-a7132ac96f51"
	    },
	    "spec": {
		"experiments": [
		    {
			"action": "loss",
			"desc": "node network loss",
			"matchers": [
			    {
				"name": "names",
				"value": [
				    "cnu3.192.168.3.13"
				]
			    },
			    {
				"name": "namespace",
				"value": [
				    "default"
				]
			    },
			    {
				"name": "percent",
				"value": [
				    "60"
				]
			    },
			    {
				"name": "interface",
				"value": [
				    "eth0"
				]
			    },
			    {
				"name": "local-port",
				"value": [
				    "31966"
				]
			    }
			],
			"scope": "node",
			"target": "network"
		    }
		]
	    },
	    "status": {
		"expStatuses": [
		    {
			"action": "loss",
			"error": "can not find the nodes",
			"scope": "node",
			"state": "Error",
			"success": false,
			"target": "network"
		    }
		],
		"phase": "Error"
	    }
	}
	
今天没弄明白🤔

## 9.3 Istio 自己的Fault Injection

检查bookinfo是否启用的是默认路由

kubectl get destinationrules -o yaml 【输出当前执行的路由规则文件】

1、注入HTTP延迟故障：为jason 用户的reviews的v1 服务和 ratings服务间注入 7s 延迟，执行故障文件

	kubectl apply -f samples/bookinfo/networking/virtual-service-ratings-test-delay.yaml

2、打开网页开发者模式查看结果

![image](https://github.com/zyx8629/-ISTIO/blob/main/images/%E5%BB%B6%E6%97%B67s.png)

发现jason只能访问第一版本的reviews，并看到了bug，但是不清楚怎么看出7s延时


## 9.3 基于grafana里的alert功能实现动态报警



 
# 十、istio数据持久化


	
