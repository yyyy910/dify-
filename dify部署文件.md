# Dify 本地私有化部署全流程文档（终极版）

> **环境**：Windows 11 + Docker Desktop（WSL2 后端）+ 手机热点网络  
> **目标**：本地部署 Dify，接入 DeepSeek API，实现可对话 AI 机器人  
> **时间**：2026年9月11日 — 9月12日（历时约 14 小时）  
> **作者**：我自己

---

## 目录

1. 环境准备
2. 镜像源配置（多次失败与调整）
3. 部署 Dify
4. 排错全记录
5. 部署成功后的必做事项
6. 故障排查方法论
7. 完整部署时间线
8. 部署成功后的健康检查清单
9. 常用命令汇总
10. 隐藏知识点
11. 技术栈全景图
12. 最终架构图
13. 经验总结
14. 下一步计划
15. 附录：Docker 核心概念速查

---

## 一、环境准备

### 1.1 网络环境说明

- **原始网络**：校园网（存在 DNS 缓存污染、出口带宽限制）
- **备选网络**：手机 5G 热点（绕过校园网限制，但存在流量和稳定性问题）
- **最终使用**：手机热点

### 1.2 启用 WSL2

以**管理员身份**打开 PowerShell，执行：

```powershell
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
```

重启电脑后继续：

```powershell
wsl --install
wsl --set-default-version 2
```

> **说明**：WSL2 是 Windows 内置的 Linux 子系统，Docker 依赖它运行。

### 1.3 安装 Ubuntu 发行版

```powershell
wsl --install -d Ubuntu
```

**遇到的问题**：命令行连接 GitHub 超时，无法下载 Ubuntu 镜像。  
**解决方法**：改用 **Microsoft Store（微软商城）** 搜索 "Ubuntu" 安装。

**遇到的问题**：Ubuntu 分发版注册状态损坏。  
**解决方法**：

```powershell
# 1. 关闭所有 WSL 进程
wsl --shutdown

# 2. 注销损坏的 Ubuntu 分发
wsl --unregister Ubuntu

# 3. 强制默认版本为 WSL2
wsl --set-default-version 2
```

然后重新从 Microsoft Store 安装 Ubuntu。

### 1.4 安装 Docker Desktop

- 从官网下载 Docker Desktop for Windows
- 安装时勾选 **Use WSL 2 instead of Hyper-V**
- 安装完成后启动 Docker Desktop，等待右下角鲸鱼图标变成**绿色稳定状态**

**额外踩坑**：发现网络缓存还是校园网，执行清 DNS 缓存：

```powershell
ipconfig /flushdns
ipconfig /all
```

发现看到的是 **WSL 虚拟网卡**，不是手机热点的无线网卡，所以看不到真实 DNS：

```powershell
ipconfig /all | findstr "WLAN"
```

**结论**：WSL 内部存在旧的 DNS 缓存，需要绕开。后来在 Docker 配置里**写死 `dns` 字段**，强制 WSL 内部使用公共 DNS，绕开这个 bug。

---

## 二、镜像源配置（多次失败与调整）

### 2.1 第一次配置（部分失效）

```json
{
  "builder": {
    "gc": {
      "defaultKeepStorage": "20GB",
      "enabled": true
    }
  },
  "registry-mirrors": [
    "https://docker.mirrors.ustc.edu.cn",
    "https://hub-mirror.c.163.com"
  ],
  "experimental": false
}
```

**问题**：执行 `docker pull hello-world` 失败，说明部分镜像源失效。

### 2.2 第二次配置（改为可用源）

```json
{
  "builder": {
    "gc": {
      "defaultKeepStorage": "20GB",
      "enabled": true
    }
  },
  "registry-mirrors": [
    "https://hub-mirror.c.163.com"
  ],
  "experimental": false
}
```

**问题**：大体积镜像（postgres、minio、dify-api）拉取仍然超时，报错：

```
dial tcp registry-1.docker.io:443: connect: connection timed out
```

**原因分析**：

