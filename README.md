# Kong API Gateway 教程

## 网站推荐

 - [Docker Playground 练习场](https://labs.play-with-docker.com/) 
    - 是Docker 官方提供的测试环境，可以在Docker 官方提供的环境上测试部署docker容器，并以秒级的速度完成部署，可以进行生产部署前的测试
 - [Kong 官方文档](https://docs.konghq.com/?itm_source=website&itm_medium=nav&_ga=2.96762199.919100626.1589503734-1475338618.1587999353) 
    - kong的官方文档，注意要根据生产环境kong的版本选择相应的官方文档
 - [Kong 官方文档第三方翻译v1.1版](https://github.com/qianyugang/kong-docs-cn)
    - kong 文档的第三方翻译，可以作为参考资料来学习，当前时间2020/05此文档只对应了Kong 1.1.版本的官方文档
 - [Kong docker](https://hub.docker.com/_/kong) 
    - Kong Gateway的官方 docker repository， 可以查阅kong docker 的部署文档
 - [Kong docker compose](https://github.com/Kong/docker-kong/tree/master/compose) 
    - 通过 docker compose 来一键部署Kong，此链接为Kong 官方提供的kong 的docker compose模版

## Kong Gateway 简介 

**预计阅读时间： 5分钟**

### 总览
Kong是一个开源，轻量化， 为微服务而设计的API Gateway（网关）

Kong是一个在Nginx中运行的Lua应用程序，并且可以通过lua-nginx模块实现。Kong不是用这个模块编译Nginx，而是与OpenResty一起分发，OpenResty已经包含了lua-nginx-module。OpenResty不是Nginx的分支，而是一组扩展其功能的模块

![Kong 架构](https://docs.konghq.com/assets/images/docs/getting-started-guide/Kong-GS-overview.png)


功能 | 说明
---- | ------
Service | Service object 是Kong Gateway用来引用其管理的上游API和微服务的ID。
Route | Route 定义了 请求request在到达了Kong API Gateway 之后如何发送给对应的Service
Consumers | Consumers 是你的API的最终用户(end user)。Consumer Object 可以帮助你控制谁可以访问你的API，同时也可以通过使用日志插件记录用户流量（traffic）
Admin API | Kong Gateway自带了一个内部的RESTful API，用于管理目的。API命令可以在群集中的任何节点上运行，并且配置将一致地应用于所有节点。
Plugins | 插件提供了用于修改和控制Kong Gateway功能的模块化系统。例如，为了保护你的API，你可能需要一个访问密钥，可以使用key-auth插件进行设置。插件提供了广泛的功能，包括访问控制，缓存，速率限制，日志记录等等。
Rate Limting Plugin | 使用此插件，您可以限制客户端在给定时间内可以发出的HTTP请求数量。
Proxy Caching Plugin | 该插件提供了反向代理缓存实现。它在给定的时间段内根据响应代码，内容类型和请求方法缓存响应实体。
Key Auth plugin | 该插件可让您向Service或Route添加密钥身份验证（也称为API密钥）。
Load Balancing | Kong Gateway提供了两种负载平衡方法：基于DNS的直接方法或使用环形平衡器的方法。在本指南中，您将使用环形平衡器，该平衡器需要配置上游和目标实体。使用这种方法，后端服务的添加和删除由Kong Gateway处理，并且不需要DNS更新。

### 了解Kong Gateway中的流量
默认情况下，Kong Gateway在其配置的代理端口8000和8443上侦听流量。它评估传入的客户端API请求，并将它们路由(Route)到适当的后端API。在路由请求和提供响应时，可以根据需要通过插件应用策略。

例如，在路由请求之前，可能需要客户端进行身份验证。这带来了许多好处，包括：

- Kong 可以通过插件等轻松实现验证功能(authentication)，后端服务不需要自己的验证(authentication)逻辑
- 后端服务Service只会处理有效的请求(valid request)，因此不会浪费机能来处理无效请求
- 所有的请求都会被记录，实现了流量集中监控

![Kong traffic](https://docs.konghq.com/assets/images/docs/getting-started-guide/gateway-traffic.png)

## Kong docker 快捷部署 

**预计阅读时间： 3分钟**
1. 通过docker-compose 部署：
 - `git clone https://github.com/RIC06X/Kong-API-Gateway-` 
 - `cd docker-compose`
 - `docker-compose up -d` 启动服务
 - `docker-compose down` 停止服务
2. 通过docker 部署
 配置数据库
 ``` $bash
 docker run -d --name kong-database \
                -p 5432:5432 \
                -e "POSTGRES_USER=kong" \
                -e "POSTGRES_DB=kong" \
                postgres:9.6
 ```
 kong迁移数据
``` $bash
docker run --rm \
    --link kong-database:kong-database \
    -e "KONG_DATABASE=postgres" \
    -e "KONG_PG_HOST=kong-database" \
    -e "KONG_CASSANDRA_CONTACT_POINTS=kong-database" \
    kong kong migrations bootstrap
```
 启动kong
 ```$bash
 docker run -d --name kong \
    --link kong-database:kong-database \
    -e "KONG_DATABASE=postgres" \
    -e "KONG_PG_HOST=kong-database" \
    -e "KONG_CASSANDRA_CONTACT_POINTS=kong-database" \
    -e "KONG_PROXY_ACCESS_LOG=/dev/stdout" \
    -e "KONG_ADMIN_ACCESS_LOG=/dev/stdout" \
    -e "KONG_PROXY_ERROR_LOG=/dev/stderr" \
    -e "KONG_ADMIN_ERROR_LOG=/dev/stderr" \
    -e "KONG_ADMIN_LISTEN=0.0.0.0:8001, 0.0.0.0:8444 ssl" \
    -p 8000:8000 \
    -p 8443:8443 \
    -p 8001:8001 \
    -p 8444:8444 \
    kong
 ```
 详细部署说明请参考[kong dockerhub官方网站](https://hub.docker.com/_/kong)
 
## Kong 准备工作

默认情况下，Kong侦听以下端口：
`:8000` Kong在该端口上侦听来自客户端的传入HTTP流量，并将其转发到你的上游服务upstream service。
`:8443` Kong在其上侦听传入的HTTPS流量。此端口具有与端口类似的行为:8000，除了它仅期望HTTPS通信。可以通过配置文件禁用此端口。
`:8001` 用于配置Kong的Admin API http监听端口。
`:8444` 用于配置Kong的Admin API HTTPS监听端口。


