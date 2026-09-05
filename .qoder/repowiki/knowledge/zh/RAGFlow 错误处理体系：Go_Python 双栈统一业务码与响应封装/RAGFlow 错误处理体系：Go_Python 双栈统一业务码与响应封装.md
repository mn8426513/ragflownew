---
kind: error_handling
name: RAGFlow 错误处理体系：Go/Python 双栈统一业务码与响应封装
category: error_handling
scope:
    - '**'
source_files:
    - internal/common/error_code.go
    - internal/common/http.go
    - internal/handler/error.go
    - internal/service/ingestion_task_service.go
    - api/utils/api_utils.py
    - api/common/exceptions.py
    - common/exceptions.py
    - internal/handler/error_response_test.go
    - internal/handler/error_test.go
---

## 1. 总体方案

RAGFlow 采用 **Go + Python 双后端**，两套语言各自维护一套“业务错误码 + 统一 JSON 响应体”的错误处理约定，并在 HTTP 层通过中间件/处理器集中输出。核心思想是：**业务侧只抛出领域错误或返回 ErrorCode，由 handler 层将其映射为统一的 `{code, message, data}` JSON 响应**，对外不暴露内部异常堆栈。

- Go 端（`internal/`）基于 Gin，使用 `internal/common/error_code.go` 中的 `ErrorCode` 枚举和 `ResponseWithCodeData` / `ErrorWithCode` 等函数统一响应；未捕获的 panic 在 Agent 画布执行器等关键路径用 `recover()` 兜底。
- Python 端（`api/`、`admin/server/`）使用自定义异常类（如 `AdminException`、`TaskCanceledException`、`ModelException`），并通过 `api/utils/api_utils.py` 中的 `server_error_response`、`get_json_result`、`build_error_result` 将异常转换为标准 JSON 响应。

## 2. 关键文件与包

| 语言 | 文件 | 职责 |
|---|---|---|
| Go | `internal/common/error_code.go` | 定义所有业务错误码常量（`CodeSuccess`…`CodeNotImplemented`）、`ErrorCode.Message()`、`CodedError` 以及哨兵错误（`ErrInvalidToken`、`ErrNotFound`、`ErrBucketNotFound`、`ErrTaskNotFound`） |
| Go | `internal/common/http.go` | 提供 `SuccessWithData*`、`ErrorWithCode`、`ResponseWithCodeData`、`ResponseWithHttpCodeData` 等统一响应构造器 |
| Go | `internal/handler/error.go` | 全局 NoRoute 处理、`jsonInternalError`（记录原始 error 但仅返回通用 `CodeServerError` 消息，防止泄露敏感信息）、`IngestionTaskErrorCode` 将服务层特定错误映射到 `ErrorCode` |
| Go | `internal/service/ingestion_task_service.go` | 定义 `InvalidTaskTransitionError`、`TaskStatusConflictError` 等结构化领域错误，供 handler 层 `errors.As` 识别 |
| Python | `api/common/exceptions.py` | Admin 服务端专用异常基类 `AdminException` 及派生（`UserNotFoundError`、`UserAlreadyExistsError`、`CannotDeleteAdminError`、`NotAdminError`） |
| Python | `common/exceptions.py` | 通用异常：`TaskCanceledException`、`ArgumentException`、`NotFoundException`、`ModelException`（含 `retryable` 标记） |
| Python | `api/utils/api_utils.py` | `server_error_response`（Quart 全局异常处理器）、`get_json_result`、`build_error_result`、`RET_CODE_TO_HTTP_STATUS` 映射表，负责把业务 code 映射到 HTTP 状态码 |

## 3. 架构与约定

### 3.1 Go 端错误流

1. **业务层**：service 层返回具体错误类型（如 `*service.InvalidTaskTransitionError`、`*service.TaskStatusConflictError`）或 `common` 包中的哨兵错误（`common.ErrTaskNotFound` 等）。
2. **handler 层**：通过 `IngestionTaskErrorCode(err)` 将领域错误转换为 `common.ErrorCode`，再调用 `common.ErrorWithCode` / `common.ResponseWithCodeData` 输出 JSON。
3. **HTTP 层**：
   - `HandleNoRoute` 对未注册路由返回固定 JSON（404 或兼容 Python 的 `MethodNotAllowed` 语义）。
   - `jsonInternalError` 先 `common.Warn(... zap.Error(err) ...)` 记录完整堆栈，再返回 `CodeServerError` 的通用消息，**禁止将原始 error 字符串直接返回给客户端**（该行为由 `error_response_test.go` 断言保证）。
