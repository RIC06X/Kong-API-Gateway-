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

预计阅读时间： **5分钟**

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

## docker 快捷部署

预计阅读时间： **3分钟**

1. 通过docker-compose 部署：
   1. `git clone https://github.com/RIC06X/Kong-API-Gateway-`
   2. `cd docker-compose`
   3. `docker-compose up -d` 启动服务
   4. `docker-compose down` 停止服务
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

预计阅读时间： **2分钟**

### Kong 端口介绍

在部署你的服务之前，需要确保：

- Kong Gateway 已经安装成功并且正在运行
- 确认Kong Admin API 端口正在监听相应端口

默认情况下，Kong侦听以下端口：

- `:8000` Kong在该端口上侦听来自客户端的传入HTTP流量，并将其转发到你的上游服务upstream service。
- `:8443` Kong在其上侦听传入的HTTPS流量。此端口具有与端口类似的行为:8000，除了它仅期望HTTPS通信。可以通过配置文件禁用此端口。
- `:8001` 用于配置Kong的Admin API http监听端口。
- `:8444` 用于配置Kong的Admin API HTTPS监听端口。

### 验证 Kong Gateway的配置

你可以通过cURL或者HTTPie 向 Kong Admin API 发送请求

查看现有kong配置  `curl -i -X GET http://localhost:8001`

## 部署你的服务Service

预计阅读时间： **5分钟**

在这一章节，你将学习如何用 `Routes` 来公开/暴露你的 `Services` 服务

### 什么是 `Service`，`Routes`

通过配置 Service 和 Route, Kong Gateway可以向客户端公开你想部署的服务。
首先你需要明确定义一个 Service。在 Kong Gateway中， `Service` 是一个代表你外部上游API(external upstream API)或微服务的一个实体(entity)。例如，数据转换微服务，计费API等。

`Service`的主要属性是其**URL**，`Service`通过URL监听请求(request)。

在开始对`Service`发出请求之前，你需要为其添加一条`Route`。`Route`定义了请求到达Kong Gateway后如何（以及是否）发送到`Service`。单个`Service`可以有多个`Routes`。

配置`Service`和`Route`后，你可以开始通过Kong Gateway发出请求。

下图说明了请求和回复如何通过`Route`和`Service`映射到后端API

