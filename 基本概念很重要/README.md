## 问题 1 ：deploment和service的关系是什么

![image](https://github.com/zyx8629/-ISTIO/blob/main/images/Question-1.jpeg)

举个 🌰 

![image](https://github.com/zyx8629/-ISTIO/blob/main/images/%E6%88%AA%E5%B1%8F2020-11-24%20%E4%B8%8A%E5%8D%8810.04.03.png)

如上图所示，利用 devlopment 发布了一个叫做 petclinic 的应用，在 k8s 集群中，想访问这个应用，利用 pod IP、container IP、虚拟地址就可以实现，但是这个应用是需要支持外部访问的，因为用户是通过外部浏览器进行访问集群内部服务的。同时，由于 pod 存在销毁再创建的机制，IP 是不固定的，又增加了外部访问的难度。因此，需要一个固定的虚拟地址来表示一系列可提供相同服务 pod ，那么 service 就是用来解决这个问题的。其中，会产生一个固定的访问端口，直接暴露这组的服务，还可以规定访问类型（如 LB CIP NP）。而且，Service 和 pods 是通过一个 Selector进行绑定的，为 Service 指定 app：petclinic 即可与 deploment 创建的该应用绑定，通过执行查看 pod 的 endpoint 也可以查看是否绑定成功。
 
## 问题 2 ：apiversion是什么，为什么要设置这个东西

![image](https://github.com/zyx8629/-ISTIO/blob/main/images/Question-2.png)

## 问题 3 ：如何理解kubernetes operator这个概念

![image](https://github.com/zyx8629/-ISTIO/blob/main/images/Question-3.png)

![image](https://github.com/zyx8629/-ISTIO/blob/main/images/flagger-canary-overview.png)

举个 🌰

在前面学习过程中，我使用过一个叫做 flagger 的框架，实现了一下自动化金丝雀发布。整个实现过程就是通过根据官方样例，自定义分析参数、度量指标写一个 CRD.yaml 文件，并执行，那么 flagger 在这次实践中就是一个 operator ，它会自动帮我从 Prometheus 中提取 metric 数据，按设定的公式进行阈值计算，如果该值达到预期的设定，就按权重进行逐步的流量导入，如果不成功则自动退回就版本。这样的过程就是基于一个 operator 实现一个自动化扩展 K8s 资源的操作。

## 问题 4 ：用一个场景对istio流量规则的实现进行描述 

