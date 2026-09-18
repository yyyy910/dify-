
# Dify 本地私有化部署 · 学习全记录

> 从零基础到能独立运维 Dify
> 时间：2026年9月11日 — 至今

---

## 目录

1. 项目简介
2. 第 1 周：部署 + 网络排错
3. 第 2 周：Linux + Docker 排错
4. 第 3 周：Dify 深度运维 + Nginx
5. 附录：故障排查手册

---

## 一、项目简介

**目标**：在 Windows 本机部署 Dify，接入 DeepSeek API，实现可对话 AI 机器人。

**环境**：
- Windows 11
- Docker Desktop（WSL2 后端）
- 手机热点（部分时段用校园网）

**最终成果**：
- 本地运行的 Dify（10 个容器）
- 能通过浏览器访问、上传文档、对话
- GitHub 仓库（持续更新学习记录）

---

## 二、第 1 周：部署 + 网络排错（9月11-17日）

### 2.1 部署流程

**启用 WSL2**：

```powershell
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
重启后：

powershell
wsl --install
wsl --set-default-version 2
安装 Ubuntu：命令行连 GitHub 超时 → 改用 Microsoft Store 安装。

安装 Docker Desktop：勾选 "Use WSL 2 instead of Hyper-V"。

配置镜像源：先后试了 ustc、163、dockerproxy、腾讯云、DaoCloud，最终改为在 yml 里直接给镜像加前缀。

2.2 遇到的 11 个报错
序号	报错	原因	解决
1	insecure-registries must be array	JSON 格式错误	改为 "insecure-registries": []
2	dial tcp registry-1.docker.io:443	加速器失效触发回退	直接在 compose 文件加镜像前缀
3	unknown directive "server"	记事本 BOM 头	用 PowerShell 无 BOM 写入
4	502 Bad Gateway	Nginx 缓存了旧容器 IP	docker compose restart nginx
5	401 Unauthorized	Cookie 冲突	清浏览器 Cookie，用 127.0.0.1
6	Can't locate revision	数据库版本迁移冲突	清空 volumes/db/data 重建
7	Vector store type is not configured	缺少向量数据库	增加 weaviate 服务
8	Failed to request plugin daemon	缺 PLUGIN_DAEMON_URL	补充环境变量
9	Base model deepseek-flash not found	插件模型名不匹配	改用 DeepSeek 原生插件
10	amqp://guest:**@127.0.0.1:5672 Connection refused	Worker 默认连 RabbitMQ	加 CELERY_BROKER_URL
11	Authentication Fails, your api key is invalid	API Key 失效	DeepSeek 官网重新生成
2.3 网络分层排错法
7 步排查顺序：

步骤	命令	验证层
1	ping 127.0.0.1	本机协议栈
2	ipconfig	网络身份
3	ping 本机IP	网卡
4	ping 网关	内网
5	Test-NetConnection 114.114.114.114 -Port 53	公网
6	ping api.deepseek.com	DNS
7	Test-NetConnection 127.0.0.1 -Port 8080	端口
关键心得：

ping 不通 ≠ 网络不通（ICMP 可能被屏蔽）

优先用 Test-NetConnection（走 TCP）

从底层往上查，能精准定位问题层

我的基准数据：

本机 IP：10.168.4.119

网关：10.168.4.5

ping 网关延迟：2-4ms

ping DeepSeek 延迟：74ms

三、第 2 周：Linux + Docker 排错（9月18-24日）
3.1 Linux 核心命令
10 个必会命令：

命令	作用	我的实操
pwd	显示当前目录	在容器里执行 → /app/api
ls	列出文件	看到 Dify 代码目录
cd	切换目录	cd /app 成功
cat	查看文件内容	看了 .env.example
find	找文件	找到配置文件
grep	搜关键词	搜到 DB 相关配置
tail	看文件末尾	（容器里无日志文件，改用 docker logs）
head	看文件开头	看了配置前 10 行
ps	看进程	容器里没有，用 docker top 代替
netstat/ss	看端口	容器里没有，用 docker compose ps 代替
3.2 容器里的坑
现象：在容器里执行 ps aux，报 command not found。

原因：Dify 镜像为了安全，故意删掉了 ps、netstat 等工具。

替代方案：

从宿主机看进程：docker top <容器名>

从宿主机看端口：docker compose ps

内核提供的 /proc 目录永远存在：ls /proc 能看到所有 PID

3.3 Docker 排错三件套
powershell
# 1. 看状态
docker compose ps

# 2. 看日志
docker compose logs <服务名> --tail 30

# 3. 进容器
docker exec -it <容器名> bash
3.4 模拟故障练习
故障 1：改坏 DB_PASSWORD

现象：api 一直 Restarting，日志报 password authentication failed

原因：密码不一致

修复：改回正确密码 + --force-recreate

故障 2：占用 8080 端口

现象：nginx 启动失败，日志报 address already in use

原因：另一个容器占了 8080

修复：docker stop <占用容器> + 重建 nginx

3.5 关键教训
restart vs up -d 的区别：

操作	读新配置吗？	什么时候用
restart	❌	只重启进程
up -d	✅（配置变了就重建）	改了配置
up -d --force-recreate	✅（无条件重建）	强制生效
四、第 3 周：Dify 深度运维 + Nginx（9月25日-至今）
4.1 Dify 10 个服务
服务	职责
nginx	反向代理，对外入口
web	前端界面
api	后端主逻辑
worker	异步任务处理
plugin_daemon	插件管理
sandbox	代码执行沙箱
db	数据库（PostgreSQL）
redis	消息队列
minio	对象存储
weaviate	向量数据库
4.2 启动顺序
text
第一层（无依赖）：db、redis、minio、weaviate
独立：sandbox
第二层（依赖第一层）：api、worker、plugin_daemon
第三层（依赖 api）：web
第四层（依赖 api 和 web）：nginx
4.3 Nginx 反向代理
配置示例：

nginx
location /console/api { proxy_pass http://api:5001; }
location /api { proxy_pass http://api:5001; }
location /files { proxy_pass http://api:5001; }
location / { proxy_pass http://web:3000; }
匹配规则：

/api/xxx → api

/console/api/xxx → api

其他所有 → web

4.4 反向代理 vs NAT vs 负载均衡
类型	定义	例子
反向代理	服务端部署，隐藏后端	Dify 的 nginx
NAT	私有 IP 转公网 IP	手机热点、家用路由器
负载均衡	请求分发到多台服务器	大厂多机架构
判断口诀：

客户端搭的 → 正向

服务端搭的 → 反向

4.5 日志级别
INFO（默认）：只打印关键事件。

DEBUG：打印每一步细节（HTTP 请求、SQL、签名）。

切换方法：在 api.environment 加 LOG_LEVEL: DEBUG，重建生效。

过滤命令：

powershell
docker compose logs api | findstr /i "error"
docker compose logs api | findstr /i "minio"
4.6 自定义域名的尝试
做法：改 C:\Windows\System32\drivers\etc\hosts，加一行 127.0.0.1 mydify.local。

结果：ping mydify.local 能通，但浏览器访问一直转圈。

原因：Dify 环境变量写死了 localhost:8080，浏览器地址 mydify.local 和 API 地址 localhost 不一致 → 跨域。

结论：本地用 localhost 就够了，自定义域名的价值在云服务器部署时体现。

4.7 HTTPS 决策
结论：本地不配置 HTTPS。

理由：

本地价值不大

Windows 配置复杂

云服务器上用 Let's Encrypt 更简单

五、附录：故障排查手册
5.1 排错四步法
text
第一步：看状态
  docker compose ps

第二步：看日志
  docker compose logs <服务名> --tail 30

第三步：抓关键词
  在日志里找 ERROR / FATAL / refused / timeout

第四步：修复
  根据关键词对症下药
5.2 常见报错对应表
报错关键词	原因	解决
password authentication failed	密码错	检查 api 和 db 的密码
Connection refused	依赖服务没启动	先启动依赖
address already in use	端口冲突	找到占用者并停止
is restarting	容器崩溃循环	看日志找原因
unhealthy	健康检查失败	检查 healthcheck 命令
timed out	网络问题	换源或检查 DNS
5.3 三个救命命令
powershell
docker compose ps
docker compose logs <服务名> --tail 30
docker compose config -q
5.4 数据备份
powershell
cd C:\Users\LENOVO\dify-local
docker compose down
Copy-Item -Recurse -Force .\volumes "D:\dify-backup\volumes-日期"
docker compose up -d