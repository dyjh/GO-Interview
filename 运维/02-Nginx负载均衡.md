# Nginx 负载均衡

## Nginx 的作用

Nginx 常用于：

- 反向代理。
- 负载均衡。
- 静态资源服务。
- TLS 终止。
- 限流限速。
- 网关入口。

在后端系统中，Nginx 通常位于客户端和应用服务之间。

## 基础反向代理

```nginx
server {
    listen 80;
    server_name example.com;

    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

## upstream 负载均衡

```nginx
upstream backend {
    server 10.0.0.1:8080;
    server 10.0.0.2:8080;
}

server {
    listen 80;

    location / {
        proxy_pass http://backend;
    }
}
```

默认策略是轮询。

## 常见负载均衡策略

### 轮询

默认策略，请求依次分配到后端。

```nginx
upstream backend {
    server app1:8080;
    server app2:8080;
}
```

### 权重

适合后端机器配置不同。

```nginx
upstream backend {
    server app1:8080 weight=3;
    server app2:8080 weight=1;
}
```

### ip_hash

根据客户端 IP hash 到固定后端，适合简单会话粘滞。

```nginx
upstream backend {
    ip_hash;
    server app1:8080;
    server app2:8080;
}
```

注意：

- NAT 场景大量用户可能来自同一 IP，导致不均衡。
- 服务扩缩容会影响 hash 结果。
- 更推荐把会话放 Redis，而不是依赖粘滞会话。

### least_conn

把请求转发给当前连接数较少的后端。

```nginx
upstream backend {
    least_conn;
    server app1:8080;
    server app2:8080;
}
```

适合请求处理时间差异较大的场景。

## 健康检查和故障转移

开源 Nginx 常用被动健康检查：

```nginx
upstream backend {
    server app1:8080 max_fails=3 fail_timeout=30s;
    server app2:8080 max_fails=3 fail_timeout=30s;
}
```

如果某个后端连续失败，会在一段时间内被暂时摘除。

主动健康检查通常需要 Nginx Plus 或第三方模块，或者交给网关/注册中心处理。

## 超时配置

```nginx
location / {
    proxy_connect_timeout 3s;
    proxy_send_timeout 10s;
    proxy_read_timeout 10s;
    proxy_pass http://backend;
}
```

配置过短会误伤慢请求，过长会拖垮连接资源。需要结合业务接口 SLA。

## 限流

```nginx
http {
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;

    server {
        location /api/ {
            limit_req zone=api burst=20 nodelay;
            proxy_pass http://backend;
        }
    }
}
```

含义：

- `rate=10r/s`：每秒 10 个请求。
- `burst=20`：允许一定突发。
- `nodelay`：突发请求不排队延迟，超限直接处理/拒绝。

## 配置检查和重载

```bash
nginx -t
nginx -s reload
```

systemd 环境：

```bash
systemctl reload nginx
```

## 常见请求头

- `Host`：原始域名。
- `X-Real-IP`：真实客户端 IP。
- `X-Forwarded-For`：代理链路 IP。
- `X-Forwarded-Proto`：原始协议 http/https。

后端服务获取真实 IP 时要结合可信代理配置，不能盲目信任外部传入的 `X-Forwarded-For`。

## 面试追问

### Nginx 为什么性能高？

事件驱动、异步非阻塞 I/O、多进程模型、内存占用低，适合处理大量并发连接。

### Nginx 负载均衡和注册中心有什么区别？

Nginx 是入口代理和负载均衡器。注册中心负责维护服务实例列表和健康状态。二者可以结合：注册中心动态生成 Nginx upstream，或者由网关直接接入注册中心。

### 如何处理后端服务发布？

常见方式是滚动发布：先摘流量，等待连接处理完成，部署新版本，健康检查通过后再加入 upstream。容器环境中通常由 Kubernetes Service/Ingress 处理。

## 文章导航
上一篇：[01-Docker基础命令](01-Docker基础命令.md)