> Docker 的 `registry-mirrors` 机制是"**优先尝试**"，而非"**强制代理**"。当加速器返回错误（哪怕只是临时波动），Docker 会**自动回退**到官方仓库 `registry-1.docker.io`。而官方源在国内网络环境下必然超时。此外，Docker Desktop 自身可能有一个**内部透明代理** (`http.docker.internal:3128`)，会绕过你配置的加速器。

### 2.3 加入 DNS 强制配置

```json
{
  "builder": {
    "gc": {
      "defaultKeepStorage": "20GB",
      "enabled": true
    }
  },
  "registry-mirrors": [
    "https://dockerproxy.com"
  ],
  "dns": ["223.5.5.5", "114.114.114.114"],
  "experimental": false
}
```

**问题**：`dockerproxy.com` 这个镜像源连不上，**连接超时**，不是 DNS 解析失败，是这个公共镜像代理当前不可用。

### 2.4 更换腾讯云源

```json
{
  "builder": {
    "gc": {
      "defaultKeepStorage": "20GB",
      "enabled": true
    }
  },
  "registry-mirrors": [
    "https://ccr.ccs.tencentyun.com"
  ],
  "dns": ["223.5.5.5", "114.114.114.114"],
  "experimental": false
}
```

**结果**：小镜像能拉，大镜像仍然不稳。

### 2.5 第五次配置（最终方案）

```json
{
  "registry-mirrors": [
    "https://docker.xuanyuan.me",
    "https://docker.1ms.run",
    "https://docker.m.daocloud.io"
  ],
  "insecure-registries": [],
  "debug": false,
  "experimental": false
}
```

**遇到的问题**：配置中的 `insecure-registries` 报 `must be array`，因为 JSON 语法错误。**改为 `"insecure-registries": []` 即可。**

### 2.6 终极思路转变

尝试了多轮公共镜像源后，发现都有各自的限流或不可用问题（`docker.xuanyuan.me` 提示"免费节点当前繁忙"，`docker.1panel.live` 对部分镜像返回 403）。

**最终方案**：**放弃依赖 `daemon.json` 的镜像源，改为直接在 `docker-compose.yml` 里给每个镜像加前缀**，彻底绕过 Docker 的回退逻辑。

---

## 三、部署 Dify

### 3.1 下载 dify-local 源码

从 Dify 官方仓库或社区下载 dify-local 完整源码文件夹，放在：

```
C:\Users\LENOVO\dify-local\
```

### 3.2 修改 docker-compose.yml（最终版）

