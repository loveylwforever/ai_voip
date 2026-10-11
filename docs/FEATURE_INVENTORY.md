# AI VoIP 智能外呼平台 - 完整功能清单

> **版本日期**: 2026年9月19日  
> **仓库**: https://github.com/loveylwforever/ai_voip  
> **许可**: GPL-3.0  
> **用途**: 为工程师 Jamie 提供产品/工程检查清单

---

## 📋 目录

1. [总览](#总览)
2. [后端功能点](#后端功能点)
3. [前端功能点](#前端功能点)
4. [通话引擎与中间件](#通话引擎与中间件)
5. [权限与多租户](#权限与多租户)
6. [集成与部署](#集成与部署)
7. [代码中可见的缺口](#代码中可见的缺口)

---

## 总览

### 技术栈

| 层级 | 技术选型 |
|------|---------|
| **后端** | Python 3.10, FastAPI, uvicorn, psycopg3, PyJWT, cryptography (AES-GCM token), httpx, websockets, boto3, **usip** |
| **前端** | Vue 3.5, Vite 6, Element Plus, Pinia, Vue Router, Axios, xlsx, Sass |
| **数据库** | PostgreSQL 14+ (使用原生 SQL DDL，非 Alembic/ORM) |
| **包管理** | 后端: `uv` + `pyproject.toml`; 前端: npm |
| **媒体依赖** | ffmpeg (文档提及), MinIO/S3 兼容存储(可选) |

### 项目结构

```text
/workspace
├── README.md                 # 主文档 (中文)
├── LICENSE                   # GPL-3.0
├── index.png, script.png     # UI 截图
├── models/                   # 空占位目录 (.gitkeep); sys_config 引用 ./models
├── ai_voip_backend/          # FastAPI 后端
│   ├── pyproject.toml
│   ├── .env.example
│   ├── docs/API.md
│   ├── sql/pgsql/001…090_*.sql  # 90个数据库脚本文件
│   └── src/ai_voip_backend/
└── web-admin-vue3/           # Vue 管理后台 SPA
    ├── package.json
    ├── .env.example
    └── src/
```

### 架构关系

```text
Vue 管理后台 (端口 5173)
    ↓ HTTP /api/v1/*
FastAPI 后端 (端口 3900)
    ↓ 
    ├─→ PostgreSQL (任务、脚本、中继、会话、媒体元数据...)
    ├─→ 存储 (本地 data/ 或 S3/MinIO)
    ├─→ 服务商 (MiniMax TTS WebSocket, 阿里云百炼 ASR WebSocket/文件, 本地流式 ASR HTTP)
    └─→ [如果 AIVOIP_SIP_RUNTIME_ENABLED=1]
          TaskRuntimeScheduler → SipUsipExecutor (uSIP + RTP 播放/ASR)
```

**默认端口**: 
- API: `3900`
- 管理后台: `5173`
- 静态媒体挂载: `/local-media`

### 模块/包说明

| 模块 | 路径 | 职责 |
|------|------|------|
| **后端入口** | `ai_voip_backend/src/ai_voip_backend/app.py` | FastAPI 应用, CORS, 错误处理, `/api/v1`, 本地媒体挂载, 可选调度器 |
| **服务器** | `ai_voip_backend/src/ai_voip_backend/server.py` | uvicorn 在 `0.0.0.0:3900` 启动 |
| **路由聚合** | `ai_voip_backend/src/ai_voip_backend/api/router.py` | 聚合所有路由模块 |
| **CLI工具** | `ai_voip_backend/src/ai_voip_backend/cli.py` | `list-sql`, `apply-sql`, `bundle-sql`, `bootstrap-admin`, `show-log`, 数据库检查 |
| **前端路由** | `web-admin-vue3/src/router/index.js` | 路由 + 认证/管理员守卫 |
| **菜单配置** | `web-admin-vue3/src/utils/menu.js` | 侧边栏菜单按 `admin` / `both` 分类 |
| **状态管理** | `web-admin-vue3/src/stores/auth.js` | token + 基于角色的菜单 |
| **API客户端** | `web-admin-vue3/src/api/modules.js` | 完整的 API 客户端包装器 |

---

## 后端功能点

### API 路由概览

**总计约 93 个路由处理器**，分布在 `api/routes/` 下的 13 个模块中。

API 文档: `ai_voip_backend/docs/API.md`

| 路径前缀 | 模块文件 | 功能范围 |
|----------|---------|----------|
| `/auth` | `routes/auth.py` | 验证码 + 登录 |
| `/health` | `routes/health.py` | 健康检查 (公开) |
| `/system` | `routes/system.py` | 系统配置 (管理员), `/me` 当前用户 |
| `/users` | `routes/users.py` | 用户CRUD + 中继分配 (管理员) |
| `/storage` | `routes/storage_profiles.py` | 存储配置文件 (管理员) |
| `/providers` | `routes/providers.py` | ASR/TTS 服务商配置 (管理员) |
| `/trunks` | `routes/trunks.py` | SIP 中继管理 |
| `/scripts` | `routes/scripts.py` | 脚本、版本、节点、边、演示 |
| `/contacts` | `routes/contacts.py` | 批次 + 记录 + 导入 |
| `/tasks` | `routes/tasks.py` | 任务、运行、会话 |
| `/qc` | `routes/qc.py` | 质检规则 (管理员)、质检结果 |
| `/media` | `routes/media.py` | 资产、TTS、录音、转写 |
| `/runtime` | `routes/runtime.py` | 脚本步骤/模拟、队列、调度、会话进度 |

### 认证与授权

#### 登录流程
- **验证码**: `services/captcha_service.py` 生成图形验证码
- **登录**: 用户名/密码验证
- **密码加密**: PBKDF2-SHA256 (`security.py`)
- **Token机制**: 
  1. 生成 JWT token (`auth.py`)
  2. 使用 **AES-GCM 加密** JWT token
  3. 返回加密后的十六进制字符串给客户端
  4. 每次请求解密验证 (`dependencies.py`)

#### 角色体系
数据种子文件 `sql/pgsql/090_seed_data.sql` 定义的角色:
- `super_admin` - 超级管理员
- `enterprise_admin` - 企业管理员
- `ops_admin` - 运维管理员
- `agent_user` - 坐席用户
- `qc_admin` - 质检管理员
- `viewer` - 查看者

#### 权限控制
- **管理员门禁**: `permissions.py` 将 `super_admin` / `ops_admin` / `enterprise_admin` 视为管理员
- **依赖注入**: `get_current_admin_user` 用于管理员 API
- **数据范围**: 非管理员按 `created_by` / 中继分配过滤 — **非细粒度 RBAC** (API.md 明确说明)

### 租户/组织管理

**⚠️ 无租户/组织表结构**。
- 单数据库架构
- 资源按用户所有权划分
- 管理员分配中继 (`user_trunk_assignment` 表在 `085_access_control_tables.sql` 中)
- 最接近"多用户"的模型是管理员 vs 操作员，不是 SaaS 多租户

### 活动/任务管理

**"活动" ≈ `call_task`** (呼叫任务)

#### 数据表
- `call_task` (`050_task_tables.sql`)
- `call_task_run` (`087_task_run_tables.sql`) - 用于可重复执行批次

#### 任务配置
- 绑定脚本版本
- 绑定联系人批次
- 绑定中继或中继组
- 并发度 / CPS (每秒呼叫数) / 重试 / 呼叫时间窗口 (JSON 配置)

#### 任务状态
- `draft` - 草稿
- `queued` - 排队中
- `running` - 运行中
- `paused` - 已暂停
- `completed` - 已完成
- `terminated` - 已终止
- `failed` - 失败

#### 调度队列
- `call_task_dispatch` 表管理调度队列
- 状态: `pending|dialing|success|failed|...`

#### 服务模块
- `services/task_service.py` (~913 行代码)
- `services/task_runtime_scheduler.py` - 运行时调度器
- `runtime_service.py` - 运行时队列管理

### 呼叫调度机制

**调度器**: `services/task_runtime_scheduler.py`

**启用条件**: `AIVOIP_SIP_RUNTIME_ENABLED=1`

**调度流程**:
1. 查找运行中的任务
2. 确保调度已排队
3. 遵守任务并发度 / CPS / 全局最大并行数
4. 从池中按可用并发度选择中继
5. 创建会话 → `SipUsipExecutor.start_call(...)`

### ASR / TTS / "AI 智能体" 流程

**⚠️ 注意**: 这**不是 LLM 智能体**。流程是基于**脚本图谱**（节点/边 + 关键词匹配）。

#### 服务商模板

**模块**: `provider_service.py`

支持的服务商:
- 本地 `qwen_stream_asr_http` (Qwen 流式 ASR HTTP)
- 阿里云百炼实时 WebSocket
- MiniMax TTS WebSocket

#### TTS (文本转语音)

**在线 TTS 生成**:
- `tts_generation_service.py` 
- 通过 MiniMax WebSocket 实时生成

#### ASR (语音识别)

**离线文件 ASR**:
- `media_service.py` 
- 调用阿里云文件转写

**实时通话 ASR**:
- `sip_usip_service.py` (`RealtimeAsrTracker`)
- 在等待回复窗口期间进行实时识别

#### 对话分支

**实现位置**: `runtime_service.py`

**分支逻辑**:
- 关键词匹配
- 超时处理
- 无匹配时的后备处理
- 边 (edges) 上的条件判断

#### 播放计划

**模块**: `call_playback_service.py`

**流程链**: 开始 → 播放 → ASR 等待

#### 脚本节点类型

**数据库定义的类型**:
- `start` - 开始节点
- `playback` - 播放节点
- `asr` - 语音识别节点
- `branch` - 分支节点
- `intent` - 意图节点
- `confirm` - 确认节点
- `transfer` - 转接节点
- `fallback` - 后备节点
- `end` - 结束节点

**⚠️ UI 模板限制**: 前端 `ScriptView.vue` 仅暴露: `start`, `playback`, `asr`, `end`

#### 演示脚本

**内置演示** (通过 `POST /scripts/demo/*` 创建):
- 红酒品鉴脚本
- 少儿编程脚本

### CDR (呼叫详单)

**⚠️ 无名为 `cdr` 的表**。呼叫详情存储在:

#### 会话表
- `call_session` - 被叫号码、状态、应答时间、挂机时间、计费时长、通话时长、意图、SIP 呼叫 ID

#### 事件表
- `call_session_event` - 拨号/振铃/应答/挂机/节点/ASR 事件...

#### 变量表
- `call_session_variable` - 会话变量

#### 录音表
- `recording_file` + 转写记录

#### 统计表
- `task_daily_stat` / `trunk_daily_stat` 存在于 SQL 中
- **⚠️ 但无服务/API 写入或提供统计数据**

### 坐席管理

**⚠️ 无软电话坐席模块**。

- 菜单图标命名为 `seats.png`，但实际用于中继/联系人
- 角色包含 `agent_user`，但无坐席在线状态、ACD 坐席、或 WebRTC 电话 UI

### 中继 / SIP 管理

#### 数据表
- `sip_trunk` - SIP 中继
- `sip_trunk_group` - 中继组
- `sip_trunk_group_member` - 中继组成员
- `sip_trunk_health_log` - 中继健康日志

#### 中继类型
- `sip_account` - SIP 账号
- `gateway` - 网关 (`086_trunk_type_fields.sql`)

#### 探测机制
- TCP/UDP 连接检查
- 可选通过 uSIP 进行真实 REGISTER 注册 (`trunk_service.py`)

#### 任务中继池
- 多中继池 + 并发分配 (README + scheduler)

### 队列管理

**⚠️ 操作队列 = 任务调度列表** (`call_task_dispatch`)

**不是呼叫中心 ACD 队列**。

#### 运行时 API
- 入队
- 获取待处理
- 创建会话
- 会话进度

### Webhook

**⚠️ 未找到 Webhook** (无 webhook 路由/服务)

### 管理员功能

**仅限管理员的 UI/API**:
- 存储配置
- 服务商配置
- 中继管理
- 质检规则
- 系统配置
- 用户管理 + 中继分配

**初始化**: `ai-voip-backend bootstrap-admin`

### 配置管理

#### 环境变量
- `AIVOIP_PG_*` - PostgreSQL 配置
- `AIVOIP_JWT_*` - JWT 配置
- `AIVOIP_TOKEN_ENCRYPTION_KEY` - Token 加密密钥
- SIP 运行时 `AIVOIP_SIP_RUNTIME_*` (在 `config.py` 中; **SIP 变量不在 `.env.example` 中**)

#### 数据库配置
- `sys_config` 表: 存储后端、最大并发数、保留天数、模型根目录

#### 日志配置
- `logging_setup.py` → `log/info.log`, `log/error.log`

### 数据库模型与迁移

**版本化 SQL 脚本** (通过 CLI 应用)，**非 ORM 迁移**

| 文件 | 表 |
|------|---|
| `001_extensions.sql` | 扩展 / 辅助函数 |
| `010_base_tables.sql` | roles, users, configs, storage, speech_provider |
| `020_trunk_tables.sql` | SIP 中继/组/健康 |
| `030_script_tables.sql` | audio_asset, script 图谱 |
| `040_contact_tables.sql` | 批次, 联系人, 标签, 黑名单 |
| `050_task_tables.sql` | 任务, 调度, 会话 |
| `060_media_tables.sql` | 录音, 转写, TTS 渲染 |
| `070_qc_tables.sql` | 质检规则/结果/意图 |
| `080_stats_tables.sql` | 日统计 (**仅 schema**) |
| `085_access_control_tables.sql` | 所有权 + 中继分配 |
| `086_trunk_type_fields.sql` | 中继类型字段 |
| `087_task_run_tables.sql` | 任务运行 |
| `090_seed_data.sql` | 角色, 默认存储, sys_config |

**⚠️ 有 schema 但无 API 的功能**:
- `blacklist_phone` CRUD
- `contact_tag*` 操作
- `intent_label` / `session_intent` CRUD
- `sys_operation_log` 写入
- 日统计
- 部分质检命中/意图流程

---

## 前端功能点

### 技术栈与结构

**源码目录**: `web-admin-vue3/src/`

| 模块 | 文件 | 职责 |
|------|------|------|
| **路由** | `router/index.js` | 路由 + 认证/管理员守卫 |
| **菜单** | `utils/menu.js` | 侧边栏菜单按 `admin` / `both` 分类 |
| **状态** | `stores/auth.js` | token + 基于角色的菜单 |
| **API** | `api/modules.js` | 完整的 API 客户端包装器 |
| **布局** | `layouts/AdminLayout.vue` | 管理后台布局 |
| **组件** | `components/` | AppHeader, AppSidebar, StatCard, JsonEditor |
| **视图** | `views/` | 各功能页面 |

### 页面 / 菜单 / 路由

| 路由 | 视图组件 | 受众 | 功能 |
|------|---------|------|------|
| `/login` | `LoginView.vue` | 公开 | 登录页面 |
| `/dashboard` | `DashboardView.vue` | 所有用户 | 统计卡片 + 健康快照 |
| `/trunks` | `TrunkView.vue` | 管理员 | 中继管理 |
| `/scripts` | `ScriptView.vue` (~3147 行) | 所有用户 | 画布编辑器 |
| `/tasks` | `TaskView.vue` | 所有用户 | 任务管理 |
| `/media` | `MediaView.vue` | 所有用户 | 媒体管理 |
| `/contacts` | `ContactView.vue` | 所有用户 | 联系人管理 |
| `/storage` | `StorageProfileView.vue` | 管理员 | 存储配置 |
| `/providers` | `ProviderView.vue` | 管理员 | 服务商配置 |
| `/qc` | `QcView.vue` | 管理员 | 质检规则 |
| `/system` | `SystemConfigView.vue` | 管理员 | 系统配置 |
| `/users` | `UserManagementView.vue` | 管理员 | 用户管理 |
| `/runtime` | `RuntimeView.vue` | 所有用户 | 手动运行时调试 |

### 表单与数据录入

**重型 CRUD 表单**:
- 中继管理
- 服务商配置
- 存储配置
- 联系人管理 (支持 xlsx 导入)
- 任务管理
- 质检规则

**脚本画布**:
- 拖拽节点
- ASR 分支
- TTS vs 本地音频
- 变量配置
- 发布版本

### 仪表板与可视化

**仪表板**:
- 从列表 API 的计数聚合
- **非专用分析图表**

**统计卡片**:
- 任务统计
- 会话统计
- 中继健康

### 媒体管理

**功能**:
- 录音列表
- 预览/下载
- 转写时间线 (客服/客户)

### 基于角色的 UI

#### 路由守卫
- `meta.adminOnly` + `authStore.isAdmin` 重定向

#### 菜单范围
- `scope: 'admin'|'both'`

**⚠️ 角色限制**:
- 角色 `qc_admin` / `viewer` 已种子数据，但**未在 UI 中区分处理** (仅区分管理员 vs 非管理员)

---

## 通话引擎与中间件

### 电话集成

| 方式 | 状态 |
|------|------|
| **uSIP** (Python SIP 客户端) | ✅ 仓库内主要拨号器 (`sip_usip_service.py` ~3244 行) |
| Asterisk / FreeSWITCH / LiveKit / ESL | ❌ **未集成**为呼叫引擎 |
| FreeSWITCH 提及 | 本地 ASR HTTP API 文档显示与 `freeswitch_control` `/start|/chunk|/finish` 兼容 |
| 作者单独的 FS 推流模块 | README 中链接; **不在此仓库中** |

### 呼叫管道 (启用 SIP 运行时时)

**流程**:
1. 调度器创建会话 + 播放包
2. uSIP REGISTER (如需要) → INVITE
3. SDP/RTP: 发送 TTS/缓存音频 PCM; 接收被叫音频
4. 实时 ASR 窗口 → 关键词匹配 → 下一节点 / 后备
5. 挂机; 保存立体声录音; 会话事件进度

**不启用 `AIVOIP_SIP_RUNTIME_ENABLED` 时**:
- 运行时 API 仍支持**模拟/步骤/队列/手动会话进度**用于集成测试

### 媒体处理

#### 存储
- 本地目录默认 `./data` / `data/recorder`
- 静态挂载 `/local-media`
- S3/MinIO 通过 `storage/s3_placeholder.py` (尽管名称如此，已完全实现)

#### 功能
- TTS 缓存资产
- 录音上传/预览/下载
- 通话后文件 ASR

### 实时特性

**WebSocket 客户端**:
- MiniMax TTS
- 阿里云百炼 ASR

**通话中**:
- RTP + 流式 ASR

**⚠️ 无**:
- LiveKit/WebRTC 管理软电话

---

## 权限与多租户

### 多租户架构

**⚠️ 单数据库; 无真正多租户**

#### 实际架构
- 单一数据库实例
- 用户所有权模型
- 中继分配机制 (`user_trunk_assignment`)

**非 SaaS 多租户架构**

### 权限系统

#### 权限模型
**粗粒度**: 管理员 vs 用户

#### 角色使用情况
- 已种子数据的角色: `super_admin`, `enterprise_admin`, `ops_admin`, `agent_user`, `qc_admin`, `viewer`
- **⚠️ 种子角色未充分使用**
- UI 仅区分管理员/非管理员

#### 数据权限
- 非管理员按 `created_by` 过滤
- 中继分配控制访问
- **明确说明**: 无细粒度 RBAC (API.md)

### 访问控制表

**表**: `085_access_control_tables.sql`

- 用户所有权跟踪
- 中继分配 (`user_trunk_assignment`)

---

## 集成与部署

### 第三方集成

| 服务类型 | 服务商 | 状态 |
|----------|--------|------|
| **ASR** | 阿里云百炼 | ✅ WebSocket + 文件 API |
| **ASR** | 本地流式 | ✅ HTTP (qwen_stream_asr_http) |
| **TTS** | MiniMax | ✅ WebSocket |
| **存储** | MinIO/S3 | ✅ 完全实现 |
| **SIP** | uSIP | ✅ 内置 Python SIP 客户端 |
| **Webhook** | - | ❌ 不存在 |
| **CRM 回调** | - | ❌ 不存在 |

### CORS 配置

**硬编码**: localhost/127.0.0.1:5173 & 3900

### 部署

| 方面 | 状态 |
|------|------|
| **Docker** | ❌ 无 Dockerfile |
| **Docker Compose** | ❌ 无 |
| **Kubernetes** | ❌ 无 K8s 配置 |
| **部署方式** | 手动 `uv`/`npm` |

### 可观测性

| 功能 | 状态 |
|------|------|
| **日志** | ✅ 文件日志 (`log/info.log`, `log/error.log`) |
| **指标** | ❌ 无 Prometheus |
| **追踪** | ❌ 无 OpenTelemetry |

### 许可证

**GPL-3.0**

### 模型目录

- 空目录 `models/`
- 配置键 `system.models_root` 存在
- **⚠️ 用途未明确**

---

## 代码中可见的缺口

### 文档中明确提及的未完成功能

**来源**: README / API.md

1. **线路类型**: 仅 SIP 线路类型; **无运营商"对接线路"产品化**; ESL 未完成 (README)
2. **质检**: 这是一个**基础**版本
3. **API.md 过时**: API.md 仍然说真实外呼媒体 / SIP 执行器 / 实时 ASR "未实现" — **相对于 `sip_usip_service.py` + scheduler 已过时** (可能更早编写)
4. **无细粒度 RBAC** (API.md)
5. **作者说明**: 不完整功能供他人扩展

### Schema / UI 缺口 (具体)

| 领域 | 缺口 |
|------|------|
| **黑名单 / 联系人标签** | 仅表; 无 API/UI |
| **日统计** | 仅表 |
| **操作日志** | 仅表 |
| **意图标签 / 质检命中** | 部分 schema; 薄质检服务 (CRUD 规则 + 列表结果) |
| **脚本类型** `branch|intent|confirm|transfer` | 在数据库 CHECK 中; 不在前端模板中; transfer 不在 SIP 执行器中 |
| **LLM 对话智能体** | 缺失 |
| **Webhooks / CRM 回调** | 缺失 |
| **坐席/坐席软电话** | 缺失 |
| **真正的多租户组织** | 缺失 |
| **Docker / 生产部署配置** | 缺失 |
| **`SECURITY.md`** | 引用但缺失 |
| **SIP 环境变量** | 代码中使用, `.env.example` 中缺失 |
| **统计仪表板 / 图表** | 仪表板是简单计数器 |
| **角色** `qc_admin`/`viewer` | 已种子数据, UI 中未区分 |

### 代码中的 TODO 注释

**几乎没有** (`TODO`/`FIXME` 稀疏)。

缺口主要是 README/API 说明和**未使用的 schema**。

### 缺失的安全文档

- README 引用 `SECURITY.md`
- **文件不存在**

### 环境变量不一致

**问题**: SIP 运行时变量在代码中使用 (`config.py`)，但在 `.env.example` 中**缺失**

**影响**:
- `AIVOIP_SIP_RUNTIME_ENABLED`
- `AIVOIP_SIP_RUNTIME_*` 相关变量

---

## 实用心智模型

此仓库是一个**工作中的 SME 外呼脚本机器人平台**: 配置 SIP → 设计关键词脚本 → 导入列表 → 运行任务 → 录音/转写/轻量质检。

### 核心特性
- **"AI" 层**: ASR + TTS + 脚本分支
- **拨号器**: 可选的**进程内 uSIP 拨号器**
- **非**: FreeSWITCH/Asterisk/LiveKit
- **非**: LLM 语音智能体

### 适用场景
- 中小企业外呼场景
- 基于关键词的对话流程
- 简单的质检需求

### 扩展方向
- 集成外部呼叫中心系统
- 增强质检功能
- 添加 LLM 对话能力
- 真正的多租户架构
- 生产级部署配置

---

## 总结

**AI VoIP** 是一个功能齐全但专注于特定用例的智能外呼平台:

### 优势
✅ 完整的后端 API 和前端 UI  
✅ 内置 uSIP 拨号能力  
✅ 脚本画布可视化编辑  
✅ 集成主流 ASR/TTS 服务商  
✅ 基础的任务调度和会话管理  
✅ GPL-3.0 开源许可  

### 局限
⚠️ 非 LLM 对话智能体  
⚠️ 无真正多租户架构  
⚠️ 缺乏生产部署配置  
⚠️ 部分数据库 schema 未使用  
⚠️ 细粒度权限控制有限  
⚠️ 无 Webhook 集成  
⚠️ 统计分析功能基础  

### 建议
对于 Jamie 的工程工作，此清单可作为:
- 功能完整性检查清单
- 扩展开发的起点
- 技术债务评估
- 产品规划参考

---

**文档结束**