4. **panic/recover**：仅在隔离的执行边界使用，例如 Agent 画布组件执行器（`internal/agent/canvas/runner.go`、`canvas.go`、`workflowx/parallel.go`）中用 `defer recover()` 捕获组件级 panic，避免拖垮整个请求。

### 3.2 Python 端错误流

1. **业务层**：抛出 `AdminException`（admin server）、`TaskCanceledException`、`ModelException` 等自定义异常，或在普通代码中 `raise Exception("...")`。
2. **全局异常处理器**：`api/utils/api_utils.py::server_error_response(e)` 作为 Quart 的全局异常处理入口，根据异常内容做分类：
   - 包含 `unauthorized`/`401` → 返回 `RetCode.UNAUTHORIZED` 并设置 HTTP 401。
   - 包含 `index_not_found_exception` → 返回“无 chunk 请上传文件”的友好提示。
   - 包含 `not_found` → 返回“chunk not found”。
   - 其他 → 返回 `RetCode.EXCEPTION_ERROR`，message 为 `repr(e)`。
3. **业务码 → HTTP 状态码映射**：`RET_CODE_TO_HTTP_STATUS` 将 `RetCode.*` 映射到真实 HTTP 状态（如 `EXCEPTION_ERROR→500`、`ARGUMENT_ERROR→400`、`PERMISSION_ERROR→403`、`AUTHENTICATION_ERROR→403`），避免 H11 拒绝发送 1xx 状态码导致连接被丢弃。
4. **参数校验**：`@validate_request` 装饰器在解析请求参数时捕获 `AttributeError`、`TypeError`、`WerkzeugBadRequest`、`QuartBadRequest`，返回 `RetCode.ARGUMENT_ERROR`。

### 3.3 统一响应格式

- Go：`{code: ErrorCode, message: string, data: interface{}, total?: int}`，默认 HTTP 200，错误也走 200 + 业务 code（`ErrorWithCode`、`ResponseWithCodeData`）；需要非 200 时用 `ResponseWithHttpCodeData`。
- Python：`{code: RetCode, message: string, data: any}`，通过 `build_error_result` 将 code 映射为 HTTP 状态码。

## 4. 约定与约束

- **禁止向客户端暴露内部错误细节**：Go 的 `jsonInternalError` 强制只返回 `CodeServerError.Message()`，测试用例显式断言响应体不得包含原始 error 中的密码、IP 等敏感信息。
- **业务错误码集中管理**：所有可预见的失败都通过 `internal/common/error_code.go` 中的 `ErrorCode` 常量表达，新增错误需在此处添加常量并补充 `errorMessages` 映射。
- **领域错误必须可被 handler 识别**：service 层抛出的结构化错误（如 `InvalidTaskTransitionError`、`TaskStatusConflictError`）需在 handler 层的 `IngestionTaskErrorCode` 中显式映射到 `ErrorCode`，并由单元测试覆盖（见 `error_test.go`）。
- **Agent 组件 panic 必须被 recover**：组件执行器中使用 `recover()` 包裹单个组件运行，确保某个组件崩溃不会中断整个画布流程；未被 recover 的 panic 会视为严重 bug。
- **Python 全局异常处理器是唯一出口**：API 层不应自行 `try/except` 后直接 `return` JSON，而应让异常冒泡至 `server_error_response`，以保证日志与状态码映射一致。
- **认证/鉴权失败统一映射到 401/403**：Python 端通过 `RET_CODE_TO_HTTP_STATUS` 将 `AUTHENTICATION_ERROR` 映射为 403（兼容 Dify 外部知识 API），Go 端通过 `CodeUnauthorized`/`CodeForbidden` 常量表达。
- **任务状态机错误具象化**：`InvalidTaskTransitionError` 与 `TaskStatusConflictError` 明确携带 `TaskID`、源/目标状态，便于日志定位与前端提示，而非简单返回字符串错误。