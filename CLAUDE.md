# CLAUDE.md

本文件为 Claude Code（claude.ai/code）在操作本仓库时提供指导。

## 项目概述

`ezdata` 是一个数据处理、分析与任务调度系统，采用 Python/Flask 后端（`api/`）和 Vue 3 前端（`web/`，基于 JeecgBoot-Vue3 改造）。后端主要负责多数据源接入、统一数据模型、LLM/RAG 数据问答、低代码 ETL 数据集成、单任务与 DAG 工作流调度（基于 Celery）、以及代码沙箱执行服务。

## 仓库结构

- `api/` — Python Flask 后端、Celery Worker、调度器、沙箱服务
- `web/` — Vue 3 + Vite + Ant Design Vue 前端
- `deploy/` — Docker、Kubernetes 与本地部署资源
- `docs/` — 项目文档

## 后端（`api/`）

### 技术栈

- Flask + Flask-SQLAlchemy + Flask-CORS
- Celery + Redis（作为 broker/backend），并使用 `celery_once` 防止任务重复执行
- APScheduler 实现定时调度
- MySQL 作为默认持久化数据库
- Elasticsearch 用于日志存储（当 `LOGGER_TYPE=es` 时）
- MindsDB 适配器用于 ETL 数据源接入

### 服务入口

| 服务 | 文件 | 默认端口 | 说明 |
|---------|------|--------------|---------|
| Web API | `web_api.py` | 8001 | 主 REST API，通过 `blueprints.py` 注册所有蓝图 |
| 调度器 API | `scheduler_api.py` | 8002 | 基于 APScheduler 的任务调度服务 |
| 沙箱 API | `sandbox_api.py` | 8003 | 受限的 Python/Shell/数据处理执行沙箱 |
| Celery Worker | `tasks/__init__.py` | — | 分发 `normal_task`、`dag_task`、`dag_node_task` 等任务 |
| Flower | `celery_app.py` | 5555 | Celery 监控界面（`celery -A tasks flower`） |

### 配置说明

后端配置由 `config.py` 从环境文件加载：

```python
dotenv_path = os.environ.get('ENV', 'prod.env')   # 默认使用 prod.env
load_dotenv(dotenv_path=dotenv_path)
```

设置 `ENV=dev.env`（或 `api/` 下的其他文件）并设置 `read_env=1` 即可覆盖默认配置。`dev.env` 已作为开发模板提供。主要配置分组包括：`DB_*`、`REDIS_*`、`ES_HOSTS`、`OSS_*`/`S3_*`、`SANDBOX_*`、`LLM_*`、`RAG_*`。

### 常用后端命令

在 `api/` 目录下执行：

```bash
# 安装依赖
pip install -r requirements.txt -i https://pypi.doubanio.com/simple
pip install -r etl/requirements.txt -i https://pypi.doubanio.com/simple

# 启动 Web API（端口 8001）
python web_api.py

# 启动调度器 API（端口 8002）
python scheduler_api.py

# 启动沙箱 API（端口 8003）
python sandbox_api.py

# 启动 Celery Worker（Linux）
celery -A tasks worker

# 启动 Celery Worker（Windows）
celery -A tasks worker -P eventlet

# 启动 Celery Flower 监控
celery -A tasks flower
```

### 后端架构

- `web_apps/__init__.py` 创建 Flask 应用实例和 SQLAlchemy 的 `db` 实例。
- `blueprints.py` 是所有 API 模块的唯一注册表。每个条目将一个模块路径映射到 URL 前缀，例如 `web_apps.datasource.views.datasource_bp` → `/api/datasource`。新增业务域时，需创建包含蓝图的 `views.py` 并在此注册。
- `models.py` 定义了共享的 `BaseModel`（包含软删除、租户、审计字段）和系统表（`User`、`Role`、`Depart`、`Permission` 等）。
- `web_apps/<domain>/` 通常包含：
  - `views/*.py` — Flask 蓝图与路由处理函数
  - `services/` — 业务逻辑
