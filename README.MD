# Go Harvest 使用文档

本文档基于当前 Go Harvest 代码中的功能整理，适合用于部署、初始化、日常使用和故障排查。

## 1. 项目简介

Go Harvest 是 Harvest 的 GoFrame 重写版本，面向 PT 站点管理、签到、数据抓取、种子搜索、下载器管理、辅种、通知和自动化任务。

当前主要功能：

- WebUI 页面和 REST API。
- SQLite / PostgreSQL 初始化。
- 用户登录、JWT 鉴权、Swagger UI。
- 我的站点管理、站点配置管理、站点数据抓取、签到、搜索。
- qBittorrent / Transmission 下载器管理和种子控制。
- 种子推送、批量推送、辅种、种子迁移。
- 计划任务、手动任务、任务结果记录。
- 通知推送和消息历史。
- TMDB / 豆瓣查询。
- 旧版数据导入、数据备份导入导出。
- 主程序、站点配置、WebUI 更新。
- 服务器状态、服务状态、实时日志。

## 2. 推荐部署方式

推荐使用 PostgreSQL 数据库部署。SQLite 适合试用、轻量单机或临时环境，长期使用建议选择 PostgreSQL。

下面的 Compose 内容直接复制到自己的 `docker-compose.yml` 使用即可。

镜像内包含：

- Go Harvest 主程序。
- Redis，可在未配置外置 Redis 时由容器内部启动。
- 内置站点配置和 WebUI 静态文件。

### 2.1 启动授权变量

程序启动时会校验远端授权，必须提供：

- `EMAIL` 或 `DJANGO_SUPERUSER_EMAIL`：授权邮箱。
- `TOKEN`：授权 token。

缺少这两个值时容器会启动失败，并在日志中提示 `请配置 DJANGO_SUPERUSER_EMAIL/EMAIL 与 TOKEN`。

### 2.2 PostgreSQL 推荐版，使用 db/.env

适合正式部署。Compose 从 `./db/.env` 读取授权和数据库配置，数据目录也统一放在 `./db`、`./postgres-data` 等路径下。

先创建 `db/.env`：

```env
EMAIL=you@example.com
TOKEN=replace-with-your-token

POSTGRES_HOST=go-harvest-postgres
POSTGRES_PORT=5432
POSTGRES_DB=goharvest
POSTGRES_USER=goharvest
POSTGRES_PASSWORD=change-me
```

Compose：

```yaml
services:
  go-harvest-postgres:
    image: postgres:17-alpine
    container_name: go-harvest-postgres
    restart: unless-stopped
    env_file:
      - path: ./db/.env
        required: true
    volumes:
      - ./postgres-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U \"$${POSTGRES_USER}\" -d \"$${POSTGRES_DB}\""]
      interval: 2s
      timeout: 3s
      retries: 30
      start_period: 2s

  harvest:
    image: newptools/go-harvest:latest
    container_name: go-harvest
    restart: unless-stopped
    depends_on:
      go-harvest-postgres:
        condition: service_healthy
    env_file:
      - path: ./db/.env
        required: true
    ports:
      - "5173:5173"
    volumes:
      - ./db:/app/db
      - ./icons:/icons
      - ./sites:/app/sites
      - ./downloads:/downloads
      - ./downloaders:/downloaders
    healthcheck:
      test: [ "CMD-SHELL", "curl -fsS http://127.0.0.1:5173/api.json >/dev/null" ]
      interval: 20s
      timeout: 5s
      retries: 10
      start_period: 30s
```

启动：

```bash
mkdir -p db sites icons downloads downloaders postgres-data
docker compose up -d
```

如果需要复杂配置，可继续在 `db/.env` 中追加这些可选变量：

