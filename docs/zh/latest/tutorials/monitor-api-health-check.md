---
title: 使用 Prometheus 进行 API 健康检查
keywords:
  - API 健康检查
  - 使用 Prometheus 进行健康检查
  - API Gateway
description: 本文将介绍如何在 APISIX 中启用 Prometheus 进行 API 健康检查。
---

<!--
#
# Licensed to the Apache Software Foundation (ASF) under one or more
# contributor license agreements.  See the NOTICE file distributed with
# this work for additional information regarding copyright ownership.
# The ASF licenses this file to You under the Apache License, Version 2.0
# (the "License"); you may not use this file except in compliance with
# the License.  You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#
-->

[APISIX](https://apisix.apache.org/zh/) 中存在[健康检查](https://apisix.apache.org/zh/docs/apisix/tutorials/health-check/)机制，可以主动检查上游节点的健康状态。APISIX 通过[插件](https://apisix.apache.org/zh/docs/apisix/plugins/prometheus/)来整合 [Prometheus](https://prometheus.io/)，并暴露上游节点（APISIX 管理的同一后端API服务的多个实例）的健康检查指标给 Prometheus。一般来说，我们通过 URL **`/apisix/prometheus/metrics`** 来暴露指标。

通过本教程，我们将引导你如何在 APISIX 中启用 Prometheus 进行 API 健康检查。

## 前置条件
- 开始之前，你最好已经初步了解 APISIX。熟悉 [API gateway](https://apisix.apache.org/zh/docs/apisix/terminology/api-gateway/)，及其关键概念，比如：[routes](../terminology/route.md)、[upstream](../terminology/upstream.md)、[Admin API](../admin-api.md)、[plugins](../terminology/plugin.md) 和 HTTP 协议也将对你学习本教程有好处。
- 部署 [Docker](https://docs.docker.com/get-docker/) 用来安装 etcd 和 APISIX。
- 安装 [cURL](https://curl.se/) 用来向服务发送请求，以便验证结果。

## 启动 APISIX 示例项目

该示例项目利用了一个预定义的 [Docker Compose 配置文件](https://github.com/apache/apisix-docker/blob/master/example/docker-compose.yml) ，通过一条命令来安装并启动 APISIX、etcd、Prometheus 和其他的服务。从 GitHub 下载仓库 [apisix-docker](https://github.com/apache/apisix-docker) 并用你习惯的编辑器打开，切换到 `/example` 目录，通过运行 `docker compose up` 来启动项目。

当你启动该项目时，Docker 会下载所需要的镜像。你可以在文件 [docker-compose.yaml](https://github.com/apache/apisix-docker/blob/master/example/docker-compose.yml) 中查看所有服务列表。

## 在上游添加健康检查 API 端点

为了定期检查 API 健康状态，APISIX 需要上游服务的健康检查的端点路径。所以，你首先需要为后端服务添加 `/health` 端点。通过这些端点，你可以检查服务的重要指标，比如内存使用情况、数据库连接情况以及响应时间等等。上述启动的示例项目中，已经有了两个 REST API 服务web1和web2，他们已经配置了**健康检查**端点，URL路径为 `/health`。这种情况下，你不需要另外再进行配置。当然，你也可以替换为自己的后端服务。

> 验证服务状态最简单和标准化的方法是定义一个新的[健康检查](https://datatracker.ietf.org/doc/html/draft-inadarei-api-health-check)端点，比如 `/health` 或 `/status`。

## 在 APISIX 中设置健康检查

APISIX提供两种健康检查方式，分别是主动检查和被动检查。查看更多关于健康检查的内容以及如何启用健康检查可以[点击此处](health-check.md)。可以使用 [Admin API](../admin-api.md) 来创建上游对象。以下是通过 Admin API 创建 [Upstream](../terminology/upstream.md) 的示例，该上游具有两个节点（即上述定义的后端服务），并为上游配置了健康检查参数。

```bash
curl "http://127.0.0.1:9180/apisix/admin/upstreams/1" -H "X-API-KEY: edd1c9f034335f136f87ad84b625c8f1" -X PUT -d '
{
   "nodes":{
      "web1:80":1,
      "web2:80":1
   },
   "checks":{
      "active":{
         "timeout":5,
         "type":"http",
         "http_path":"/health",
         "healthy":{
            "interval":2,
            "successes":1
         },
         "unhealthy":{
            "interval":1,
            "http_failures":2
         }
      }
   }
}'
```

这个示例配置了一个主动检查节点 **`/health`**。**一次成功的健康检查**后，认为该节点为健康，**连续两次失败的健康检查**后，认为该节点为不健康。

> 请注意，如果是在 docker 网络之外运行服务，你可能需要上游节点的 IP 地址，而不是它们的域名（web1 和 web2）。只有上游节点数（即已解析的 IP 数）大于1时，才会启动健康检查。

## 启用 Prometheus 插件

你可以创建一个全局规则来为所有路由启用 `prometheus` 插件，配置规则时在插件选项中添加 `"prometheus": {}`。APISIX 收集内部运行指标，默认通过端口 `9091` 和路径 `/apisix/prometheus/metrics` 来暴露这些指标。也可以修改 Prometheus 配置文件 `/prometheus_conf/prometheus.yml`，来自定义输出端口、URI路径

Create a global rule to enable the `prometheus` plugin on all routes by adding `"prometheus": {}` in the plugins option. APISIX gathers internal runtime metrics and exposes them through port `9091` and URI path `/apisix/prometheus/metrics` by default that Prometheus can scrape. It is also possible to customize the export port and **URI path**, **add** **extra labels, the frequency of these scrapes, and other parameters** by configuring them in the Prometheus configuration `/prometheus_conf/prometheus.yml`file.

```bash
curl "http://127.0.0.1:9180/apisix/admin/global_rules" -H "X-API-KEY: edd1c9f034335f136f87ad84b625c8f1" -X PUT -d '
{
   "id":"rule-for-metrics",
   "plugins":{
      "prometheus":{
      }
   }
}'
```

## Create a Route

Create a [Route](https://apisix.apache.org/docs/apisix/terminology/route/) object to route incoming requests to upstream nodes:

```bash
curl "http://127.0.0.1:9180/apisix/admin/routes/1" -H "X-API-KEY: edd1c9f034335f136f87ad84b625c8f1" -X PUT -d '
{
   "name":"backend-service-route",
   "methods":[
      "GET"
   ],
   "uri":"/",
   "upstream_id":"1"
}'
```

## Send validation requests to the route

To generate some metrics, you try to send few requests to the route we created in the previous step:

```bash
curl -i -X GET "http://localhost:9080/"
```

If you run the above requests a couple of times, you can see from responses that APISX routes some requests to `node1` and others to `node2`. That’s how Gateway load balancing works!

```bash
HTTP/1.1 200 OK
Content-Type: text/plain; charset=utf-8
Content-Length: 10
Connection: keep-alive
Date: Sat, 22 Jul 2023 10:16:38 GMT
Server: APISIX/3.3.0

hello web2

...

HTTP/1.1 200 OK
Content-Type: text/plain; charset=utf-8
Content-Length: 10
Connection: keep-alive
Date: Sat, 22 Jul 2023 10:16:39 GMT
Server: APISIX/3.3.0

hello web1
```

## Collecting health check data with the Prometheus plugin

Once the health checks and route are configured in APISIX, you can employ Prometheus to monitor health checks. APISIX **automatically exposes health check metrics data** for your APIs if the health check parameter is enabled for upstream nodes. You will see metrics in the response after fetching them from APISIX:

```bash
curl -i http://127.0.0.1:9091/apisix/prometheus/metrics
```

Example Output:

```bash
# HELP apisix_http_requests_total The total number of client requests since APISIX started
# TYPE apisix_http_requests_total gauge
apisix_http_requests_total 119740
# HELP apisix_http_status HTTP status codes per service in APISIX
# TYPE apisix_http_status counter
apisix_http_status{code="200",route="1",matched_uri="/",matched_host="",service="",consumer="",node="172.27.0.5"} 29
apisix_http_status{code="200",route="1",matched_uri="/",matched_host="",service="",consumer="",node="172.27.0.7"} 12
# HELP apisix_upstream_status Upstream status from health check
# TYPE apisix_upstream_status gauge
apisix_upstream_status{name="/apisix/upstreams/1",ip="172.27.0.5",port="443"} 0
apisix_upstream_status{name="/apisix/upstreams/1",ip="172.27.0.5",port="80"} 1
apisix_upstream_status{name="/apisix/upstreams/1",ip="172.27.0.7",port="443"} 0
apisix_upstream_status{name="/apisix/upstreams/1",ip="172.27.0.7",port="80"} 1
```

Health check data is represented with metrics label `apisix_upstream_status`. It has attributes like upstream `name`, `ip` and `port`. A value of 1 represents healthy and 0 means the upstream node is unhealthy.

## Visualize the data in the Prometheus dashboard

Navigate to http://localhost:9090/ where the Prometheus instance is running in Docker and type **Expression** `apisix_upstream_status` in the search bar. You can also see the output of the health check statuses of upstream nodes on the **Prometheus dashboard** in the table or graph view:

![Visualize the data in Prometheus dashboard](https://static.apiseven.com/uploads/2023/07/20/OGBtqbDq_output.png)

## Next Steps

You have now learned how to set up and monitor API health checks with Prometheus and APISIX.  APISIX Prometheus plugin is configured to connect [Grafana](https://grafana.com/) automatically to visualize metrics. Keep exploring the data and customize the [Grafana dashboard](https://grafana.com/grafana/dashboards/11719-apache-apisix/) by adding a panel that shows the number of active health checks.

### Related resources

- [Monitoring API Metrics: How to Ensure Optimal Performance of Your API?](https://api7.ai/blog/api7-portal-monitor-api-metrics)
- [Monitoring Microservices with Prometheus and Grafana](https://api7.ai/blog/introduction-to-monitoring-microservices)

### Recommended content

- [Implementing resilient applications with API Gateway (Health Check)](https://dev.to/apisix/implementing-resilient-applications-with-api-gateway-health-check-338c)
