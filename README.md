# -ISTIO

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
   
   4、查看一下kubectl get svc -n istio-system
   
       *发现 LoadBalancer 的状态是 <pending>，想修改一下配置文件 NodePort 【并不是这样解决看来，因为install/kubernetes 目录没了】
       
       kubectl get pods -n istio-system  【没啥问题，先往下做吧】
       istio-egressgateway-8d84f88b8-zbxfw    1/1     Running   0          98m
       istio-ingressgateway-bd4fdbd5f-55p4z   1/1     Running   0          98m
       istiod-74844f57b-mmxz2                 1/1     Running   0          100m
   
   
       