```env
TZ=Asia/Shanghai
LOGGER_LEVEL=info
LOGGER_STDOUT=true

AUTO_UPDATE=false
AUTO_UPDATE_TAG=upgrade_all
GIT_PROXY=https://gh-proxy.org/

HARVEST_MEMORY_LIMIT=
HARVEST_GOGC=80
HARVEST_SITE_TASK_CONCURRENCY=5
HARVEST_DOWNLOADER_TASK_CONCURRENCY=2
HARVEST_DOWNLOADER_STATUS_CONCURRENCY=3
HARVEST_WS_STATUS_CONCURRENCY=3
HARVEST_PUSH_DOWNLOAD_CONCURRENCY=2
HARVEST_DB_LOG_RETENTION_DAYS=15

# 留空时使用容器内置 Redis；外置 Redis 示例：redis://192.168.1.10:6379/15
CACHE_REDIS_CONNECTION=
REDIS_SERVER_PORT=6379
REDIS_MAXMEMORY=128mb
REDIS_MAXMEMORY_POLICY=allkeys-lru
```

### 2.3 PostgreSQL 简单版，不使用 .env

适合不想维护 `.env` 文件的部署。该版本只写必要变量：启动授权、PostgreSQL 初始化参数、Go Harvest 连接 PostgreSQL 所需参数。

```yaml
services:
  go-harvest-postgres:
    image: postgres:17-alpine
    container_name: go-harvest-postgres
    restart: unless-stopped
    environment:
      POSTGRES_DB: "goharvest"
      POSTGRES_USER: "goharvest"
      POSTGRES_PASSWORD: "change-me"
    volumes:
      - ./postgres-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U \"$${POSTGRES_USER}\" -d \"$${POSTGRES_DB}\""]
      interval: 2s
      timeout: 3s
      retries: 30
      start_period: 2s

  harvest:
    image: newptools/go-harvest:latest
    container_name: go-harvest
    restart: unless-stopped
    depends_on:
      go-harvest-postgres:
        condition: service_healthy
    environment:
      EMAIL: "you@example.com"
      TOKEN: "replace-with-your-token"
      POSTGRES_HOST: "go-harvest-postgres"
      POSTGRES_PORT: "5432"
      POSTGRES_DB: "goharvest"
      POSTGRES_USER: "goharvest"
      POSTGRES_PASSWORD: "change-me"
    ports:
      - "5173:5173"
    volumes:
      - ./db:/app/db
      - ./sites:/app/sites
      - ./icons:/icons
      - ./downloads:/downloads
      - ./downloaders:/downloaders
    healthcheck:
      test: [ "CMD-SHELL", "curl -fsS http://127.0.0.1:5173/api.json >/dev/null" ]
      interval: 20s
      timeout: 5s
      retries: 10
      start_period: 30s
```

启动：

```bash
mkdir -p db sites icons downloads downloaders postgres-data
docker compose up -d
```

### 2.4 PostgreSQL 初始化

首次启动后访问：

```text
http://127.0.0.1:5173/setup
```

初始化页面选择 PostgreSQL。使用上面的 Compose 时，连接信息通常为：

```text
Host: go-harvest-postgres
Port: 5432
Database: goharvest
User: goharvest
Password: db/.env 或 Compose environment 中的 POSTGRES_PASSWORD；已配置时可留空提交
```

数据库地址、端口、库名、用户会优先从环境变量填充；如果设置了 `POSTGRES_PASSWORD`，密码也会在后端自动读取，但不会回显到初始化页面。

### 2.5 SQLite 可选版，不使用 .env

SQLite 只适合轻量或临时场景。SQLite 版本不需要任何 PostgreSQL 相关环境变量。

```yaml
services:
  harvest:
    image: newptools/go-harvest:latest
    container_name: go-harvest
    restart: unless-stopped
    environment:
      EMAIL: "you@example.com"
      TOKEN: "replace-with-your-token"
    ports:
      - "5173:5173"
    volumes:
      - ./db:/app/db
      - ./sites:/app/sites
      - ./icons:/icons
      - ./downloads:/downloads
      - ./downloaders:/downloaders
    healthcheck:
      test: [ "CMD-SHELL", "curl -fsS http://127.0.0.1:5173/api.json >/dev/null" ]
      interval: 20s
      timeout: 5s
      retries: 10
      start_period: 30s
```

