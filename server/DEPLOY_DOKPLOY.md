# Mem0 Server — Dokploy 部署指南

## 概述

本指南介绍如何在 [Dokploy](https://dokploy.com) 上部署 Mem0 Server。

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

---

## 步骤一：准备 Docker Compose 配置

在 Dokploy 控制台中使用以下 `docker-compose.yml` 内容（生产版本，去除了开发模式的热重载挂载）：

```yaml
name: mem0

services:
  mem0:
    image: ghcr.io/mem0ai/mem0-server:latest  # 或者填写你自己构建并推送的镜像地址
    # build:                                   # 如果使用源码构建，取消注释以下两行
    #   dockerfile: server/Dockerfile
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

> **注意：** 与开发版相比，生产配置做了以下调整：
> - 移除了 mem0 库源码和 server 代码的热重载挂载
> - 历史数据库改为命名卷 `history_data`（持久化更可靠）
> - neo4j 健康检查间隔适当放宽，避免频繁检查

---

## 步骤二：在 Dokploy 中创建项目

1. 登录 Dokploy 控制台
2. 点击 **Create Project**，填写项目名称（如 `mem0`）
3. 在项目内点击 **Create Service** → 选择 **Docker Compose**
4. **Source** 选项中选择 **Raw**，将上方的 `docker-compose.yml` 内容粘贴进去
5. 点击 **Save**

---

## 步骤三：配置环境变量

在 Dokploy 服务的 **Environment** 面板中，添加以下变量：

### 必填

| 变量 | 说明 |
|------|------|
| `OPENAI_API_KEY` | 硅基流动或其他服务的 API Key |

### 可选（有默认值）

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `OPENAI_BASE_URL` | `https://api.siliconflow.cn/v1` | LLM/Embedding API 地址 |
| `LLM_MODEL` | `Qwen/Qwen2.5-72B-Instruct` | 使用的 LLM 模型 |
| `LLM_TEMPERATURE` | `0.2` | 生成温度 |
| `LLM_MAX_TOKENS` | `2000` | 最大输出 Token 数 |
| `EMBEDDING_MODEL` | `BAAI/bge-m3` | Embedding 模型 |
| `EMBEDDING_DIMS` | `1024` | 向量维度 |
| `POSTGRES_PASSWORD` | `postgres` | PostgreSQL 密码（建议修改） |
| `NEO4J_PASSWORD` | `mem0graph` | Neo4j 密码（建议修改） |

> **安全建议：** 生产环境中务必修改 `POSTGRES_PASSWORD` 和 `NEO4J_PASSWORD`，确保两处（服务定义和 `NEO4J_AUTH`）保持一致。

---

## 步骤四：配置域名和 HTTPS

1. 在 Dokploy 服务的 **Domains** 面板中点击 **Add Domain**
2. 填写域名（如 `mem0.example.com`）
3. **Container Port** 填写 `8000`
4. 开启 **HTTPS**，选择 Let's Encrypt 自动签发证书
5. 保存后，Dokploy 会自动通过 Traefik 完成反向代理配置

---

## 步骤五：部署

1. 点击 **Deploy** 按钮
2. 在 **Logs** 面板观察启动日志，注意：
   - neo4j 首次启动需要约 60-90 秒完成初始化，mem0 服务会等待其健康检查通过后再启动
   - 正常启动后可看到 `Uvicorn running on http://0.0.0.0:8000`

---

## 步骤六：验证部署

服务启动后，访问以下地址验证：

```
# OpenAPI 文档
https://mem0.example.com/docs

# 健康探测（创建一条测试记忆）
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

将环境变量设置为：

```
OPENAI_BASE_URL=https://api.openai.com/v1
LLM_MODEL=gpt-4o
EMBEDDING_MODEL=text-embedding-3-small
EMBEDDING_DIMS=1536
```

**Q: 如何查看 Neo4j 浏览器界面？**

Neo4j 浏览器运行在容器内部 7474 端口，生产环境不建议直接暴露。如有需要，可在 Dokploy 中为 neo4j 服务单独添加一个域名，Container Port 填 `7474`。
