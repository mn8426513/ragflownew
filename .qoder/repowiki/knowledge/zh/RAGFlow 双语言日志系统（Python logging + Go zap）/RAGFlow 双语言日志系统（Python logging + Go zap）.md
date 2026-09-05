---
kind: logging_system
name: RAGFlow 双语言日志系统（Python logging + Go zap）
category: logging_system
scope:
    - '**'
source_files:
    - common/log_utils.py
    - internal/common/logger.go
    - api/ragflow_server.py
    - admin/server/admin_server.py
    - cmd/ragflow_server.go
    - cmd/ragflow-cli.go
    - api/apps/restful_apis/system_api.py
    - admin/server/routes.py
---

## 1. 使用的系统与框架

RAGFlow 是一个 Python + Go 混合后端项目，日志系统按语言分别实现：
- **Python 侧**：基于标准库 `logging`，通过 `common/log_utils.py` 统一初始化，输出到文件与 stdout。
- **Go 侧**：基于 `go.uber.org/zap` 结构化日志，通过 `internal/common/logger.go` 初始化，同时输出到 stdout 和经 `lumberjack.v2` 轮转的日志文件。

两个子系统都遵循“进程启动时集中初始化、运行时可调整级别、结构化字段记录”的设计。

## 2. 关键文件

| 组件 | 路径 | 作用 |
|---|---|---|
| Python 日志核心 | `common/log_utils.py` | 根 logger 初始化、RotatingFileHandler、包级级别管理、异常包装 |
| Go 日志核心 | `internal/common/logger.go` | zap 全局 Logger/Sugar、JSON/Console 编码器、lumberjack 轮转、Gin HTTP 访问日志中间件 |
| Python API 入口 | `api/ragflow_server.py` | 调用 `init_root_logger("ragflow_server")` |
| Admin 服务入口 | `admin/server/admin_server.py` | 调用 `init_root_logger("admin_service")` |
| Go 主进程 | `cmd/ragflow_server.go` | 解析配置后调用 `common.InitLogger(level, FileOutput, serverName)` |
| CLI 入口 | `cmd/ragflow-cli.go` | 以无文件模式初始化日志（仅 stdout） |
| Python 动态调级 API | `api/apps/restful_apis/system_api.py` | 暴露 `/system/log_level` 接口查询/设置包级日志级别 |
| Admin 动态调级路由 | `admin/server/routes.py` | 同样提供动态调级能力 |

## 3. 架构与约定

### Python 日志
- **初始化**：各 Python 进程在启动时调用 `init_root_logger(logfile_basename)`，将根 logger 的输出定向到 `logs/<logfile_basename>.log`，并附加一个 `StreamHandler` 输出到 stdout。每个进程独立维护自己的日志文件。
- **轮转策略**：使用 `RotatingFileHandler(maxBytes=10MB, backupCount=5)`，单文件最大 10MB，保留最近 5 个备份。
- **格式**：默认格式为 `%(asctime)-15s %(levelname)-8s %(process)d %(message)s`，包含时间、级别、进程号与消息。
- **包级级别控制**：通过环境变量 `LOG_LEVELS` 传入形如 `peewee=WARNING,root=INFO` 的键值对，支持运行时通过 `set_log_level(pkg_name, level)` 动态调整任意包的日志级别；`get_log_levels()` 暴露当前所有包的级别映射。
- **内置降噪**：`peewee`、`pdfminer` 等第三方库默认被压制到 WARNING。
- **异常封装**：`log_exception(e, *args)` 会先 `logging.exception` 打印堆栈，再尝试提取对象 `.text` 字段并作为错误抛出。

### Go 日志
- **初始化**：`InitLogger(level, file, serviceName)` 创建全局 `*zap.Logger` 与 `*zap.SugaredLogger`，并通过 `zap.NewAtomicLevelAt` 支持运行时级别切换。
- **编码格式**：使用自定义 `EncoderConfig`，字段名固定为 `timestamp`、`level`、`service`、`caller`、`msg`、`stacktrace`；时间采用 `2006-01-02 15:04:05.000-07:00` 格式（带毫秒与时区偏移），便于日志聚合系统解析。
- **输出目标**：stdout + lumberjack 轮转文件。默认轮转参数：单文件 100MB、保留 10 份、最多 30 天、是否压缩由 `FileOutput.Compress` 决定。
- **进程标识**：每条日志自动附带 `pid` 字段，并通过 `Named(serviceName)` 区分不同子进程（如 admin、ingestor、syncer）。
- **HTTP 访问日志**：`GinLogger()` 中间件为每个请求输出一条结构化日志，包含 `status`、`method`、`path`、`latency`、`client_ip`、`size`、`has_query`、`query_len`。**刻意不记录原始 query string**，只记录其长度，避免 OAuth code、API key 等敏感信息泄露。
- **级别语义**：`Debug < Info < Warn < Error < Fatal < Panic`；`Error` 包装函数在 debug 模式下额外附带 `%+v` 详细错误信息。
- **运行时调级**：`SetLogLevel(level)` 通过 `atomicLevel.SetLevel` 热更新，无需重启进程。

## 4. 约定与约束

- **进程边界**：每个 Python/Go 进程各自初始化独立的 logger，不存在跨进程共享的日志实例。
- **日志位置**：Python 日志写入项目根目录下的 `logs/`；Go 日志写入 `logs/<filename>`，文件名由启动参数或配置指定。
- **级别来源**：Python 通过 `LOG_LEVELS` 环境变量配置包级级别；Go 通过命令行/配置文件传入 level，并在启动后可能根据配置重新初始化。
- **安全约束**：Go 的 Gin 访问日志明确禁止记录原始 query string，仅记录是否存在及长度，这是代码注释中显式声明的安全约定。
- **调试开关**：Go 提供 `IsDebugEnabled()` 供业务逻辑在 debug 开启时才构造昂贵错误详情；Python 通过包级级别控制实现类似效果。
- **统一入口**：业务代码不应直接 import `logging` 或 `zap`，而应通过 `common.log_utils`（Python）或 `internal/common` 包（Go）提供的封装函数调用，以保证格式一致与可观测性。
- **异常处理**：Python 侧推荐使用 `log_exception` 包装异常；Go 侧通过 `common.Error(msg, err, fields...)` 统一记录错误并附带结构化字段。
- **进程退出**：Go 侧通过 `SyncLog()` 确保关闭前 flush 缓冲；Python 侧依赖标准库 handler 的析构行为。

## 5. 未覆盖范围

- 前端（React/Vite）未发现统一的客户端日志框架，前端日志不在本仓库范围内。
- 没有发现集中式日志收集（如 ELK/Loki）的集成代码，日志以本地文件 + stdout 形式存在，由外部编排层（Docker/K8s）负责采集。
- 没有发现 trace/correlation ID 贯穿 Python 与 Go 两侧的统一机制（尽管 Go 日志包含 caller/service/pid）。