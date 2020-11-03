# -ISTIO

环境：

Kubernetes1.18

istio1.7


一、安装Istio

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

二、Bookinfo应用

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
         
       kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
       kubectl get gateway
  
   6、查看 INGRESS_HOST 和 INGRESS_PORT 变量，暴露 GATEWAY_URL
       
       export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT  # 此时鲁莽执行并没有什么用，浏览器不可以访问，故查看 istio-ingressgateway 服务的外部IP
       
       kubectl get svc istio-ingressgateway -n istio-system  
       【输出】
       NAME                   TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)                                                                      AGE
       istio-ingressgateway   LoadBalancer   10.96.231.38   <pending>     15021:31880/TCP,80:31966/TCP,443:32273/TCP,31400:31340/TCP,15443:30333/TCP   16h
       【果然还是 LoadBalancer 状态的问题】
       
   7、由于这个版本的istio不知道怎么永久修改 ingressgateway的 type ，只能先默认LB状态。因此，查看开发手册https://istio.io/latest/zh/docs/tasks/traffic-management/ingress/ingress-control/ ，需先使用外部load balancer确定 ingress ip 和 port的方法
       
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
      ![image](https://github.com/zyx8629/-ISTIO/blob/main/images/bookinfo.png)
      可以注意到刷新几次应用的页面，就会看到 productpage 页面中会随机展示 reviews 服务的不同版本的效果（红色、黑色的星形或者没有显示）。reviews 服务出现这种情况是因为我们还没有使用 Istio 来控制版本的路由。
     
   10、应用默认目标规则（未开启TLS）
   
        kubectl apply -f samples/bookinfo/networking/destination-rule-all.yaml
        kubectl get destinationrules -o yaml
        
三、利用Jaeger对Bookinfo进行分布式追踪


      
       
       
