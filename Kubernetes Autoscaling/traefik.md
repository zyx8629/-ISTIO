开源的边缘路由工具，通过对providers提供的度量API执行监听，实现动态配置路由

## 1、Quick Start

1⃣️ 利用docker-compose.yml 定义一个反向代理服务，其中运行Traefik镜像

    version: '3'

    services:
      reverse-proxy:
        # The official v2 Traefik docker image
        image: traefik:v2.3
        # Enables the web UI and tells Traefik to listen to docker
        command: --api.insecure=true --providers.docker
        ports:
          # The HTTP port
          - "80:80"
          # The Web UI (enabled by --api.insecure=true)
          - "8080:8080"
        volumes:
          # So that Traefik can listen to the Docker events
          - /var/run/docker.sock:/var/run/docker.sock
          
* docker-compose up -d reverse-proxy
* 此时可访问 http://localhost:8080/api/rawdata 可观察当前代理服务是Traefik及API

2⃣️ 发现新服务、建立路由

* 重新编辑docker-compose.yml 文件，增加新的服务 【whoami】

        # ...
      whoami:
        # A container that exposes an API to show its IP address
        image: traefik/whoami
        labels:
          - "traefik.http.routers.whoami.rule=Host(`whoami.docker.localhost`)"

* 在次执行文件启动该服务
* 刷新http://localhost:8080/api/rawdata 可观察whoami服务已经发布，根据上述定义的label，可利用curl可以访问。
* curl -H Host:whoami.docker.localhost http://127.0.0.1
* 此时无论执行几次访问都只能访问一个服务
* 水平扩展容器 docker-compose up -d --scale whoami=2

        Starting traefik_whoami_1 ... 
        Starting traefik_whoami_1 ... done
        Creating traefik_whoami_2 ... done
        
* 此时连续执行 curl -H Host:whoami.docker.localhost http://127.0.0.1 访问，发现可以访问到两个服务，遵循轮询
* 查看 http://localhost:8080/api/rawdata 也能看见新的whoami服务启动