```yaml
services:
  # ==========================================
  # 基础数据服务 (Database & Cache & Storage)
  # ==========================================
  db:
    image: docker.m.daocloud.io/postgres:14
    restart: always
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: 你的数据库密码
      POSTGRES_DB: dify
    volumes:
      - ./volumes/db/data:/var/lib/postgresql/data
    networks:
      - default
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: docker.m.daocloud.io/redis:6-alpine
    restart: always
    volumes:
      - ./volumes/redis/data:/data
    command: redis-server --requirepass 你的数据库密码
    networks:
      - default
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  minio:
    image: docker.m.daocloud.io/minio/minio:latest
    restart: always
    ports:
      - "9000:9000"
      - "9001:9001"
    environment:
      MINIO_ROOT_USER: 你的MinIO用户名
      MINIO_ROOT_PASSWORD: 你的minio密码
    volumes:
      - ./volumes/minio/data:/data
    command: server /data --console-address ":9001"
    networks:
      - default
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 10s
      retries: 5

  # ==========================================
  # 向量数据库 (Vector Store)
  # ==========================================
  weaviate:
    image: docker.m.daocloud.io/semitechnologies/weaviate:1.19.0
    restart: always
    volumes:
      - ./volumes/weaviate:/var/lib/weaviate
    environment:
      QUERY_DEFAULTS_LIMIT: 25
      AUTHENTICATION_ANONYMOUS_ACCESS_ENABLED: 'true'
      PERSISTENCE_DATA_PATH: '/var/lib/weaviate'
      DEFAULT_VECTORIZER_MODULE: 'none'
      CLUSTER_HOSTNAME: 'node1'
    networks:
      - default

  # ==========================================
  # 核心应用服务 (Core Dify Services)
  # ==========================================
  api:
    image: docker.m.daocloud.io/langgenius/dify-api:1.14.2
    restart: always
    environment:
      MODE: api
      MIGRATION_ENABLED: "true"
      SECRET_KEY: 你的密钥
      DB_HOST: db
      DB_PORT: 5432
      DB_USERNAME: postgres
      DB_PASSWORD: 你的数据库密码
      DB_DATABASE: dify
      REDIS_HOST: redis
      REDIS_PORT: 6379
      REDIS_PASSWORD: 你的数据库密码
      CELERY_BROKER_URL: redis://:你的数据库密码@redis:6379/1
      BROKER_URL: redis://:你的数据库密码@redis:6379/1
      STORAGE_TYPE: s3
      S3_ENDPOINT: http://minio:9000
      S3_BUCKET_NAME: dify
      S3_ACCESS_KEY: 你的MinIO用户名
      S3_SECRET_KEY: 你的minio密码
      VECTOR_STORE: weaviate
      WEAVIATE_ENDPOINT: http://weaviate:8080
      CONSOLE_API_URL: http://localhost:8080
      CONSOLE_WEB_URL: http://localhost:8080
      APP_API_URL: http://localhost:8080
      FILES_URL: http://localhost:8080
      PLUGIN_DAEMON_URL: http://plugin_daemon:5002
      PLUGIN_DAEMON_KEY: 你的密钥
      CONSOLE_CORS_ALLOW_ORIGINS: 'http://localhost:8080,http://127.0.0.1:8080'
      WEB_API_CORS_ALLOW_ORIGINS: 'http://localhost:8080,http://127.0.0.1:8080'
      COOKIE_SECURE: "false"
      SESSION_COOKIE_SECURE: "false"
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
      minio:
        condition: service_healthy
      weaviate:
        condition: service_started
    volumes:
      - ./volumes/app/storage:/app/api/storage
    networks:
      - default
      - ssrf_proxy_network

  worker:
    image: docker.m.daocloud.io/langgenius/dify-api:1.14.2
    restart: always
    environment:
      MODE: worker
      SECRET_KEY: 你的密钥
      DB_HOST: db
      DB_PORT: 5432
      DB_USERNAME: postgres
      DB_PASSWORD: 你的数据库密码
      DB_DATABASE: dify
      REDIS_HOST: redis
      REDIS_PORT: 6379
      REDIS_PASSWORD: 你的数据库密码
      CELERY_BROKER_URL: redis://:你的数据库密码@redis:6379/1
      BROKER_URL: redis://:你的数据库密码@redis:6379/1
      STORAGE_TYPE: s3
      S3_ENDPOINT: http://minio:9000
      S3_BUCKET_NAME: dify
      S3_ACCESS_KEY: 你的MinIO用户名
      S3_SECRET_KEY: 你的minio密码
      CONSOLE_API_URL: http://localhost:8080
      APP_API_URL: http://localhost:8080
      PLUGIN_DAEMON_URL: http://plugin_daemon:5002
      PLUGIN_DAEMON_KEY: 你的密钥
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
      minio:
        condition: service_healthy
    volumes:
      - ./volumes/app/storage:/app/api/storage
    networks:
      - default
      - ssrf_proxy_network

  web:
    image: docker.m.daocloud.io/langgenius/dify-web:1.14.2
    restart: always
    environment:
      CONSOLE_API_URL: http://localhost:8080
      APP_API_URL: http://localhost:8080
    depends_on:
      - api
    networks:
      - default

  plugin_daemon:
    image: docker.m.daocloud.io/langgenius/dify-plugin-daemon:0.6.0-local
    restart: always
    environment:
      SECRET_KEY: 你的密钥
      SERVER_KEY: 你的密钥
      DIFY_INNER_API_URL: http://api:5001
      DIFY_INNER_API_KEY: 你的密钥
      PLUGIN_REMOTE_INSTALLING_HOST: 0.0.0.0
      PLUGIN_REMOTE_INSTALLING_PORT: 5003
      PLUGIN_WORKING_PATH: /app/storage
      DB_HOST: db
      DB_PORT: 5432
      DB_USERNAME: postgres
      DB_PASSWORD: 你的数据库密码
      DB_DATABASE: dify
      REDIS_HOST: redis
      REDIS_PORT: 6379
      REDIS_PASSWORD: 你的数据库密码
      S3_ENDPOINT: http://minio:9000
      S3_BUCKET_NAME: dify
      S3_ACCESS_KEY: 你的MinIO用户名
      S3_SECRET_KEY: 你的minio密码
    depends_on:
      - db
      - redis
      - minio
    volumes:
      - ./volumes/plugin_daemon:/app/storage
    networks:
      - default

  sandbox:
    image: docker.m.daocloud.io/langgenius/dify-sandbox:latest
    restart: always
    environment:
      API_KEY: dify-sandbox
      SANDBOX_PORT: 8194
    networks:
      - ssrf_proxy_network

  nginx:
    image: docker.m.daocloud.io/nginx:latest
    restart: always
    ports:
      - "8080:80"
      - "8443:443"
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d
      - ./volumes/certbot/conf:/etc/letsencrypt
    depends_on:
      - api
      - web
    networks:
      - default

networks:
  default:
    driver: bridge
  ssrf_proxy_network:
    driver: bridge
    internal: true
```

