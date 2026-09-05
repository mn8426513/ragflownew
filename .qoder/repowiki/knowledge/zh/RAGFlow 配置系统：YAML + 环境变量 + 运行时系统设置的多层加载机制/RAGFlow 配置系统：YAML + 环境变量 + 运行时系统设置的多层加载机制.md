---
kind: configuration_system
name: RAGFlow 配置系统：YAML + 环境变量 + 运行时系统设置的多层加载机制
category: configuration_system
scope:
    - '**'
source_files:
    - common/config_utils.py
    - common/settings.py
    - common/constants.py
    - conf/service_conf.yaml
    - docker/service_conf.yaml.template
    - docker/.env
    - conf/system_settings.json
    - internal/common/environments.go
    - admin/server/config.py
---

## 1. 整体方案

RAGFlow 采用 **三层配置叠加** 的架构：

| 层级 | 来源 | 作用 | 优先级 |
|---|---|---|---|
| 基础 YAML | `conf/service_conf.yaml` | 服务连接参数（MySQL/ES/Redis/MinIO/OceanBase/Infinity/GaussDB/SereneDB/NATS/ClickHouse/OTEL）及 LLM 默认模型、认证开关、SMTP 等 | 最低 |
| 本地覆盖 YAML | `conf/local.service_conf.yaml` | 部署方私有覆盖，不入库 | 最高 |
| 环境变量 | `.env` / Docker Compose / K8s Secret | 运行时注入（如 `DOC_ENGINE`、`DB_TYPE`、`MINIO_*`、`TEI_MODEL`、`SANBOX_*`、`RAGFLOW_SECRET_KEY` 等），并通过 `${VAR:-default}` 模板渲染到 `service_conf.yaml.template` | 与 YAML 同层，按字段覆盖 |

Python 侧通过 `common/config_utils.py` 的 `read_config()` 自动合并 `local.service_conf.yaml` 与 `service_conf.yaml`，并以 `get_base_config(key, default)` 暴露统一读取接口；Go 侧通过 `internal/common/environments.go` 集中声明所有环境变量常量，并由各模块直接 `os.Getenv` 读取。

## 2. 关键文件与包

- **核心加载器**：`common/config_utils.py` — `load_yaml_conf`、`read_config`、`get_base_config`、`decrypt_database_config`、`update_config`（带 `FileLock` 并发写保护）。
- **配置文件**：`conf/service_conf.yaml`（生产基线）、`conf/local.service_conf.yaml`（部署覆盖）、`docker/service_conf.yaml.template`（Docker 模板，`${VAR:-default}` 占位符由 compose 渲染）。
- **环境变量清单**：`docker/.env`（Compose 默认值）、`internal/common/environments.go`（Go 端常量表，含 200+ 个 `Env*` 常量）。
- **Python 启动入口**：`common/settings.py::init_settings()` — 解析 `DB_TYPE`、`DOC_ENGINE`、`STORAGE_IMPL`、`user_default_llm`、`authentication`、`oauth`、`smtp`、`otel` 等，并实例化存储后端、文档引擎、消息引擎、检索器。
- **系统设置持久化**：`conf/system_settings.json` — 以 JSON 数组形式定义可动态修改的系统变量（如 `sandbox.provider_type`、`mail.*`、`enable_whitelist`），由 Go Admin 服务读写数据库。
- **模型元数据**：`conf/models/*.json`（每个 LLM 厂商一个文件）、`conf/llm_factories.json`、`conf/all_models.json`，由 `common/settings.py` 在启动时加载为 `FACTORY_LLM_INFOS`。
- **Admin 侧配置**：`admin/server/config.py` 复用 `common.config_utils.read_config` 管理 admin 进程配置。

## 3. 架构与约定

### 3.1 YAML 分层加载
`read_config()` 先尝试加载 `conf/local.service_conf.yaml`，再加载 `conf/service_conf.yaml`，后者被前者 `dict.update` 覆盖。因此部署方只需维护 `local.service_conf.yaml` 中的差异项。

### 3.2 环境变量覆盖策略
`get_base_config(key, default=None)` 的默认值优先取 `os.environ.get(key.upper())`，即同名环境变量可覆盖 YAML 中的对应键。例如 `DOC_ENGINE`、`DB_TYPE`、`STORAGE_IMPL`、`REGISTER_ENABLED`、`DISABLE_PASSWORD_LOGIN` 等均通过此方式生效。

### 3.3 数据库连接加密
`decrypt_database_config()` 支持可选的 RSA/自定义解密：当 `encrypt_password=true` 且提供 `private_key`、`encrypt_module` 时，密码字段会被动态解密。`show_configs()` 在日志中自动对 `password`、`secret_key`、`access_key`、`client_secret`、`http_secret_key`、`sas_token` 等敏感字段打码。

### 3.4 多后端选择
- **文档引擎**：`DOC_ENGINE` 决定使用 ES / Infinity / OpenSearch / OceanBase / GaussDB / SereneDB / SeekDB，并在 `init_settings()` 中实例化对应的 `docStoreConn`。
- **对象存储**：`STORAGE_IMPL` 通过 `StorageFactory` 选择 MinIO / Azure SPN / Azure SAS / AWS S3 / OSS / OpenDAL / GCS。
- **元数据库**：`DB_TYPE` 控制 MySQL / PostgreSQL / GaussDB / OceanBase，其中 GaussDB 元数据库通过独立的 `GAUSSDB_METADATA_*` 环境变量隔离，不与 DocEngine 共用配置。

### 3.5 LLM 默认模型
`user_default_llm.default_models` 支持 `chat_model`、`embedding_model`、`rerank_model`、`asr_model`、`vision_model` 五种类型，每项可为字符串或 `{name, factory, api_key, base_url}` 字典；`_resolve_per_model_config()` 将工厂名拼入 model 名称（`model@factory` 格式）。

### 3.6 运行时系统设置
`system_settings.json` 中的条目通过 Admin API 动态写入数据库，Go 侧 `internal/common/system_settings.go` 提供读取接口，实现无需重启即可调整邮件、沙箱等运行时参数。

## 4. 约定与约束

- **禁止硬编码连接串**：所有外部服务地址必须通过 YAML 或环境变量注入，`docker/.env` 中的默认密码标注为“非生产环境不得使用”。
- **本地覆盖文件命名固定**：必须以 `local.` 前缀放在 `conf/` 目录下，否则不会被 `read_config()` 识别。
- **GaussDB 元数据库隔离**：元数据库仅读 `GAUSSDB_METADATA_*` 环境变量，`service_conf.yaml` 中的 `gaussdb` 段专用于 DocEngine，二者不可混用。
- **Secret Key 生成策略**：若未设置 `RAGFLOW_SECRET_KEY`（长度≥32），则从 Redis 的 `ragflow:system:secret_key` 获取或自动生成并缓存，避免 Redis 淘汰导致签名失效。
- **并发安全**：`update_config()` 使用 `filelock.FileLock` 保证多进程同时写 YAML 时的原子性。
- **环境变量命名规范**：Go 端所有环境变量集中在 `internal/common/environments.go` 的 `Env*` 常量中声明，新增变量需在此注册，便于 IDE 提示和静态检查。
- **模板渲染依赖 Compose**：`docker/service_conf.yaml.template` 中的 `${VAR:-default}` 语法由 Docker Compose 在启动时替换，不得手动编辑该模板作为运行配置。
- **敏感信息脱敏**：`show_configs()` 会隐去密码类字段，但 `print_rag_settings()` 不会，调用方需谨慎。