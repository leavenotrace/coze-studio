# Coze Studio 部署问题修复报告

## 修复日期

2025-11-09

## 问题概述

Coze Studio 在使用 Docker Compose 部署后，前端无法正常访问 API，出现 301 重定向、404 和 500 错误。

---

## 问题 1: Nginx 301 重定向

### 症状

```bash
curl -H "Host: coze.happygpts.cn" http://localhost/
# 返回: 301 Moved Permanently
```

### 根本原因

`docker/nginx/conf.d/default.conf` 中存在两个冲突的 `Cache-Control` 头配置：

```nginx
add_header Cache-Control "no-cache";
add_header Cache-Control "public, max-age=3600";
```

### 解决方案

删除冲突的 `no-cache` 配置，只保留一个：

```nginx
add_header Cache-Control "public, max-age=3600";
```

### 修改文件

- `docker/nginx/conf.d/default.conf`

---

## 问题 2: API 请求返回 404

### 症状

```
Request URL: https://coze.happygpts.cn/api/passport/web/email/login/
Status Code: 404 Not Found
```

### 根本原因

项目使用 `jwilder/nginx-proxy` 作为反向代理，但存在以下问题：

1. nginx-proxy 默认只代理静态文件到 `coze-web` 容器
2. 没有配置 API 路径（`/api`, `/v1`, `/v2`, `/v3`, `/admin`）到后端服务 `coze-server`
3. nginx-proxy 容器不在 `coze-network` 网络中，无法访问后端服务

### 解决方案

#### 1. 创建自定义 vhost 配置

创建 `docker/vhost.d/coze.happygpts.cn` 文件，配置 API 代理：

```nginx
# API proxy to backend - must come before location /
location ~ ^/(api|v[1-3]|admin) {
    proxy_pass http://coze-server:8888;

    proxy_set_header Host $http_host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    # WebSocket support
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";

    # Timeout settings
    proxy_connect_timeout 60s;
    proxy_send_timeout 60s;
    proxy_read_timeout 60s;

    # Replace minio URLs in responses
    sub_filter ':8889' ':8888/local_storage';
    sub_filter 'minio:9000' '$http_host/local_storage';
    sub_filter_once off;
    sub_filter_types 'application/json' 'text/event-stream';
}

# MinIO storage proxy
location /local_storage/ {
    if ($args ~* "(^|.*&)x-wf-file_name=[^&]*(&.*|$)") {
        set $args $1$2;
    }
    if ($args ~* "^x-wf-file_name=[^&]*$") {
        set $args "";
    }

    rewrite ^/local_storage/(.*)$ /$1 break;
    proxy_pass http://minio:9000;
    proxy_set_header Host minio:9000;
}
```

#### 2. 修改 docker-compose.yml

将 nginx-proxy 加入 coze-network，并禁用 HTTPS 强制重定向：

```yaml
nginx-proxy:
  image: jwilder/nginx-proxy:alpine
  container_name: nginx-proxy
  restart: always
  ports:
    - '80:80'
    - '443:443'
  volumes:
    - /var/run/docker.sock:/tmp/docker.sock:ro
    - ./certs:/etc/nginx/certs
    - ./vhost.d:/etc/nginx/vhost.d
    - ./html:/usr/share/nginx/html
  networks:
    - proxy-tier
    - coze-network # 添加这一行

coze-web:
  image: cozedev/coze-studio-web:latest
  container_name: coze-web
  restart: always
  expose:
    - '80'
  environment:
    - VIRTUAL_HOST=coze.happygpts.cn
    - LETSENCRYPT_HOST=coze.happygpts.cn
    - LETSENCRYPT_EMAIL=bob@happyshare.io
    - HTTPS_METHOD=noredirect # 添加这一行，禁用 HTTPS 重定向
  depends_on:
    - coze-server
  networks:
    - coze-network
    - proxy-tier
```

#### 3. 重启服务

```bash
docker compose -f docker/docker-compose.yml up -d nginx-proxy
docker compose -f docker/docker-compose.yml restart nginx-proxy
```

### 修改文件

- `docker/vhost.d/coze.happygpts.cn` (新建)
- `docker/docker-compose.yml`

---

## 问题 3: API 请求返回 500 Internal Server Error

### 症状

```
Request URL: https://coze.happygpts.cn/api/intelligence_api/search/get_draft_intelligence_list
Status Code: 500 Internal Server Error
```

后端日志显示：

```
an error happened during the Search query execution: dial tcp 172.18.0.5:9200: connect: connection refused
```

### 根本原因

Elasticsearch 索引创建失败，原因是：

1. 索引配置需要 `smartcn`（中文分词器）插件
2. Bitnami Elasticsearch 容器默认没有安装该插件
3. 索引分片状态为 UNASSIGNED，导致集群状态为 RED

错误详情：

```
java.lang.IllegalArgumentException: Unknown analyzer type [smartcn] for [smartcn]
```

### 解决方案

#### 1. 删除损坏的索引

