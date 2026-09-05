---
kind: dependency_management
name: 多语言依赖管理：Go/Python/前端三套声明式锁定 + 私有镜像与二进制预缓存
category: dependency_management
scope:
    - '**'
source_files:
    - go.mod
    - go.sum
    - pyproject.toml
    - uv.lock
    - web/package.json
    - web/package-lock.json
    - admin/client/pyproject.toml
    - sdk/python/pyproject.toml
    - agent/sandbox/pyproject.toml
    - ragflow_deps/download_deps.py
    - ragflow_deps/download_go_deps.py
    - .github/workflows/sep-tests.yml
---

## 1. 使用的系统与方法

RAGFlow 是一个 Go + Python + React 的混合仓库，依赖管理按语言分治，各自使用官方生态的标准工具并配合企业级约束：

- **Go**：`go.mod` + `go.sum` 双文件锁定（module `ragflow`，Go 1.26.4），通过 `replace` 指令把 `github.com/infiniflow/infinity-go-sdk`、`github.com/AkmalOt/gomsg` 等第三方包替换为内部维护版本；CI 中设置 `GOPRIVATE=github.com/infiniflow/onnxruntime_go` 以访问私有 fork。
- **Python**：根目录 `pyproject.toml` 用 PEP 621 声明直接依赖，`uv.lock` 作为 uv 的完整锁文件（含 hash、wheel URL）；子项目 `admin/client/pyproject.toml`、`sdk/python/pyproject.toml`、`agent/sandbox/pyproject.toml` 各自独立发布。所有 Python 包统一通过阿里云 PyPI 镜像 `https://mirrors.aliyun.com/pypi/simple` 拉取，仅 `trio` 显式走 `pypi.org`。
- **前端**：`web/package.json` 声明依赖，`web/package-lock.json`（lockfileVersion 3）锁定全部 npm 包解析结果；Node 版本通过 `engines.node >= 18.20.4` 约束。
- **二进制/非 Python 依赖**：`ragflow_deps/` 目录预缓存 Chrome/chromedriver、OfficeOxide、PDFium、Tika JAR、Stagehand、uv 二进制、NLP 模型 (`cl100k_base.tiktoken`) 以及 `libssl1.1` deb 包，由 `download_deps.py` / `download_go_deps.py` 在构建时校验并注入 Docker 镜像。

## 2. 关键文件

| 语言 | 清单/锁文件 | 作用 |
|---|---|---|
| Go | `go.mod`, `go.sum` | 模块声明、依赖版本锁定、`replace` 重定向 |
| Python | `pyproject.toml`, `uv.lock` | 直接依赖、测试依赖组、约束依赖（CVE 修复）、索引与覆盖 |
| 前端 | `web/package.json`, `web/package-lock.json` | npm 依赖与精确解析树 |
| 二进制 | `ragflow_deps/Dockerfile`, `download_deps.py`, `download_go_deps.py` | 预下载并打包非 pip/go/npm 的二进制/模型 |
| CI | `.github/workflows/sep-tests.yml` | 设置 `GOPRIVATE` 控制私有模块访问 |

## 3. 架构与约定

### 3.1 版本策略
- **直接依赖**：核心库采用“精确版本”或“窄范围”（如 `anthropic==0.76.0`、`litellm==1.84.0`、`quart-auth==0.11.0`），避免上游破坏性升级。
- **条件依赖**：通过 `; sys_platform == ...` 区分平台（如 `onnxruntime` vs `onnxruntime-gpu`），并通过注释强制与 `internal/common.DeepDocORTVersion` 保持一致。
- **约束依赖（constraint-dependencies）**：在 `pyproject.toml` 的 `[tool.uv]` 中以 CVE 编号为注释来源，集中提升被间接引入的脆弱传递依赖（`pyasn1>=0.6.4`、`urllib3>=2.7.0`、`lxml>=6.1.1`、`protobuf>=5.29.6`、`azure-core>=1.38.0` 等）。
- **覆盖/排除**：`override-dependencies` 解决 `attrs` 版本冲突；`exclude-dependencies` 剔除 `unclecode-litellm`、`agentrun-mem0ai` 等无用的传递依赖。

### 3.2 私有源与镜像
- Python 默认从阿里云镜像拉取，`trio` 因不在镜像上而通过 `[[tool.uv.index]] explicit = true` 指定回 pypi.org。
- Go 通过 `replace` 将 `infinity-go-sdk` 指向内部分支，并通过 `GOPRIVATE` 允许拉取 `github.com/infiniflow/onnxruntime_go`（org-owned fork of yalue/onnxruntime_go）。
- 前端 `package-lock.json` 中的 resolved URL 显示实际拉取自 `registry.npmmirror.com`（淘宝镜像），表明 CI/本地已配置 npm 镜像。

### 3.3 二进制依赖隔离
`ragflow_deps/` 是“离线可复现”的关键：Dockerfile 不联网安装系统库，而是先运行 `download_deps.py` / `download_go_deps.py` 把 Chrome、PDFium、Tika、uv 等二进制及 NLP 模型下载到镜像层，保证构建与部署完全可重现且不依赖外部网络。

### 3.4 多子项目隔离
- `admin/client`、`sdk/python`、`agent/sandbox` 各自拥有独立的 `pyproject.toml` + `uv.lock`，可单独发布为 `ragflow-cli`、`ragflow-sdk` 等包。
- Go 后端只有一个 module `ragflow`，所有 Go 代码共享同一份 `go.mod`/`go.sum`。

## 4. 约定与约束

- **Python 必须使用 uv**：根 `pyproject.toml` 与 `uv.lock` 共同存在，且 CI/文档中均以 `uv` 作为安装入口；新增依赖应同时更新 `pyproject.toml` 和重新生成 `uv.lock`。
- **传递依赖安全基线不可绕过**：`[tool.uv].constraint-dependencies` 中列出的最低版本（如 `urllib3>=2.7.0`、`lxml>=6.1.1`、`nltk>=3.10.0`）是安全红线，任何新引入的包不得将其降级。
- **Go replace 需加注释说明原因**：现有 `replace` 指令均附带注释解释为何替换（如 onnxruntime_go 的静态链接语义），新增替换也应遵循此惯例。
- **平台相关依赖必须显式标注**：如 `onnxruntime` 系列使用 `sys_platform` 标记，新增此类依赖时应沿用相同模式。
- **二进制依赖必须纳入 ragflow_deps**：任何需要预下载的运行时二进制（浏览器驱动、OCR 引擎、NLP 模型等）都应放入 `ragflow_deps/` 并由 `download_deps.py` 管理，禁止在 Dockerfile 中临时 `wget`。
- **前端依赖锁定不可忽略**：`web/package-lock.json` 必须随 `package.json` 一起提交，变更依赖后需重新生成 lock 文件。
