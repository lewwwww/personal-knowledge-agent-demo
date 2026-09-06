# Dify 本地 Docker 部署详细指南

> 本指南记录了在 Windows / macOS / Linux 上通过 Docker Compose 本地部署 Dify 开源版的完整流程，包括常见坑点与排错方法。

---

## 目录

1. [环境准备](#1-环境准备)
2. [下载配置文件](#2-下载配置文件)
3. [修改配置](#3-修改配置)
4. [启动服务](#4-启动服务)
5. [验证部署](#5-验证部署)
6. [常见坑点与排错](#6-常见坑点与排错)
7. [运维命令](#7-运维命令)

---

## 1. 环境准备

### 硬件要求

| 配置 | 最低 | 推荐 |
|---|---|---|
| CPU | 2 核 | 4 核+ |
| 内存 | 4GB | 8GB+ |
| 磁盘 | 10GB | 20GB+ |

> ⚠️ Dify 由 15 个容器组成，内存低于 4GB 可能频繁崩溃。Windows 用户建议在 Docker Desktop 设置中限制 WSL2 内存上限。

### 软件要求

- **Docker Desktop** 24.0+（Windows/macOS）或 **Docker Engine**（Linux）
- **Docker Compose** v2.0+（Docker Desktop 已内置）
- 浏览器（Chrome / Edge / Firefox）

### 验证 Docker 安装

```bash
docker --version
docker compose version
```

### Windows 用户：配置国内镜像加速（可选但推荐）

编辑 Docker Desktop → Settings → Docker Engine，添加国内镜像源：

```json
{
  "registry-mirrors": [
    "https://docker.1ms.run",
    "https://docker.m.daocloud.io"
  ]
}
```

点击 Apply & Restart 重启 Docker。

### Windows 用户：限制 WSL2 内存（推荐）

如果系统内存 ≤ 16GB，建议限制 WSL2 内存上限，避免 Docker 占用过多导致系统崩溃。

创建文件 `C:\Users\<你的用户名>\.wslconfig`：

```ini
[wsl2]
memory=4GB
processors=4
swap=2GB
```

然后在 PowerShell 中执行：

```powershell
wsl --shutdown
```

重启 Docker Desktop 生效。

---

## 2. 下载配置文件

### 方式一：克隆官方仓库（推荐）

```bash
git clone https://github.com/langgenius/dify.git
cd dify/docker
```

### 方式二：单独下载配置文件（轻量）

如果不需要完整源码，只下载 docker 目录的配置文件：

```bash
# 创建项目目录
mkdir dify-local && cd dify-local

# 下载 docker-compose.yaml
curl -L -o docker-compose.yaml https://raw.githubusercontent.com/langgenius/dify/main/docker/docker-compose.yaml

# 下载 .env.example
curl -L -o .env.example https://raw.githubusercontent.com/langgenius/dify/main/docker/.env.example
```

> 如果 GitHub 访问慢，可以使用镜像代理：
> `https://gh-proxy.com/https://raw.githubusercontent.com/...`

---

## 3. 修改配置

### 3.1 复制环境变量文件

```bash
cp .env.example .env
```

### 3.2 修改端口（重要）

Dify 默认使用 80 端口，Windows/macOS 上可能被占用。编辑 `.env`：

```bash
# 找到这一行，将 80 改为 8080（或其他未被占用的端口）
EXPOSE_NGINX_PORT=8080
```

同时修改 URL 配置（如果改了端口）：

```bash
CONSOLE_API_URL=http://localhost:8080
CONSOLE_WEB_URL=http://localhost:8080
SERVICE_API_URL=http://localhost:8080
APP_WEB_URL=http://localhost:8080
FILES_URL=http://localhost:8080
```

### 3.3 生成密钥（可选）

`.env` 中的 `SECRET_KEY` 建议替换为随机字符串：

```bash
# Linux/macOS
openssl rand -base64 42

# Windows PowerShell
[Convert]::ToBase64String((1..42 | ForEach-Object { Get-Random -Maximum 256 }))
```

---

## 4. 启动服务

### 4.1 一键启动

```bash
docker compose up -d
```

首次启动会拉取约 12 个镜像（约 3-5GB），根据网络速度需要 5-20 分钟。

### 4.2 查看启动状态

```bash
# 查看所有容器状态
docker compose ps

# 实时查看日志
docker compose logs -f

# 查看某个服务的日志
docker compose logs -f nginx
docker compose logs -f api
```

### 4.3 等待服务就绪

所有容器启动后，API 服务还需要 30-60 秒初始化。看到以下状态表示就绪：

```
NAME                            STATUS
dify-local-nginx-1             Up (healthy)
dify-local-api-1               Up (healthy)
dify-local-worker-1            Up (healthy)
dify-local-db_postgres-1       Up (healthy)
dify-local-redis-1             Up (healthy)
...
```

---

## 5. 验证部署

### 5.1 访问 Web 界面

浏览器打开：`http://localhost:8080`（如果你改了端口，用你的端口）

首次访问会跳转到管理员账号设置页面，设置邮箱和密码后登录。

### 5.2 验证 API

```bash
curl http://localhost:8080/health
```

返回 `{"status":"ok"}` 表示 API 正常。

---

## 6. 常见坑点与排错

### 坑 1：nginx / ssrf_proxy / sandbox 容器反复重启

**症状**：`docker compose ps` 显示这些容器不断 Restarting，日志报 `cp: -r not specified` 或 `not a directory`。

**原因**：docker-compose.yaml 中挂载了配置文件，但本地不存在这些文件。Docker Desktop Windows 会把不存在的挂载源自动创建为**目录**，导致容器内复制文件失败。

**解决**：
1. 停止服务：`docker compose down`
2. 检查 docker/nginx/、docker/ssrf_proxy/、docker/sandbox/ 目录下是否有配置文件
3. 如果缺失，从官方仓库下载对应文件：
   ```bash
   # nginx 配置
   curl -L -o nginx/nginx.conf.template https://raw.githubusercontent.com/langgenius/dify/main/docker/nginx/nginx.conf.template
   # ...（其他配置文件类似）
   ```
4. 确认文件是**文件**不是目录，重新启动：`docker compose up -d`

### 坑 2：ssrf_proxy 报 Syntax error: "(" unexpected

**症状**：ssrf_proxy 容器启动失败，日志报 `/docker-entrypoint.sh: 1: Syntax error: "(" unexpected`。

**原因**：下载的 `.sh` 文件是 Windows CRLF 换行符，Linux 容器的 sh 解释器无法识别。

**解决**：将 `.sh` 文件转换为 LF 换行符。

```bash
# Linux/macOS
sed -i 's/\r$//' ssrf_proxy/docker-entrypoint.sh

# Windows PowerShell（用 .NET 转换）
$content = [System.IO.File]::ReadAllText("ssrf_proxy\docker-entrypoint.sh")
$content = $content -replace "`r`n", "`n"
[System.IO.File]::WriteAllText("ssrf_proxy\docker-entrypoint.sh", $content)
```

### 坑 3：访问 localhost:8080 被拒绝（Connection refused）

**症状**：浏览器访问 `http://localhost:8080` 报 ERR_CONNECTION_REFUSED，但 nginx 容器显示 Up。

**原因**：nginx 容器内 `/etc/nginx/conf.d/` 目录为空，缺少 `default.conf.template`，entrypoint 脚本无法生成 server 配置。

**解决**：
1. 下载缺失的配置文件：
   ```bash
   curl -L -o nginx/conf.d/default.conf.template https://raw.githubusercontent.com/langgenius/dify/main/docker/nginx/conf.d/default.conf.template
   ```
2. 重建 nginx 容器：
   ```bash
   docker compose up -d --force-recreate nginx
   ```

### 坑 4：Docker Desktop 频繁崩溃

**症状**：Docker Desktop 突然退出，报错 `dockerDesktopLinuxEngine: The system cannot find the file specified`。

**原因**：系统内存不足。WSL2 占用大量内存且缓存不释放，叠加其他应用，导致系统杀死 Docker 进程。

**解决**：
1. 限制 WSL2 内存（见 [环境准备](#windows-用户限制-wsl2-内存推荐)）
2. 关闭不必要的应用（浏览器标签页、其他占用内存的程序）
3. 定期清理 Docker 无用镜像和容器：
   ```bash
   docker system prune -a
   ```
4. 如果仍频繁崩溃，考虑增加物理内存或迁移到云服务器部署

### 坑 5：模型调用报 404 / Connection error

**症状**：在 Dify 中配置模型后，测试连接报 `404` 或 `connection error`。

**原因**：
- API Base URL 格式错误（缺少 `/v1` 后缀，或多了斜杠）
- 使用了 Dify 的「Custom Endpoint」（它期望 Dify 特定接口，不是通用 OpenAI 兼容端点）
- Docker 容器内访问宿主机用了 `127.0.0.1`（应该用 `host.docker.internal`）

**解决**：
- 使用 DeepSeek 插件或 OpenAI-API-compatible 插件，不要用 Custom Endpoint
- API Base URL 格式：`https://api.deepseek.com/v1`（注意 `/v1` 后缀）
- 如果使用本地代理（如 CC-Switch），容器内地址用 `http://host.docker.internal:端口/v1`

### 坑 6：Web 应用分享链接 404

**症状**：发布应用后，访问 Web App URL 报 404 或 "App not found"。

**原因**：`.env` 中的 `APP_WEB_URL` 等 URL 配置与实际访问地址不一致（比如改了端口但没更新 URL 配置）。

**解决**：编辑 `.env`，将所有 URL 配置改为实际访问地址（含端口），然后重启：
```bash
docker compose up -d
```

---

## 7. 运维命令

### 常用命令

```bash
# 启动服务
docker compose up -d

# 停止服务
docker compose down

# 重启所有服务
docker compose restart

# 重启单个服务
docker compose restart api

# 查看服务状态
docker compose ps

# 查看日志
docker compose logs -f
docker compose logs -f api

# 进入容器
docker compose exec api bash

# 查看容器资源占用
docker stats
```

### 数据备份与恢复

```bash
# 备份数据库
docker compose exec db_postgres pg_dump -U postgres dify > backup.sql

# 恢复数据库
docker compose exec -T db_postgres psql -U postgres dify < backup.sql
```

### 升级 Dify

```bash
# 拉取最新镜像
docker compose pull

# 重启服务
docker compose up -d
```

---

## 参考链接

- Dify 官方文档：https://docs.dify.ai/
- Dify GitHub：https://github.com/langgenius/dify
- Dify 社区：https://discord.gg/FngNHpbcY7

---

*本指南基于 Dify 1.17.0 版本编写，其他版本可能略有差异。*