### 3.3 创建必要目录与 Nginx 配置

```powershell
mkdir nginx\conf.d, volumes\app\storage, volumes\db\data, volumes\redis\data, volumes\minio\data, volumes\plugin_daemon, volumes\sandbox\conf, volumes\certbot\conf, volumes\weaviate
```

**`nginx/conf.d/default.conf` 内容**：

```nginx
server {
    listen 80;
    server_name _;
    client_max_body_size 100M;

    location /console/api {
        proxy_pass http://api:5001;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
    location /api {
        proxy_pass http://api:5001;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
    location /files {
        proxy_pass http://api:5001;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
    location / {
        proxy_pass http://web:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### 3.4 启动前检查

**检查 ①**：确认 `default.conf` 存在

```powershell
Test-Path nginx\conf.d\default.conf
```

**检查 ②**：确认 80 端口是否被占用

```powershell
netstat -ano | findstr :80
```

- 没有输出 → 80 端口空闲
- 有输出 → 被占用，需把 nginx 端口改为 `"8080:80"` 和 `"8443:443"`

### 3.5 拉取与启动

```powershell
docker compose config -q   # 验证 YAML 语法
docker compose pull        # 拉取所有镜像
docker compose up -d       # 后台启动
docker compose ps          # 查看状态
```

访问地址：`http://localhost:8080/install`

---

## 四、排错全记录

| 序号 | 报错信息 | 原因 | 解决方法 |
|---|---|---|---|
| 1 | `insecure-registries must be array` | JSON 格式错误 | 改为 `"insecure-registries": []` |
| 2 | `dial tcp registry-1.docker.io:443` | 加速器失效触发回退 | 直接在 compose 文件加镜像前缀 |
| 3 | `unknown directive "server"` | PowerShell 保存时带 BOM 头 | 用无 BOM 方式写入文件 |
| 4 | `502 Bad Gateway` | Nginx 缓存了旧容器 IP | `docker compose restart nginx` |
| 5 | `401 Unauthorized` | Cookie 冲突或 SECRET_KEY 不一致 | 清浏览器 Cookie，用 127.0.0.1 访问 |
| 6 | `Can't locate revision identified by 'c3f1a9b2e6d4'` | 数据库版本迁移冲突 | 清空 `volumes/db/data` 重建 |
| 7 | `Vector store type is not configured` | 缺少向量数据库 | 增加 `weaviate` 服务并配置 |
| 8 | `Failed to request plugin daemon` | 缺 `PLUGIN_DAEMON_URL` | 补充环境变量 |
| 9 | `Base model deepseek-flash not found` | 插件模型名不匹配 | 改用 DeepSeek 原生插件 |
| 10 | `amqp://guest:**@127.0.0.1:5672 Connection refused` | Worker 默认连 RabbitMQ | 加 `CELERY_BROKER_URL: redis://...` |
| 11 | `Authentication Fails, Your api key: ****5d45 is invalid` | API Key 失效 | 去 DeepSeek 官网重新生成 |