### 2.6 SQLite 可选版，使用 db/.env

`db/.env` 只需要授权信息，不要写数据库相关环境变量：

```env
EMAIL=you@example.com
TOKEN=replace-with-your-token
```

Compose：

```yaml
services:
  harvest:
    image: newptools/go-harvest:latest
    container_name: go-harvest
    restart: unless-stopped
    env_file:
      - path: ./db/.env
        required: true
    ports:
      - "5173:5173"
    volumes:
      - ./db:/app/db
      - ./sites:/app/sites
      - ./icons:/icons
      - ./downloads:/downloads
      - ./downloaders:/downloaders
    healthcheck:
      test: [ "CMD-SHELL", "curl -fsS http://127.0.0.1:5173/api.json >/dev/null" ]
      interval: 20s
      timeout: 5s
      retries: 10
      start_period: 30s
```

SQLite 初始化时数据库文件固定为：

```text
/app/db/data.sqlite3
```

### 2.7 访问地址和目录

默认访问地址：

- WebUI: `http://127.0.0.1:5173`
- 初始化页面: `http://127.0.0.1:5173/setup`
- Swagger UI: `http://127.0.0.1:5173/swagger`
- OpenAPI JSON: `http://127.0.0.1:5173/api.json`

默认挂载目录：

- `./db:/app/db`，配置、数据库、日志、媒体文件，`db/.env` 也放在这里。
- `./sites:/app/sites`，用户自定义站点配置。
- `./downloads:/downloads`，下载目录。
- `./downloaders:/downloaders`，下载器相关目录。
- `./postgres-data:/var/lib/postgresql/data`，PostgreSQL 数据目录。
- `./icons:/icons`, 自定义站点图标目录。

### 2.8 常用环境变量

必需变量：

```env
EMAIL=you@example.com
TOKEN=replace-with-your-token
```

PostgreSQL 必需变量：

```env
POSTGRES_HOST=go-harvest-postgres
POSTGRES_PORT=5432
POSTGRES_DB=goharvest
POSTGRES_USER=goharvest
POSTGRES_PASSWORD=change-me
```

可选变量：

```env
TZ=Asia/Shanghai
LOGGER_LEVEL=info
LOGGER_STDOUT=true
AUTO_UPDATE=false
AUTO_UPDATE_TAG=upgrade_all
GIT_PROXY=https://gh-proxy.org/

HARVEST_MEMORY_LIMIT=
HARVEST_GOGC=80
HARVEST_SITE_TASK_CONCURRENCY=5
HARVEST_DOWNLOADER_TASK_CONCURRENCY=2
HARVEST_DOWNLOADER_STATUS_CONCURRENCY=3
HARVEST_WS_STATUS_CONCURRENCY=3
HARVEST_PUSH_DOWNLOAD_CONCURRENCY=2
HARVEST_DB_LOG_RETENTION_DAYS=15
HARVEST_CLIENT_MAX_BODY_SIZE=2214592512

CACHE_REDIS_CONNECTION=
REDIS_SERVER_PORT=6379
REDIS_MAXMEMORY=128mb
REDIS_MAXMEMORY_POLICY=allkeys-lru
```

说明：