- `tasks/` — Celery 任务定义。`tasks/__init__.py` 中的 `task_dict` 供调度器和 Worker 使用。
- `etl/` — 数据集成引擎：`registry.py` 解析读写器，`transform_algs.py` 包含内置转换算法，`etl_task.py` 负责分批执行抽取/处理/加载。
- `utils/` — 共享工具：认证、缓存、DAG 辅助、查询构建器、存储、沙箱辅助、日志、校验等。
- `mindsdb/` — MindsDB 处理程序集成，用于扩展更多数据库连接器。

### 安全说明

- `sandbox_api.py` 执行用户提供的 Python 与 Shell 代码。它使用 `RestrictedPython`（配置启用时）、基于 AST 的危险模式过滤，以及可配置的允许模块白名单（`SANDBOX_ALLOWED_MODULES`）。
- `dev.env` 中的 `SAFE_MODE` 控制动态代码是在沙箱服务中运行，还是在主 API 进程中运行。

## 前端（`web/`）

### 技术栈

- Vue 3、Vite 6、TypeScript 4.9
- Ant Design Vue 4、Vxe Table、Pinia、Vue Router 4
- 基于 JeecgBoot-Vue3 的项目结构（`src/api`、`src/views`、`src/router` 等）
- Jest 单元测试（目前仅有一个冒烟测试）

### 配置说明

- `.env.development` — 开发服务器代理后端地址 `http://127.0.0.1:8001/api`。
- `vite.config.ts` — 路径别名：`/@/` 和 `@/` 映射到 `src/`，`/#/` 和 `#/` 映射到 `types/`。
- `tsconfig.json` — 开启 strict，支持 `tsx`/`vue`，paths 与 Vite 别名保持一致。

### 常用前端命令

在 `web/` 目录下执行：

```bash
# 安装依赖（文档使用 pnpm，npm --legacy-peer-deps 也可）
pnpm install

# 启动开发服务器
pnpm dev

# 生产构建
pnpm build

# 带包体积分析的构建
pnpm build:report

# 预览生产构建
pnpm preview

# 使用 Prettier 格式化
pnpm batch:prettier

# 运行 Jest 冒烟测试
npx jest
```

### 前端架构

- `src/main.ts` 负责应用启动，初始化 Pinia、i18n、路由守卫、全局组件，并注册 LLM 聊天路由。
- `src/api/` — 按域分组的 HTTP API 客户端（`sys`、`model`、`demo`、`common` 等）。
- `src/views/` — 页面组件，与后端业务域对应（`dataManage`、`task`、`llm`、`rag`、`alert`、`algorithm` 等）。
- `src/store/modules/` — Pinia 状态模块（app、user、permission 等）。
- `src/router/` — 路由定义、路由守卫、菜单生成与辅助工具。
- `src/components/` — 共享组件与全局注册组件。
- `src/utils/http/` — Axios 配置与拦截器。
- 当设置 `VITE_GLOB_QIANKUN_MICRO_APP_NAME` 时，项目可作为 Qiankun 子应用运行。

## 数据库初始化

数据库 SQL 脚本位于 `api/sql/ezdata.sql`，启动服务前需导入 MySQL。后端 `api/models.py` 中的模型也可用于创建表：

```bash
cd api
python -c "from web_apps import db, app; from models import *; app.app_context().push(); db.create_all()"
```

## 部署

- `deploy/docker/` 与 `deploy/kubernetes/` 包含编排资源。
- `api/Dockerfile` 基于 Conda 构建镜像，安装后端依赖，并通过 `install_handlers.py` 可选安装 MindsDB 处理程序。
- `api/docker-compose.yml` 通过 `api/init.sh` 与 `api/supervisord.ini` 启动完整后端栈（Web API、调度器、Celery Worker、Flower）。

## 开发约定

- 后端模块使用中文注释和文档字符串，请保持此风格。
- 后端路由蓝图通过 `utils.common_utils.import_class` 按字符串路径动态导入。
- 前端路径别名：`@/foo` 和 `/@/foo` → `src/foo`；`#/foo` 和 `/#/foo` → `types/foo`。
- 前端生产构建会移除 `console` 与 `debugger`（`esbuild.drop`）。