### 关键修复过程说明

**关于 BOM 头问题**：  
Windows PowerShell 默认保存文件会加 `\ufeff`（BOM 头），Nginx 把开头的 `server` 识别成乱码，报 `unknown directive`。解决方式：

```powershell
[System.IO.File]::WriteAllText("$PWD\nginx\conf.d\default.conf", $conf, (New-Object System.Text.UTF8Encoding $false))
```

**关于 amqp:// 问题**：  
Dify 1.14.2 镜像**默认**会用 RabbitMQ（`amqp://guest:guest@127.0.0.1:5672`）。当配置里没有 `CELERY_BROKER_URL` 时，Worker 就会去连不存在的 RabbitMQ。解决方法是在 `api` 和 `worker` 里都加上：

```yaml
CELERY_BROKER_URL: redis://:你的数据库密码@redis:6379/1
BROKER_URL: redis://:你的数据库密码@redis:6379/1
```

**关于 API Key 问题**：  
Dify 保存 API Key 时**不做实时验证**，只显示绿点。真正调用时才报 401。解决方式是去 DeepSeek 官网**删除旧的、生成新的**，然后更新到 Dify 的模型供应商配置里。

---

## 五、部署成功后的必做事项

### 5.1 第一次访问与管理员账号初始化

1. 浏览器访问 `http://localhost:8080/install`
2. 设置管理员账号（邮箱 + 用户名 + 密码 ≥8位含字母数字）
3. 提交后自动跳转登录页
4. 用刚设置的账号登录

> ⚠️ **提醒**：本地部署没有配置邮件服务器，**找回密码功能不可用**。密码一定要记牢或存进密码管理器。

### 5.2 安装 DeepSeek 插件

因为 Dify 1.14.2 默认不带 DeepSeek，需要手动装：

**方式一（在线安装）**：
1. 左下角 **设置 → 模型供应商**
2. 找到 **DeepSeek**，点击 **安装**
3. 安装完成后点进去，**只填 API Key**（不用填地址、不用选协议）

**方式二（离线安装）**：
```powershell
Invoke-WebRequest -Uri "https://marketplace.dify.ai/api/v1/plugins/langgenius/deepseek/0.0.5/download" -OutFile ".\volumes\plugin_daemon\deepseek.difypkg"
docker compose restart plugin_daemon api
```
然后在 **设置 → 模型供应商 → 安装 → 从本地安装** 选择该文件。

### 5.3 配置 DeepSeek API Key

1. 去 `https://platform.deepseek.com` 注册
2. 左侧 **API Keys** → 创建 API Key
3. **立刻复制**（弹窗关闭后不可再查看）
4. 充值（**没有余额 API 会返回 401**）
5. 在 Dify 的 DeepSeek 配置里填入 Key

### 5.4 创建第一个聊天应用

1. 左侧 **工作室 → 创建空白应用**
2. 选 **Chatflow**（多轮对话）或 **聊天助手**（单轮）
3. 起个名字 → 创建
4. 点开 **LLM 积木** → 选择 `deepseek-chat` 或 `deepseek-v4-flash`
5. 点右上角 **发布**
6. 点 **预览** → 输入"你好"测试

### 5.5 验证成功的判断标准

- 后端：`docker compose ps` 所有容器是 `Up` 或 `healthy`
- 前端：`http://localhost:8080` 能打开并登录
- AI：应用里发消息能在 **3-5 秒内** 收到回复

---

## 六、故障排查方法论

### 6.1 排查五步法

```
1. 复现问题    → 确认操作步骤，能稳定复现
2. 定位服务    → docker compose ps 找出哪个容器异常
3. 查看日志    → docker compose logs <服务名> --tail 50
4. 抓关键词    → Error / Exception / refused / timeout / 401 / 502
5. 对症下药    → 网络问题换源 / 配置问题改 YAML / 数据问题清卷
```

### 6.2 常见报错分类与对应方向