- `EMAIL` 和 `TOKEN` 是启动授权必需变量。
- `POSTGRES_HOST`、`POSTGRES_PORT`、`POSTGRES_DB`、`POSTGRES_USER`、`POSTGRES_PASSWORD` 只在 PostgreSQL 部署时需要。
- `AUTO_UPDATE` 控制容器启动时是否自动执行更新，`AUTO_UPDATE_TAG` 控制更新范围。
- `GIT_PROXY` 用于 GitHub 资源更新。
- `HARVEST_MEMORY_LIMIT` 可限制 Go 运行时内存目标；留空表示不额外设置。
- `HARVEST_GOGC` 控制 Go GC 触发比例，默认 `80`。
- `HARVEST_SITE_TASK_CONCURRENCY` 控制站点任务并发。
- `HARVEST_DOWNLOADER_TASK_CONCURRENCY` 控制下载器任务并发。
- `HARVEST_DOWNLOADER_STATUS_CONCURRENCY` 控制下载器状态刷新并发。
- `HARVEST_WS_STATUS_CONCURRENCY` 控制 WebSocket 状态推送并发。
- `HARVEST_PUSH_DOWNLOAD_CONCURRENCY` 控制推送种子时的下载并发。
- `HARVEST_DB_LOG_RETENTION_DAYS` 控制数据库日志保留天数。
- `HARVEST_CLIENT_MAX_BODY_SIZE` 控制上传请求体大小上限，单位字节；默认约 2GB，用于支持旧版 SQLite 数据库上传。
- `CACHE_REDIS_CONNECTION` 存在时使用外置 Redis，例如 `redis://192.168.1.10:6379/15`。
- `CACHE_REDIS_CONNECTION` 留空时，容器内部 Redis 会使用 `REDIS_SERVER_PORT`。
- `REDIS_MAXMEMORY` 和 `REDIS_MAXMEMORY_POLICY` 仅用于内置 Redis。

## 3. 初始化系统

首次启动后访问：

```text
http://127.0.0.1:5173/setup
```

初始化需要设置：

- 数据库类型：`sqlite` 或 `pgsql`。
- 管理员用户名、密码、邮箱【非必需】。
- JWT Secret：可留空，系统会自动生成。

SQLite 初始化注意事项：

- 数据库文件名固定为 `data.sqlite3`。
- SQLite 时不需要填写数据库 host、port、user、pass。
- 时区固定为 `Asia/Shanghai`。

初始化完成后，系统会：

- 创建数据库表。
- 创建管理员用户。
- 写入 `/app/db/config.yaml`。
- 初始化缓存。
- 启动计划任务调度器。

## 4. 登录和鉴权

### 4.1 WebUI 登录

初始化完成后访问：

```text
http://127.0.0.1:5173
```

使用初始化时创建的管理员账号登录。

### 4.2 API 登录

登录接口：

```http
POST /api/token/pair
Content-Type: application/json

{
  "username": "admin",
  "password": "password"
}
```

后续请求使用：

```http
Authorization: Bearer <access_token>
```

Token 相关接口：

- `POST /api/token/pair` 登录。
- `POST /api/token/refresh` 刷新 token。
- `POST /api/token/verify` 验证 token。

### 4.3 Swagger UI

访问：

```text
http://127.0.0.1:5173/swagger
```

在 Swagger UI 右上角 `Authorize` 中填入 access token 即可调试需要鉴权的接口。

## 5. 站点管理

### 5.1 站点配置

站点配置来源：

- 镜像内置 `/app/internal`。
- 用户挂载目录 `/app/sites`。
- 运行时可通过更新任务拉取最新配置。

常用接口：

- `GET /api/mysite/website` 获取站点配置列表。
- `GET /api/mysite/website/names` 获取站点名称列表。
- `GET /api/mysite/website/available` 获取可添加站点。
- `GET /api/mysite/website/{site}` 获取单个站点配置。

自定义站点配置建议放在宿主机 `./sites`，容器内对应 `/app/sites`。

### 5.2 我的站点

常用接口：

- `GET /api/mysite/mysite` 我的站点列表。
- `GET /api/mysite/mysite/{id}` 我的站点详情。
- `POST /api/mysite/mysite` 新增站点。
- `PUT /api/mysite/mysite/{id}` 更新站点。
- `DELETE /api/mysite/mysite/{id}` 删除站点。

我的站点常用字段：

- `site`：站点配置名。
- `nickname`：显示名称。
- `mirror`：站点访问地址。
- `cookie`：站点 Cookie。
- `user_agent`：浏览器 UA。
- `user_id`：站点用户 ID，部分站点抓取个人信息需要。
- `passkey` / `authkey`：下载链接或特殊接口需要。
- `proxy`：站点代理，格式如 `http://127.0.0.1:7890`。
- `available`：是否可用。
- `sign_in`：是否参与自动签到。
- `get_info`：是否参与自动抓取数据。