```bash
docker compose -f docker/docker-compose.yml exec elasticsearch \
  curl -X DELETE "localhost:9200/project_draft,coze_resource"
```

#### 2. 验证集群健康

```bash
docker compose -f docker/docker-compose.yml exec elasticsearch \
  curl -s "localhost:9200/_cluster/health?pretty"
```

应该返回：

```json
{
  "status": "green",
  "active_shards_percent_as_number": 100.0
}
```

#### 3. 重启后端服务

```bash
docker compose -f docker/docker-compose.yml restart coze-server
```

后端会在首次使用时自动重新创建索引（不使用 smartcn 分词器）。

### 修改文件

无需修改文件，仅需执行命令清理数据

---

## 额外修复: Elasticsearch 文件权限问题

### 症状

Elasticsearch 容器反复重启，日志显示：

```
java.nio.file.AccessDeniedException: /bitnami/elasticsearch/data/node.lock
```

### 解决方案

修复数据目录权限：

```bash
sudo chown -R 1001:1001 docker/data/bitnami/elasticsearch
docker compose -f docker/docker-compose.yml restart elasticsearch
```

---

## 验证修复

### 1. 检查 API 健康状态

```bash
curl -i https://coze.happygpts.cn/api/plugin_api/library_resource_list \
  -X POST -H "Content-Type: application/json" -d '{}'
```

预期响应：

```
HTTP/2 200
{"code":700012006,"msg":"authentication failed: missing session_key in cookie"}
```

### 2. 检查 Elasticsearch 状态

```bash
docker compose -f docker/docker-compose.yml exec elasticsearch \
  curl -s "localhost:9200/_cluster/health?pretty"
```

预期响应：

```json
{
  "status": "green",
  "number_of_nodes": 1,
  "active_shards_percent_as_number": 100.0
}
```

### 3. 检查所有容器状态

```bash
docker compose -f docker/docker-compose.yml ps
```

所有容器应该处于 `Up` 状态。

---

## 总结

### 修改的文件

1. `docker/nginx/conf.d/default.conf` - 删除冲突的 Cache-Control 头
2. `docker/vhost.d/coze.happygpts.cn` - 新建 API 代理配置
3. `docker/docker-compose.yml` - 添加网络配置和环境变量

### 执行的命令

```bash
# 修复 Elasticsearch 权限
sudo chown -R 1001:1001 docker/data/bitnami/elasticsearch

# 删除损坏的索引
docker compose -f docker/docker-compose.yml exec elasticsearch \
  curl -X DELETE "localhost:9200/project_draft,coze_resource"

# 重启服务
docker compose -f docker/docker-compose.yml restart elasticsearch
docker compose -f docker/docker-compose.yml up -d nginx-proxy
docker compose -f docker/docker-compose.yml restart coze-server
```

### 关键技术点

1. **nginx-proxy 工作原理**：通过监听 Docker socket 自动生成反向代理配置
2. **vhost.d 配置**：可以为特定域名添加自定义 nginx 配置
3. **Docker 网络**：容器需要在同一网络中才能通过服务名互相访问
4. **Elasticsearch 分析器**：索引配置需要与已安装的插件匹配

### 最终状态

✅ 前端可以正常访问
✅ API 请求正确路由到后端
✅ Elasticsearch 集群健康
✅ 所有服务正常运行

---

## 注意事项

1. **中文搜索功能**：由于删除了需要 smartcn 的索引，中文搜索可能不如预期精确。如需完整的中文分词支持，需要：

   - 使用支持 smartcn 插件的 Elasticsearch 镜像
   - 或者在现有容器中手动安装插件

2. **HTTPS 配置**：当前禁用了 HTTPS 强制重定向。如需启用 HTTPS：

   - 确保域名 DNS 正确解析
   - 删除 `HTTPS_METHOD=noredirect` 环境变量
   - Let's Encrypt 会自动申请证书

3. **生产环境建议**：
   - 配置 Elasticsearch 副本数为 0（单节点集群）
   - 定期备份数据目录
   - 监控容器资源使用情况


---

## 问题 4: 文件上传 413 错误

### 症状
```
Request URL: https://coze.happygpts.cn/api/bot/upload_file
Status Code: 413 Content Too Large
```

### 根本原因
Nginx 默认的 `client_max_body_size` 限制为 1MB，但后端配置支持最大 1GB 的文件上传（`MAX_REQUEST_BODY_SIZE=1073741824`）。

### 解决方案
在 `docker/vhost.d/coze.happygpts.cn` 文件顶部添加：

```nginx
# Increase upload size limit to 1GB (matching backend MAX_REQUEST_BODY_SIZE)
client_max_body_size 1024M;
```

重新加载 nginx 配置：
```bash
docker compose -f docker/docker-compose.yml exec nginx-proxy nginx -s reload
```

### 修改文件
- `docker/vhost.d/coze.happygpts.cn`

---

## 更新说明

### 2025-11-09 更新
- 添加文件上传大小限制配置（问题 4）
- 将 `client_max_body_size` 设置为 1024M，匹配后端的 1GB 限制