| 报错关键词 | 大概率原因 | 排查方向 |
|---|---|---|
| `Connection refused` | 目标服务没起来 / 端口错 | `docker compose ps` 看状态 |
| `timeout` | 网络不通 / 被墙 | 换镜像源 / 检查 DNS |
| `401 Unauthorized` | API Key 无效 / Cookie 冲突 | 重新生成 Key / 换无痕窗口 |
| `403 Forbidden` | 加速器拒绝 / 权限不够 | 换源 / 检查权限 |
| `404 Not Found` | URL 写错 / 资源不存在 | 检查路径 / 版本号 |
| `502 Bad Gateway` | Nginx 找不到后端 | 重启 nginx |
| `500 Internal Server Error` | 后端代码崩了 | 看 API 日志最后几行 |
| `unknown directive` | 配置文件格式错 | 检查 BOM 头 / 缩进 |
| `Cannot connect to amqp://` | 消息队列配置错 | 加 `CELERY_BROKER_URL` |
| `Vector store type is not configured` | 缺向量数据库 | 加 weaviate 服务 |

### 6.3 三个救命命令

```powershell
# 1. 看所有容器状态（哪个挂了）
docker compose ps

# 2. 看某个服务的最近日志（报错在哪）
docker compose logs <服务名> --tail 50

# 3. 验证 YAML 语法（配置对不对）
docker compose config -q
```

---

## 七、完整部署时间线

```
【第一天 下午-晚上】
1. 启用 WSL2 → 重启 → 安装 Ubuntu
2. 安装 Docker Desktop
3. 配置镜像源（第一次）
4. docker pull hello-world 失败
5. 换镜像源（第二次、第三次…）
6. 大镜像仍然超时
7. 开始修改 docker-compose.yml，加镜像前缀
8. 不断更换前缀源

【第一天 深夜 - 第二天 凌晨】
9. 补齐目录和 nginx/default.conf
10. docker compose up -d
11. 遇到 BOM 头问题（Nginx 报 unknown directive）
12. 无 BOM 重写 default.conf
13. 遇到 502（Nginx 缓存旧 IP）→ 重启 nginx
14. 遇到数据库迁移冲突 → 清 volumes/db/data
15. 遇到缺少向量数据库 → 加 weaviate
16. 遇到 plugin_daemon 报错 → 补环境变量
17. 遇到 amqp:// 问题 → 加 CELERY_BROKER_URL
18. 遇到 API Key 401 → 重新生成 Key

【第二天 凌晨4点】
19. 部署成功，能对话 ✅
```

---

## 八、部署成功后的健康检查清单

每次重启电脑后，**部署前**先做这几项检查，能省 80% 的排错时间。

```powershell
# 1. Docker Desktop 是否启动
docker version

# 2. WSL2 是否正常运行
wsl --list --verbose

# 3. 网络是否畅通（手机热点是否连上）
ping api.deepseek.com

# 4. 端口是否被占用
netstat -ano | findstr :8080

# 5. 数据卷是否还在
dir C:\Users\LENOVO\dify-local\volumes

# 6. 启动
cd C:\Users\LENOVO\dify-local
docker compose up -d

# 7. 等 30 秒后检查状态
docker compose ps
```

---

## 九、常用命令汇总

### 9.1 WSL 相关

```powershell
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
wsl --install
wsl --set-default-version 2
wsl --install -d Ubuntu
wsl --shutdown
wsl --unregister Ubuntu
wsl --list --verbose
```

### 9.2 Docker 相关

```powershell
docker pull <镜像名>          # 拉取镜像
docker run -d -p 8080:80 nginx  # 后台运行容器并映射端口
docker ps                     # 查看运行中的容器
docker ps -a                  # 查看全部容器
docker stop <容器ID>          # 停止容器
docker rm <容器ID>            # 删除已停止的容器
docker rmi <镜像名>           # 删除镜像
docker logs <容器名>          # 查看日志
docker info | findstr "Registry Mirrors"  # 验证镜像源
```

### 9.3 Docker Compose 相关

