# MinIO 容器 unhealthy 问题完整排查与解决方案

## 一、问题现象

1. 执行 `docker compose ps` 查看状态，`milvus-minio` 服务持续显示 `unhealthy`；
2. 宿主机通过 `curl` 访问 MinIO 健康接口 `http://localhost:9000/minio/health/live`，返回 200 OK，业务功能实际可用；
3. 容器日志无服务崩溃、启动失败类报错，仅存在密钥配置弃用警告。

## 二、根因分析

### 核心原因：健康检查命令执行失败

MinIO 官方镜像为精简镜像，容器内部未预装 `curl` 工具。
原 `docker-compose.yml` 中使用 `curl` 作为健康检查命令，容器内找不到该指令，命令执行失败（退出码 127），导致 Docker 判定服务不健康。

> 关键认知：Docker 健康检查命令在**容器内部**执行，与宿主机环境完全隔离，宿主机可用的工具不代表容器内存在。

### 次要问题：配置变量弃用

原配置使用 `MINIO_ACCESS_KEY` / `MINIO_SECRET_KEY` 配置密钥，该组环境变量在新版 MinIO 中已被废弃，会持续输出警告日志。

### 常见误区

`minio ready` 不是有效命令：`ready` 是 MinIO 客户端 `mc` 的子命令，而非 MinIO 服务端命令，直接执行会失败。

## 三、分步排查流程

### 1. 查看容器启动日志

```bash
docker logs milvus-minio
```


排查结果：服务正常启动、端口监听正常，仅提示密钥变量弃用，无服务运行异常。

### 2. 宿主机验证服务可用性

```bash
curl -I http://localhost:9000/minio/health/live
```

排查结果：返回 `HTTP/1.1 200 OK`，证明 MinIO 业务功能本身正常。

### 3. 进入容器验证工具缺失

```bash
# 进入容器内部
docker exec -it milvus-minio sh

# 测试 curl 是否存在
curl
```

排查结果：提示 `curl: command not found`，确认容器内无 curl 工具，健康检查命令无法执行。

### 4. 验证官方内置健康检查方式

```bash
# 容器内执行：使用镜像自带的 mc 客户端检查集群状态
mc ready local
```

排查结果：输出 `The cluster is ready`，命令执行成功，可作为新的健康检查规则。

## 四、最终解决方案

修改 `docker-compose.yml` 中 minio 服务的配置，替换健康检查命令与密钥环境变量，然后重建容器生效。

### 原始问题配置

```yaml
minio:
  container_name: milvus-minio
  image: minio/minio:RELEASE.2024-05-28T17-19-04Z
  environment:
    MINIO_ACCESS_KEY: minioadmin
    MINIO_SECRET_KEY: minioadmin
  ports:
    - "9001:9001"
    - "9000:9000"
  volumes:
    - ${DOCKER_VOLUME_DIRECTORY:-.}/volumes/minio:/minio_data
  command: minio server /minio_data --console-address ":9001"
  healthcheck:
    test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
    interval: 30s
    timeout: 20s
    retries: 3
```

### 修正后配置

```yaml
minio:
  container_name: milvus-minio
  image: minio/minio:RELEASE.2024-05-28T17-19-04Z
  environment:
    # 替换为新版变量，消除弃用警告
    MINIO_ROOT_USER: minioadmin
    MINIO_ROOT_PASSWORD: minioadmin
  ports:
    - "9001:9001"
    - "9000:9000"
  volumes:
    - ${DOCKER_VOLUME_DIRECTORY:-.}/volumes/minio:/minio_data
  command: minio server /minio_data --console-address ":9001"
  # 使用镜像自带 mc 客户端做健康检查，无需额外依赖
  healthcheck:
    test: ["CMD", "mc", "ready", "local"]
    interval: 10s
    timeout: 5s
    retries: 5
    start_period: 20s # 启动缓冲期，容器刚启动时暂不执行检查
```

### 重建容器生效

```bash
# 重新启动 minio 服务（加载新配置）
docker compose up -d minio
```

## 五、结果验证

等待 20~30 秒后，执行状态查看命令：

```bash
docker compose ps
```

`milvus-minio` 状态变为 `Up xx minutes (healthy)`，问题解决。

## 六、避坑总结

1. **环境隔离原则**：Docker 健康检查运行在容器内部，宿主机的工具、网络环境不能等同于容器内部；
2. **优先使用内置工具**：MinIO 精简镜像默认不带 `curl`/`wget`，应优先使用官方自带的 `mc` 客户端做健康检查；
3. **命令区分**：`minio` 是服务端指令，`mc` 是 MinIO 客户端指令，`ready` 仅属于 `mc` 命令；
4. **版本兼容性**：新版 MinIO 统一使用 `MINIO_ROOT_USER` / `MINIO_ROOT_PASSWORD` 配置密钥，旧变量已废弃；
5. **状态判断**：容器显示 `unhealthy` 不代表服务一定不可用，优先通过日志、端口访问、容器内命令逐层验证功能。