列表接口返回会做轻量化：

- `latest_state` / `status` 只保留最近两天站点数据。
- `latest_sign_info` / `sign_info` 只保留当天签到数据。
- 详情接口可查看完整记录。

### 5.3 浏览器插件接口

常用接口：

- `GET /api/auth/monkey/get_site/{host}` 根据域名匹配站点配置。
- `POST /api/auth/monkey/save_site` 保存浏览器插件采集到的站点 Cookie。
- `POST /api/auth/monkey/iyuu` IYUU 辅种站点匹配。

安全 token 使用时需要 `Monkey.` 前缀。系统接口会在旧数据导入场景中自动补前缀，页面中不需要手工填写 `Monkey.`。

## 6. 签到、数据抓取和搜索

### 6.1 单站操作

- `GET /api/mysite/info/{mysite_id}` 抓取单个站点数据。
- `GET /api/mysite/sign/{mysite_id}` 单站签到。
- `GET /api/mysite/search/{my_site_id}?key=keyword` 单站搜索。

单站抓取和单站签到会快速返回接收结果，具体执行在后台完成。单站抓取和签到本身不发送完成通知，但抓取到站点新消息时仍会按通知配置发送消息提醒。

### 6.2 批量操作

- `GET /api/mysite/info` 抓取所有启用站点数据。
- `GET /api/mysite/sign` 所有启用站点签到。

批量任务会通过任务系统执行，并按通知配置发送汇总。

### 6.3 聚合搜索

HTTP 聚合搜索：

```http
POST /api/mysite/search/http
Content-Type: application/json

{
  "key": "keyword",
  "max_count": 100,
  "sites": [1, 2, 3]
}
```

`sites` 不传时搜索所有可用站点。

WebUI 通常使用 WebSocket / SSE 搜索能力，适合实时展示各站点搜索结果。

## 7. 下载器管理

支持：

- qBittorrent。
- Transmission。

下载器配置接口：

- `GET /api/option/downloaders` 下载器列表。
- `GET /api/option/downloaders/{id}` 下载器详情。
- `POST /api/option/downloaders` 新增下载器。
- `PUT /api/option/downloaders/{id}` 更新下载器。
- `DELETE /api/option/downloaders/{id}` 删除下载器。

下载器常用字段：

- `name`：名称。
- `category`：类型，`Qb` / `Tr`，也兼容 `qbittorrent`、`qb`、`transmission`、`tr`。
- `host`：地址，例如 `http://192.168.1.10:8080`。
- `username` / `password`：登录信息。
- `sort_id`：排序。

### 7.1 下载器状态和控制

常用接口：

- `GET /api/option/downloaders/status` 全部下载器状态。
- `GET /api/option/downloaders/test/{downloader_id}` 测试连接。
- `GET /api/option/downloaders/main/{downloader_id}` 下载器全量数据。
- `GET /api/option/downloaders/speed/{downloader_id}` 下载器速度。
- `GET /api/option/downloaders/torrents/{downloader_id}` 获取种子列表。
- `POST /api/option/downloaders/control/{downloader_id}` 控制种子。

控制动作兼容 `action` 和 `command` 字段。

### 7.2 分类、标签和 Tracker

常用接口：

- `GET /api/option/downloaders/category/{downloader_id}` 分类列表。
- `POST /api/option/downloaders/category/{downloader_id}` 创建分类。
- `PUT /api/option/downloaders/category/{downloader_id}` 修改分类。
- `DELETE /api/option/downloaders/category/{downloader_id}` 删除分类。
- `POST /api/option/downloaders/category/set/{downloader_id}` 设置种子分类。
- `DELETE /api/option/downloaders/category/set/{downloader_id}` 移除种子分类。
- `GET /api/option/downloaders/tags/{downloader_id}` 标签列表。
- `POST /api/option/downloaders/tags/{downloader_id}` 创建标签。
- `DELETE /api/option/downloaders/tags/{downloader_id}` 删除标签。
- `POST /api/option/downloaders/tags/set/{downloader_id}` 添加种子标签。
- `DELETE /api/option/downloaders/tags/set/{downloader_id}` 移除种子标签。
- `GET /api/option/downloaders/trackers/{downloader_id}` 获取 Tracker。
- `PUT /api/option/downloaders/trackers/replace/{downloader_id}` 替换 Tracker。