```powershell
docker compose config -q              # 验证 YAML 语法
docker compose pull                   # 拉取所有镜像
docker compose up -d                  # 后台启动所有服务
docker compose ps                     # 查看服务状态
docker compose logs <服务名> --tail 30 # 查看某个服务日志
docker compose restart <服务名>        # 重启某个服务
docker compose down                   # 停止并删除所有容器（数据保留）
docker compose exec <服务名> <命令>    # 在容器内执行命令
```

### 9.4 Windows 网络相关

```powershell
ipconfig /flushdns               # 清空 DNS 缓存
ipconfig /all                    # 查看所有网络信息
ipconfig /all | findstr "WLAN"   # 只看无线网卡
netstat -ano | findstr :80       # 查看 80 端口占用
```

### 9.5 一键替换镜像前缀（PowerShell）

```powershell
(Get-Content docker-compose.yml -Encoding UTF8) -replace 'docker\.(1panel\.live|xuanyuan\.me|m\.daocloud\.io|1ms\.run|rat\.dev)/', '' | Set-Content docker-compose.yml -Encoding UTF8
```

---

## 十、隐藏知识点

### 10.1 为什么用 `docker compose` 而不是 `docker run`

- `docker run` 每次只能启动一个容器
- `docker compose` 用一个 YAML 文件统一定义多个容器 + 网络 + 卷 + 依赖关系
- 生产环境 99% 用 Compose 或 K8s，不用手敲 `docker run`

### 10.2 为什么容器之间可以用"名字"互相访问

Compose 会自动创建一个**桥接网络**，所有容器加入后可以用**服务名**作为主机名互相访问。

例如 `api` 容器里访问数据库，可以直接写 `DB_HOST: db`，不用写 IP。这就是 Docker 的**服务发现**机制。

### 10.3 为什么用 `depends_on` 和 `healthcheck`

- `depends_on` 保证启动顺序（先 db，后 api）
- `healthcheck` 保证"服务真的能用了"才让依赖它的服务启动
- 这次部署里，因为没配好 healthcheck，导致 api 在 db 还没就绪时启动，报数据库连接错误

### 10.4 数据卷 vs 绑定挂载

| 类型 | 写法 | 特点 |
|---|---|---|
| **绑定挂载 (bind mount)** | `./volumes/db/data:/var/lib/postgresql/data` | 数据存在你指定的本机路径下，方便备份 |
| **命名卷 (named volume)** | `db_data:/var/lib/postgresql/data` | Docker 管理的卷，路径在 WSL 内部 |

你用的是**绑定挂载**，所以数据都在 `C:\Users\LENOVO\dify-local\volumes`，可以直接 copy 走。

---

## 十一、技术栈全景图

```
┌───────────────────────────────────────────────────┐
│                   操作系统层                       │
│  Windows 11  ←→  WSL2 (Linux 内核)  ←→  Ubuntu    │
└───────────────────────────────────────────────────┘
                        ↓
┌───────────────────────────────────────────────────┐
│                   容器平台层                       │
│         Docker Desktop  +  Docker Compose          │
└───────────────────────────────────────────────────┘
                        ↓
┌───────────────────────────────────────────────────┐
│                   Dify 应用层                      │
│  ┌─────────┬─────────┬─────────┬──────────────┐   │
│  │ Nginx   │ Web     │ API     │ Worker       │   │
│  │(代理)   │(前端)   │(后端)   │(异步任务)    │   │
│  └─────────┴─────────┴─────────┴──────────────┘   │
│  ┌─────────┬─────────┬─────────┬──────────────┐   │
│  │Postgres │ Redis   │ MinIO   │ Weaviate     │   │
│  │(数据库) │(队列)   │(存储)   │(向量库)      │   │
│  └─────────┴─────────┴─────────┴──────────────┘   │
│  ┌──────────────────┐  ┌────────────────────┐    │
│  │ Plugin Daemon    │  │ Sandbox            │    │
│  │ (插件：DeepSeek) │  │ (代码执行沙箱)     │    │
│  └──────────────────┘  └────────────────────┘    │
└───────────────────────────────────────────────────┘
                        ↓
┌───────────────────────────────────────────────────┐
│                   外部服务层                       │
│  DeepSeek API（云端大模型，通过 HTTPS 调用）        │
└───────────────────────────────────────────────────┘
```

