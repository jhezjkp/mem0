# Mem0 Server — Dokploy 部署指南

## 概述

本指南介绍如何在 [Dokploy](https://dokploy.com) 上通过 **连接 GitHub 仓库、本地构建镜像** 的方式部署 Mem0 Server。

Mem0 Server 由三个服务组成：

| 服务 | 说明 |
|------|------|
| **mem0** | FastAPI 应用，提供记忆管理 REST API |
| **postgres** | pgvector 扩展的 PostgreSQL，用于向量存储 |
| **neo4j** | 图数据库，用于知识图谱存储 |

---

## 前提条件

- 已安装并运行 Dokploy（自托管版本）
- 服务器已安装 Docker
- 拥有硅基流动（或其他 OpenAI 兼容接口）的 API Key
- GitHub 仓库已包含本项目代码（当前分支：`dokploy`）

---

## 步骤一：在 Dokploy 中创建项目并连接 GitHub

1. 登录 Dokploy 控制台，点击 **Create Project**，填写项目名称（如 `mem0`）
2. 在项目内点击 **Create Service** → 选择 **Docker Compose**
3. 在 **Source** 面板中选择 **GitHub**：
   - 点击 **Connect GitHub**，完成 OAuth 授权
   - 选择仓库（如 `jhezjkp/mem0`）
   - **Branch** 填写 `dokploy`
   - **Compose Path** 填写 `server/docker-compose.prod.yml`

> Dokploy 会将仓库克隆到服务器本地，然后以仓库根目录为工作目录执行 docker-compose。

---

## 步骤二：准备生产版 Docker Compose 文件

在仓库中创建 `server/docker-compose.prod.yml`，内容如下（使用 `build:` 从源码本地构建镜像，而非拉取远程镜像）：

```yaml
name: mem0

services:
  mem0:
    build:
      context: .               # Dokploy 以 compose 文件所在目录(server/)为工作目录，context 用 . 即可
      dockerfile: Dockerfile   # 对应 server/Dockerfile
    ports:
      - "8000:8000"
    networks:
      - mem0_network
    volumes:
      - history_data:/app/history
    depends_on:
      postgres:
        condition: service_healthy
      neo4j:
        condition: service_healthy
    environment:
      - PYTHONUNBUFFERED=1
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - OPENAI_BASE_URL=${OPENAI_BASE_URL:-https://api.siliconflow.cn/v1}
      - LLM_MODEL=${LLM_MODEL:-Qwen/Qwen2.5-72B-Instruct}
      - LLM_TEMPERATURE=${LLM_TEMPERATURE:-0.2}
      - LLM_MAX_TOKENS=${LLM_MAX_TOKENS:-2000}
      - EMBEDDING_MODEL=${EMBEDDING_MODEL:-BAAI/bge-m3}
      - EMBEDDING_DIMS=${EMBEDDING_DIMS:-1024}
      - POSTGRES_HOST=postgres
      - POSTGRES_PORT=5432
      - POSTGRES_DB=${POSTGRES_DB:-postgres}
      - POSTGRES_USER=${POSTGRES_USER:-postgres}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD:-postgres}
      - POSTGRES_COLLECTION_NAME=${POSTGRES_COLLECTION_NAME:-memories}
      - NEO4J_URI=bolt://neo4j:7687
      - NEO4J_USERNAME=${NEO4J_USERNAME:-neo4j}
      - NEO4J_PASSWORD=${NEO4J_PASSWORD:-mem0graph}
      - HISTORY_DB_PATH=/app/history/history.db

  postgres:
    image: ankane/pgvector:v0.5.1
    restart: on-failure
    shm_size: "128mb"
    networks:
      - mem0_network
    environment:
      - POSTGRES_USER=${POSTGRES_USER:-postgres}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD:-postgres}
    healthcheck:
      test: ["CMD", "pg_isready", "-q", "-d", "postgres", "-U", "postgres"]
      interval: 5s
      timeout: 5s
      retries: 5
    volumes:
      - postgres_db:/var/lib/postgresql/data

  neo4j:
    image: neo4j:5.26.4
    networks:
      - mem0_network
    healthcheck:
      test: wget http://localhost:7687 || exit 1
      interval: 5s
      timeout: 10s
      retries: 20
      start_period: 90s
    volumes:
      - neo4j_data:/data
    environment:
      - NEO4J_AUTH=${NEO4J_USERNAME:-neo4j}/${NEO4J_PASSWORD:-mem0graph}
      - NEO4J_PLUGINS=["apoc"]
      - NEO4J_apoc_export_file_enabled=true
      - NEO4J_apoc_import_file_enabled=true
      - NEO4J_apoc_import_file_use__neo4j__config=true

volumes:
  postgres_db:
  neo4j_data:
  history_data:

networks:
  mem0_network:
    driver: bridge
```

将此文件提交并推送到远程仓库后，Dokploy 才能读取到它。

---

## 步骤三：配置环境变量

在 Dokploy 服务的 **Environment** 面板中，添加以下变量：

### 必填

| 变量 | 说明 |
|------|------|
| `API_KEY` | 接口认证密钥，所有请求必须携带，建议使用随机长字符串 |
| `OPENAI_API_KEY` | 硅基流动或其他服务的 API Key |

### 可选（有默认值）

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `OPENAI_BASE_URL` | `https://api.siliconflow.cn/v1` | LLM/Embedding API 地址 |
| `LLM_MODEL` | `Qwen/Qwen2.5-72B-Instruct` | 使用的 LLM 模型 |
| `LLM_TEMPERATURE` | `0.2` | 生成温度 |
| `LLM_MAX_TOKENS` | `4096` | 最大输出 Token 数（图谱实体提取需要输出较长 JSON，建议不低于 4096，否则截断会导致 JSON 解析错误） |
| `EMBEDDING_MODEL` | `BAAI/bge-m3` | Embedding 模型 |
| `EMBEDDING_DIMS` | `1024` | 向量维度 |
| `POSTGRES_PASSWORD` | `postgres` | PostgreSQL 密码（建议修改） |
| `NEO4J_PASSWORD` | `mem0graph` | Neo4j 密码（建议修改） |

> **安全建议：** 生产环境中务必设置 `API_KEY`，并修改 `POSTGRES_PASSWORD` 和 `NEO4J_PASSWORD`。

### API Key 使用方式

客户端请求时二选一：

```
# 方式一：标准 Header
X-API-Key: <your-key>

# 方式二：兼容 mem0ai / OpenClaw 客户端
Authorization: Token <your-key>
```

未提供或错误的 Key 返回 **401**，并在服务日志中记录客户端 IP、请求方法、路径及 Headers。

---

## 步骤四：配置域名和 HTTPS

1. 在 Dokploy 服务的 **Domains** 面板中点击 **Add Domain**
2. 填写域名（如 `mem0.example.com`）
3. **Container Port** 填写 `8000`、**Service Name** 填写 `mem0`
4. 开启 **HTTPS**，选择 Let's Encrypt 自动签发证书
5. 保存后，Dokploy 会自动通过 Traefik 完成反向代理配置

---

## 步骤五：首次部署

1. 在 Dokploy 服务页面点击 **Deploy** 按钮
2. Dokploy 会依次执行：
   - 从 GitHub 拉取代码到服务器本地
   - 以 `./server` 为 build context 执行 `docker build`
   - 启动 postgres、neo4j，等待健康检查通过
   - 启动 mem0 容器
3. 在 **Logs** 面板观察日志，neo4j 首次启动需约 60-90 秒，正常完成后可见：
   ```
   Uvicorn running on http://0.0.0.0:8000
   ```

---

## 步骤六：配置自动部署（推送触发）

每次向 `dokploy` 分支推送代码后，让 Dokploy 自动拉取并重新构建：

1. 在 Dokploy 服务的 **General** 面板中，找到 **Webhook URL**，复制该地址
2. 打开 GitHub 仓库 → **Settings** → **Webhooks** → **Add webhook**
3. **Payload URL** 粘贴刚才复制的地址
4. **Content type** 选择 `application/json`
5. **Which events** 选择 `Just the push event`
6. 勾选 **Active**，点击 **Add webhook**

此后每次 `git push origin dokploy`，Dokploy 都会自动触发重新拉取代码、重新构建镜像并重启服务。

### 手动触发更新

如果不使用 Webhook，也可在有新代码后在 Dokploy 控制台手动点击 **Deploy** 触发。

---

## 步骤七：验证部署

```bash
# 访问 OpenAPI 文档
open https://mem0.example.com/docs

# 创建一条测试记忆
curl -X POST https://mem0.example.com/memories \
  -H "Content-Type: application/json" \
  -d '{
    "messages": [{"role": "user", "content": "我叫 Alice"}],
    "user_id": "test-user"
  }'
```

---

## 数据持久化说明

Dokploy 会自动管理以下命名卷，数据在容器重启或重新部署后不会丢失：

| 卷名 | 挂载路径 | 内容 |
|------|----------|------|
| `postgres_db` | `/var/lib/postgresql/data` | 向量数据 |
| `neo4j_data` | `/data` | 知识图谱数据 |
| `history_data` | `/app/history` | 操作历史（SQLite） |

---

## 常见问题

**Q: mem0 服务启动失败，日志显示无法连接数据库**

neo4j 启动较慢，`depends_on` 的健康检查会等待它就绪。如果仍然失败，可在 Dokploy 中手动重启 mem0 服务。

**Q: 如何切换回 OpenAI 官方接口？**

在 Dokploy Environment 面板中将以下变量改为：

```
OPENAI_BASE_URL=https://api.openai.com/v1
LLM_MODEL=gpt-4o
EMBEDDING_MODEL=text-embedding-3-small
EMBEDDING_DIMS=1536
```

**Q: 如何查看 Neo4j 浏览器界面？**

Neo4j 浏览器运行在容器内部 7474 端口，生产环境不建议直接暴露。如有需要，可在 Dokploy 中为 neo4j 服务单独添加一个域名，Container Port 填 `7474`、Service Name 填 `neo4j`。