![route and service flow diagram](https://docs.konghq.com/assets/images/docs/getting-started-guide/route-and-service.png)

### 添加一个`Service`

就本示例而言，您将创建一个指向Mockbin API的服务。Mockbin是一个“回声”型公共网站，可将请求作为响应返回给请求者。该可视化将有助于了解Kong Gateway代理API请求的方式。

Kong Gateway 的RESTful Admin API使用端口`8001`。通过对Admin API发出请求可以实现对Kong Gateway 的配置，包括添加`Route`和`Service`。

定义一个 `Service`, 命名为`example_service`。定义一个URL `http://mockbin.org`

```bash
    curl -i -X POST http://localhost:8001/services \
    --data name=example_service \
    --data url='http://mockbin.org'
 ```

假如`Service`成功创建，你会收到一个 202 成功消息

验证服务的端点
`curl -i http://localhost:8001/services/example_service`

### 添加一个`Route`

为了使`Service`可以通过Kong Gateway访问，您需要为其添加一条`Route`

用客户需要请求的特定路径, 为`Service`(example_service)定义一条`Route`(/mock)

```bash
  $ curl -i -X POST http://localhost:8001/services/example_service/routes \
  --data 'paths[]=/mock' \
  --data 'name=mocking'
```
将返回一条201消息，表明路由已成功创建。

### 确认路由将请求转发到服务

使用Admin API，发出以下命令：

```bash
curl -i -X GET http://localhost:8000/mock
```

### 总结摘要

- 添加了名称example_service为URL的服务http://mockbin.org。
- 添加了名为的路线/mock。
- 这意味着，如果将HTTP请求发送到端口8000（代理端口）上的Kong Gateway节点并且与route匹配/mock，则该请求将发送到http://mockbin.org。
- 抽象一个后端/上游服务，并将您选择的路由放在前端，您现在可以将其路由给客户端以进行请求。

## 保护你的服务--流量限制

预计阅读时间： **7分钟**

流量制使您可以限制您的上游服务从API使用者接收的请求数量，或每个用户可以调用API的频率

### 为什么要使用流量限制

流量限制可保护API免受意外或恶意的过度使用。在没有流量限制的情况下，每个用户都可以根据自己的意愿进行多次请求，这可能导致大量的请求导致其他用户无法访问服务。启用流量限制后，每秒将API调用限制为固定数量的请求。

### 设置流量限制

在端口上调用Admin API 8001并配置插件以在节点上每分钟限制五（5）个请求，这些请求存储在本地和内存中。

```
curl -i -X POST http://localhost:8001/plugins \
--data "name=rate-limiting" \
--data "config.minute=5" \
--data "config.policy=local"
```

### 验证流量限制

要验证流量限制，请从CLI访问API六（6）次，以确认请求受到流量限制。

```
curl -i -X GET http://localhost:8000/mock/request
```

在第6个请求之后，您应该收到429“超出API流量限制”错误

## 通过代理缓存提高性能

预计阅读时间： **3分钟**

Kong Gateway通过缓存提供了快速的性能。代理缓存插件使用反向代理缓存实现来提供这种快速性能。它根据请求方法，可配置的响应代码，内容类型来缓存响应实体，并且可以按消费者或API进行缓存。

缓存实体将存储一段可配置的时间。达到超时后，Kong Gateway会将请求转发到上游，对结果进行缓存并从缓存进行响应，直到超时为止。该插件可以在Redis中将缓存的数据存储在内存中，或提高性能。

### 为什么要使用代理缓存

使用代理缓存，以使上游服务不会因重复的请求而陷入困境，而Kong Gateway可以响应缓存的结果。

### 设置代理缓存插件

在端口 `8001`上调用Admin API并配置插件以启用全局内存缓存，为Content-Type: `application/json`的服务设置30秒的超时时间。

```bash
curl -i -X POST http://localhost:8001/plugins \
--data name=proxy-cache \
--data config.content_type="application/json" \
--data config.cache_ttl=30 \
--data config.strategy=memory
```

### 验证代理缓存

使用Admin API 访问/ mock路由，并记下响应标头

```bash
curl -i -X GET http://localhost:8000/mock/request
```

需要仔细观察`X-Cache-Status`，`X-Kong-Proxy-Latency`以及`X-Kong-Upstream-Latency`

```
 HTTP/1.1 200 OK
 ...
 X-Cache-Key: d2ca5751210dbb6fefda397ac6d103b1
 X-Cache-Status: Miss
 X-Content-Type-Options: nosniff
 ...
 X-Kong-Proxy-Latency: 25
 X-Kong-Upstream-Latency: 37
```

再次访问/ mock路由。

这个时候，注意到的价值观的差异X-Cache-Status，X-Kong-Proxy-Latency和X-Kong-Upstream-Latency。缓存状态为hit，这意味着Kong Gateway直接从缓存中响应请求，而不是将请求代理到上游服务。

此外，请注意响应中的最小延迟，这使Kong Gateway可以提供最佳性能：

```
 HTTP/1.1 200 OK
 ...
 X-Cache-Key: d2ca5751210dbb6fefda397ac6d103b1
 X-Cache-Status: Hit
 ...
 X-Kong-Proxy-Latency: 0
 X-Kong-Upstream-Latency: 1
```

为了更快地进行测试，可以通过调用Admin API删除缓存：

```
curl -i -X DELETE http://localhost:8001/proxy-cache
```

## 保护你的服务 API Authentication

[API Auth](https://docs.konghq.com/getting-started-guide/latest/secure-services/)

## 负载均衡
[负载均衡](https://docs.konghq.com/getting-started-guide/latest/load-balancing/)