**每一层对应要学的东西**：
- **操作系统层**：Linux 基础命令、WSL2
- **容器平台层**：Docker、Compose、YAML
- **应用层**：Python（后端）、FastAPI、Vue（前端）、数据库、Redis、消息队列
- **外部服务层**：HTTP API 调用、鉴权

---

## 十二、最终架构图

```
用户浏览器 (localhost:8080)
    ↓
┌─────────────────────────────┐
│  Nginx (端口映射 8080 → 80) │  ← 反向代理
└────┬────────────────────────┘
     │
     ├──────────────┬──────────────┐
     ▼              ▼              ▼
┌─────────┐   ┌─────────┐   ┌─────────┐
│   Web   │   │   API   │   │ Worker  │
│ (前端)  │   │ (后端)  │   │(任务队列)│
└─────────┘   └────┬────┘   └────┬────┘
                   │             │
     ┌─────────────┼─────────────┼─────────────┐
     ▼             ▼             ▼             ▼
┌─────────┐  ┌─────────┐  ┌─────────┐  ┌──────────┐
│Postgres │  │  Redis  │  │  MinIO  │  │ Weaviate │
│(数据库) │  │(消息队列)│  │(文件存储)│  │(向量存储)│
└─────────┘  └─────────┘  └─────────┘  └──────────┘
                              │
                              ▼
                    ┌──────────────────┐
                    │ Plugin Daemon    │
                    │ (插件：DeepSeek) │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │ DeepSeek API     │
                    │ (云端大模型)     │
                    └──────────────────┘
```

---

## 十三、经验总结

1. **国内镜像源不稳定**：公共加速器随时可能失效，最稳妥的方式是**在 compose 文件里直接加前缀**。
2. **YAML 格式极其严格**：缩进必须用空格，不能用 Tab，冒号后必须有空格。
3. **日志是第一排查工具**：`docker compose logs` 是定位问题的入口。
4. **Windows 记事本有坑**：保存配置文件容易带 BOM 头，建议用 VS Code 编辑。
5. **数据卷是命根子**：所有数据都在 `volumes` 目录，备份它就等于备份全部。
6. **Docker 的回退机制会掩盖真相**：`registry-mirrors` 只是"优先尝试"，失败会自动回退，所以单纯依赖它不可靠。
7. **API Key 的验证是"事后"的**：Dify 保存时不验证，调用时才报错，所以看到绿点不代表能用。

---

## 十四、下一步计划

- [ ] 系统学习 Python 基础语法
- [ ] 学习 FastAPI 写后端接口
- [ ] 学习 LangChain 和 RAG
- [ ] 做第二个项目：基于代码的知识库问答机器人
- [ ] 把项目部署到云服务器
- [ ] 把本文档提交到 GitHub

---

## 十五、附录：Docker 核心概念速查

| 概念 | 一句话解释 | 你这次部署里的例子 |
|---|---|---|
| **镜像 (Image)** | 软件的"安装包"，只读 | `postgres:14`、`dify-api:1.14.2` |
| **容器 (Container)** | 镜像运行起来的实例 | `dify-local-api-1`、`dify-local-db-1` |
| **数据卷 (Volume)** | 把容器内数据挂载到本机硬盘 | `./volumes/db/data` |
| **网络 (Network)** | 容器之间的通信通道 | `default`、`ssrf_proxy_network` |
| **端口映射 (Ports)** | 本机端口 → 容器端口 | `8080:80`、`9000:9000` |
| **环境变量 (Environment)** | 传给容器的配置参数 | `DB_PASSWORD`、`SECRET_KEY` |
| **Dockerfile** | 定义如何构建镜像的脚本 | 本次未使用（用的现成镜像） |
| **docker-compose.yml** | 定义一组容器如何协作的配置文件 | 你的主配置文件 |
| **Registry** | 镜像仓库（Docker Hub、DaoCloud 等） | `docker.m.daocloud.io` |

---

*文档创建于 2026-09-14*