### 7.3 WebSocket 状态

- `GET /api/ws/downloader?downloader_id=1` 下载器主数据。
- `GET /api/ws/downloader/speed?downloader_id=1` 下载器速度。

## 8. 种子推送和辅种

### 8.1 推送种子到下载器

接口：

```http
POST /api/option/push_torrent/{downloader_id}
```

请求体：

```json
{
  "urls": ["https://example.com/file.torrent"],
  "ids": ["8888"],
  "category": "Movie",
  "save_path": "/downloads/Movie",
  "paused": false,
  "skip_hash_checking": false
}
```

说明：

- `urls` 保证为 URL 列表或空列表。
- `ids` 支持字符串或数字。
- URL 已包含 passkey/sign 等 query，或无 query 且以 `.torrent` 结尾时，会直接下载种子。
- 否则会根据站点下载链接生成函数生成下载链接。

### 8.2 批量推送浏览器插件种子

接口：

```http
POST /api/option/push_monkey/{downloader_id}/{my_site_id}
```

批量推送会异步执行，接口快速返回任务接收结果。任务完成后发送通知。

推送通知格式示例：

```text
种子ID【8887】：推送成功
种子ID【8888】：下载失败：原因
种子ID【8889】：推送失败：原因
```

并发控制：

- 种子文件下载并发由 `HARVEST_PUSH_DOWNLOAD_CONCURRENCY` 控制。
- 默认每个下载结束后会有间隔，避免对站点请求过猛。

### 8.3 辅种

常用接口：

- `GET /api/option/repeat/{downloader_id}` 启动单个下载器辅种。
- `GET /api/option/repeat/err/{downloader_id}` 查看辅种错误记录。

计划任务中也包含：

- `下载器辅种任务`
- `celery.单个下载器辅种`
- `celery.单站辅种任务`

## 9. 计划任务

常用接口：

- `GET /api/option/tasks` 可执行任务列表。
- `GET /api/option/tasks?include_internal=true` 包含内部兼容任务。
- `GET /api/option/schedule` 计划任务列表。
- `POST /api/option/schedule` 创建计划任务。
- `PUT /api/option/schedule/{id}` 更新计划任务。
- `DELETE /api/option/schedule/{id}` 删除计划任务。
- `GET /api/option/exec` 手动执行任务。
- `POST /api/option/exec` 手动执行任务并传参。
- `GET /api/option/task-results` 任务结果列表。
- `GET /api/option/task-results/{task_id}` 任务结果详情。
- `DELETE /api/option/task-results` 清理任务结果。

当前任务列表：

- `自动清理内存`
- `批量抓取站点信息`
- `自动签到任务`
- `CloudFlare测速`
- `种子迁移任务`
- `下载器辅种任务`

内部兼容任务：

- `celery.抓取单个站点信息`
- `celery.签到单个站点`
- `celery.缓存站点数据`
- `celery.单个下载器辅种`
- `celery.单站辅种任务`
- `celery.CookieCloud`
- `celery.PTPP`
- `celery.PTD`
- `celery.自动更新任务`

创建计划任务示例：

```json
{
  "name": "每天自动签到",
  "task": "自动签到任务",
  "crontab": "0 8 * * *",
  "enabled": true,
  "args": [],
  "kwargs": {}
}
```

时区固定为：

```text
Asia/Shanghai
```

## 10. 通知和消息历史

支持的通知渠道：

- 企业微信：`wechat_work_push`
- WxPusher：`wxpusher_push`
- PushDeer：`pushdeer_push`
- Server 酱：`server_chan_push`
- Bark：`bark_push`
- 爱语飞飞：`iyuu_push`
- Telegram：`telegram_push`
- PushPlus：`pushplus_push`
- 喵呜通知：`meow_push`

通知配置存储在 `option_option` 表中，配置名通常为对应渠道名，配置值为 JSON。

Telegram 示例：

```json
{
  "telegram_token": "bot_token",
  "telegram_chat_id": "chat_id",
  "parse_mode": "Markdown",
  "proxy": "http://127.0.0.1:7890"
}
```

通知测试：

```http
GET /api/option/test?title=测试&content=这是一条测试通知
```

消息历史：

- `GET /api/option/notice` 消息历史。
- `PUT /api/option/notice/{id}/read` 标记消息已读。
- `POST /api/option/notice/read` 全部消息已读。
- `DELETE /api/option/notice/{id}` 删除消息。
- `DELETE /api/option/notice` 清空消息。

## 11. 系统配置

系统配置接口：

- `GET /api/option/options` 配置列表。
- `GET /api/option/options/{id}` 配置详情。
- `POST /api/option/options` 创建配置。
- `PUT /api/option/options/{id}` 更新配置。
- `DELETE /api/option/options/{id}` 删除配置。

常见配置项：

- `tmdb_api_auth`：TMDB API 配置。
- `notice_category_enable`：通知分类开关。
- `notice_content_item`：通知内容字段开关。
- 各通知渠道配置，如 `telegram_push`。
- Cookie 导入配置，如 CookieCloud、PTPP、PTD 相关配置。

## 12. TMDB 和豆瓣

### 12.1 TMDB

TMDB 接口需要配置 `tmdb_api_auth`。

常用接口：

- `GET /api/tmdb/search/{query}` 综合搜索。
- `GET /api/tmdb/match/movie?query=xxx` 匹配电影。
- `GET /api/tmdb/match/tv?query=xxx` 匹配剧集。
- `GET /api/tmdb/movie/{id}` 电影详情。
- `GET /api/tmdb/tv/{id}` 剧集详情。
- `GET /api/tmdb/person/{id}` 人物详情。
- `GET /api/tmdb/season/{tv_id}/{season_id}` 季详情。
- `GET /api/tmdb/episode/{tv_id}/{season_id}/{episode_id}` 集详情。
- `GET /api/tmdb/popular/movies` 热门电影。
- `GET /api/tmdb/popular/tvs` 热门剧集。
- `GET /api/tmdb/top_rated/movies` 高分电影。
- `GET /api/tmdb/top_rated/tvs` 高分剧集。
- `GET /api/tmdb/upcoming/movies` 即将上映。
- `GET /api/tmdb/playing/movies` 正在上映。
- `GET /api/tmdb/airing_today/tvs` 今日播出。
- `GET /api/tmdb/on_the_air/tvs` 正在热播。

### 12.2 豆瓣

常用接口：

- `GET /api/option/douban/top250`
- `GET /api/option/douban/top250/{start}`
- `GET /api/option/douban/celebrity/{celebrity_id}`
- `GET /api/option/douban/subject/{subject_id}`
- `GET /api/option/douban/search?q=keyword`
- `GET /api/option/douban/tags?category=movie`
- `GET /api/option/douban/hot`
- `GET /api/option/douban/rank`

## 13. 数据导入和备份

### 13.1 旧版本数据导入

接口：

```http
POST /api/setup/import
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "base_url": "http://旧版本地址",
  "legacy_token": "旧版本安全token"
}
```

说明：

- 该接口走统一 Bearer 鉴权。
- `legacy_token` 使用时会自动补 `Monkey.` 前缀。
- 导入站点时会按 `site` 合并：
    - 本地已存在时合并缺失字段。
    - 本地不存在时创建。
- 系统配置项本地已有时不覆盖，只补充缺失配置。
- 导入完成后会清理并重建相关缓存。

直接读取旧版本 SQLite 数据库：

```http
POST /api/setup/sqlite
Authorization: Bearer <access_token>
Content-Type: multipart/form-data

file=<old data.sqlite3>
```

该接口只导入站点、下载器、系统配置和计划任务，跳过旧库中的用户和通知历史。

### 13.2 备份导出和导入

导出：

```http
GET /api/setup/backup
Authorization: Bearer <access_token>
```

导入：

```http
POST /api/setup/backup
Authorization: Bearer <access_token>
Content-Type: multipart/form-data

file=<backup file>
```

备份导入会保留当前系统用户和通知历史，不会恢复备份中的 `auth_user` 与 `option_noticehistory`。

## 14. 更新

常用更新接口：

- `GET /api/auth/update/log` 获取主程序更新日志。
- `GET /api/auth/update/sites` 获取站点配置更新信息。
- `GET /api/option/update?upgrade_tag=upgrade_all` 启动更新任务。

更新范围由 `upgrade_tag` 控制，通常包括：

- 主程序。
- 站点配置。
- WebUI。

主程序更新逻辑：

- 查询 GitHub Release。
- 下载当前架构对应压缩包。
- 解压并替换本地 `/app/main`。
- 设置执行权限。
- 在 Docker 部署中可通过重建或重启容器生效。

资源更新会使用 `GIT_PROXY` 访问 GitHub。

## 15. 状态、日志和排障

### 15.1 状态接口

- `GET /api/auth/server/status` 服务器状态。
- `GET /api/auth/services/status` 服务状态。
- `GET /api/auth/logs/stream` 实时日志 SSE。
- `GET /api/option/logging` 最新日志。

Docker 中日志文件默认位于：

```text
/app/db/logs
```

常见日志：

- `harvest.log`

### 15.2 查看容器日志

```bash
docker logs -f go-harvest
```

### 15.3 常见问题

#### 无法访问 WebUI

检查：

- 容器是否运行：`docker ps`
- 端口是否映射：默认宿主机 `5173`
- Go 服务是否启动：容器内 `127.0.0.1:5173`

#### 系统提示未初始化

访问：

```text
http://127.0.0.1:5173/setup
```

检查 `/app/db/config.yaml` 是否存在，以及数据库是否可访问。

#### SQLite 布尔值显示为 0/1

当前响应层会对常用布尔字段做转换。如果仍看到异常字段，需要补充字段名到响应归一化逻辑。

#### Telegram 通知超时

在 `telegram_push` 配置中设置代理：

```json
{
  "proxy": "http://127.0.0.1:7890"
}
```

#### GitHub 资源更新失败

设置或更换：

```env
GIT_PROXY=https://gh-proxy.org/
```

#### 下载器推送种子失败

检查：

- 下载器地址、账号、密码。
- 站点 Cookie 是否有效。
- 下载链接是否包含 passkey/sign。
- 下载器保存路径是否存在。
- `HARVEST_PUSH_DOWNLOAD_CONCURRENCY` 是否过高。

## 16. 本地开发

本地运行：

```bash
go run main.go
```

默认后端端口：

```text
8000
```

本地测试：

```bash
GOCACHE=/private/tmp/go-build-cache go test ./...
```

本地 Swagger：

```text
http://127.0.0.1:8000/swagger
```

本地 WebUI：

- 如果 `templates/index.html` 存在，Go 服务会直接挂载 WebUI。
- 否则只提供 API 和 `/setup` 初始化页面。

## 17. 目录说明

常用目录：

```text
/app/db                  配置、数据库、日志、媒体文件
/app/db/config.yaml      运行配置
/app/db/data.sqlite3     SQLite 数据库
/app/db/logs             日志目录
/app/sites               用户站点配置
/app/internal            内置站点配置
/app/templates           WebUI 静态文件
/downloads               下载目录
/downloaders             下载器相关目录
/icons                   图标目录
```

源码目录：

```text
api/                     GoFrame API 定义
internal/controller      控制器
internal/service         业务服务
internal/service/ptspider PT 站点抓取、签到、搜索
internal/service/downloader 下载器集成
internal/service/scheduler 任务系统
manifest/docker          Docker 镜像和容器启动配置
manifest/sql             PostgreSQL 初始化 SQL
manifest/sql/sqlite      SQLite 初始化 SQL
```
