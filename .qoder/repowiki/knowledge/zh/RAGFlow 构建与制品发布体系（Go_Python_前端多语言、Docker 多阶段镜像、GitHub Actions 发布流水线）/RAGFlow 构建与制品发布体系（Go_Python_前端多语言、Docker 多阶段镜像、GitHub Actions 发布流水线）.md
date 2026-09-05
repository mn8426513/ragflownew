---
kind: build_system
name: RAGFlow 构建与制品发布体系（Go/Python/前端多语言、Docker 多阶段镜像、GitHub Actions 发布流水线）
category: build_system
scope:
    - '**'
source_files:
    - build.sh
    - go.mod
    - pyproject.toml
    - Dockerfile
    - Dockerfile_go
    - .github/workflows/release.yml
    - docker-compose.yml
    - web/package.json
    - helm/Chart.yaml
    - helm/values.yaml
    - docker/entrypoint.sh
    - docker/entrypoint-go.sh
    - ragflow_deps/Dockerfile
---

## 1. 构建系统总览

RAGFlow 是一个 **Go + Python + React** 的多语言项目，构建体系围绕以下核心工件展开：
- Go 后端可执行文件 `bin/ragflow_server` 与 CLI `bin/ragflow-cli`（通过 CGO 静态链接 C++ 分词器、office_oxide、pdfium、pdf_oxide、ONNX Runtime）。
- Python 运行时环境（基于 uv 锁定 `pyproject.toml` + `uv.lock`，Python 3.13）。
- 预编译的 Web 前端静态资源（Vite + React，由 Node.js 22 构建）。
- Docker 镜像 `infiniflow/ragflow`（多阶段构建，生产镜像仅含运行期依赖）。
- Helm Chart（`helm/`）用于 Kubernetes 部署。
- GitHub Actions 发布流水线 `.github/workflows/release.yml`，触发条件为 `v*.*.*` 标签或每日定时任务（`nightly` 可变标签）。

## 2. 关键脚本与配置文件

| 角色 | 关键文件 | 说明 |
|---|---|---|
| Go/C++ 构建入口 | `build.sh` | 统一入口，支持 `--all/--cpp/--go/--test/--run/--clean/--strip` 等子命令；负责检测 cmake/clang++/pcre2、下载并校验 office_oxide/pdfium/pdf_oxide/onnxruntime 原生库、调用 `cmake` 构建 `librag_tokenizer_c_api.a`、设置 CGO 环境变量后 `go build -tags cgo,static` |
| Go 模块定义 | `go.mod` | 声明 go 1.26.4 及全部依赖；包含 replace 指令将 `infinity-go-sdk` 指向内部 fork，以及 onnxruntime_go 使用 org-owned fork 的注释 |
| Python 依赖 | `pyproject.toml` + `uv.lock` | 声明 Python ≥3.13、所有依赖版本、`dependency-groups.test`、ruff/pytest/coverage 配置、setuptools 打包包列表 |
| 主镜像构建 | `Dockerfile` | 多阶段：`base` → `builder`（安装 uv、Node 22、`uv sync --frozen`、`npm install && npm run build`）→ `production`（复制 Python venv、源码、Nginx 配置、预编译前端） |
| Go 专用镜像 | `Dockerfile_go` | 纯 Go 后端镜像：`base` → `go-builder`（从 `infiniflow/github_action_runner` 拉取 ORT 静态库、执行 `./build.sh --cpp && ./build.sh --go`）→ `web-builder`（仅构建前端）→ `production`（只拷贝 `bin/ragflow_server` 与 Nginx 配置） |
| 依赖预烘焙镜像 | `ragflow_deps/Dockerfile` | 提供 HuggingFace 模型、tika、chrome/chromedriver、uv、stagehand-server 等二进制，被主镜像以 `FROM ... AS builder` 方式挂载 |
| 编排与启动 | `docker-compose.yml` / `docker/*.yml` / `docker/entrypoint*.sh` | 单机 Docker Compose 拉起 API/Admin/Ingestor/Syncer/Nginx/MinIO/MySQL/Redis/Infinity/Elasticsearch 等；entrypoint 注入 `service_conf.yaml.template` |
| CI 发布流水线 | `.github/workflows/release.yml` | 矩阵式交叉编译 CLI（linux/darwin/windows × amd64/arm64），自托管 runner 上构建服务端二进制（CGO_ENABLED=1），再 `docker build -t infiniflow/ragflow:<tag>` 推送 Docker Hub，并按 tag 前缀决定是否推送到 PyPI |
| Helm 部署 | `helm/Chart.yaml` + `helm/values.yaml` + `helm/templates/*` | 在 K8s 中部署 ragflow、elasticsearch/opensearch、infinity、minio、mysql、redis、ingress 等 |

## 3. 架构与设计约定

### 3.1 Go/C++ 混合构建链
- `build.sh` 是单一事实来源：先 `check_cpp_deps`（cmake、clang++、pcre2），再 `build_cpp` 生成 `internal/binding/cpp/cmake-build-release/librag_tokenizer_c_api.a`，并在构建后用 `objcopy --redefine-syms` 将 re2 符号重命名为私有命名空间 `ragtokre2_`，避免与 ONNX Runtime 内置 re2 冲突导致 SIGSEGV。
- `setup_cgo_env` 通过版本化符号链接（如 `liboffice_oxide.a/v0.1.9/...`）让 Go 构建缓存根据路径变化失效，从而在升级原生库时自动重新链接。
- Linux 下强制使用 `ld.lld` 合并 Chromium 构建的 pdfium 的 `.eh_frame`；同时 `-Wl,--allow-multiple-definition` 允许多个 Rust 静态库共存。
- ONNX Runtime 以 `--whole-archive` + `-Wl,--export-dynamic` 静态嵌入，并通过 `dlopen(NULL)` 解析 `OrtGetApiBase`；macOS 原生构建被显式拒绝（Apple ld64 不支持这些 GNU 链接器标志）。
- 原生库优先从 `/opt/ragflow-native-libs`（CI runner 预置）复制到 `~/ragflow-native-libs` 用户缓存，跳过网络下载。

### 3.2 Python 依赖管理
- 使用 `uv` 作为包管理器与虚拟环境工具，`pyproject.toml` 中 `requires-python = ">=3.13,<3.14"` 严格限定 Python 版本。
- `uv.lock` 冻结所有依赖解析结果；Dockerfile 中通过 `sed` 将 pypi.org 替换为阿里云镜像（`NEED_MIRROR=1`），实现离线/代理环境构建。
- `dependency-groups.test` 集中测试依赖；`tool.ruff` 设置行宽 200、启用 ASYNC lint；`tool.pytest.ini_options` 定义 testpaths、markers（p0-p3/smoke/auth/asyncio）。

### 3.3 前端构建
- `web/package.json` 使用 Vite 7.x + React 18；构建脚本 `npm run build` 输出到 `web/dist`。
- Dockerfile 中 Node.js 仅存在于 `builder` 阶段，最终 production 镜像不含 Node 运行时。
- 构建产物通过 `COPY --from=builder /ragflow/web/dist /ragflow/web/dist` 注入镜像。

### 3.4 镜像分层策略
- `Dockerfile`：`base`（Ubuntu 24.04 + 系统依赖 + uv + Node 22 + Chrome/Chromedriver + Tika + Infinity resource）→ `builder`（`uv sync --frozen` + `npm install/build`）→ `production`（仅复制 Python venv、源码、Nginx 配置、前端 dist、VERSION）。
- `Dockerfile_go`：`base` → `go-builder`（基于 `infiniflow/github_action_runner`，拷贝 ORT 静态库并执行 `build.sh --cpp/--go`）→ `web-builder` → `production`（仅拷贝 `bin/ragflow_server` 与 Nginx 配置，无 Python 运行时）。
- 两个镜像均通过 `ENTRYPOINT ["./entrypoint.sh"]` / `entrypoint-go.sh` 启动，并注入 `conf/service_conf.yaml.template`。

### 3.5 版本与发布
- 版本号来源：`git describe --tags --match=v* --first-parent --always` 写入 `/ragflow/VERSION`，供运行时读取。
- 发布流水线：
  - 触发：`v*.*.*` 标签或每日 cron `0 13 * * *` 打 `nightly` 可变标签。
  - CLI 跨平台构建：`CGO_ENABLED=0`，`-trimpath -ldflags="-s -w -X main.version=... -X main.commit=..."`，产出 `dist/cli/ragflow-cli-<tag>-<os>-<arch>`。
  - 服务端二进制：自托管 runner 上 `./build.sh --cpp && ./build.sh --go`，然后 `docker build -f Dockerfile .` 推送 `infiniflow/ragflow:<tag>` 与 `latest`。
  - PyPI 发布：仅当 tag 以 `v` 开头时，`cd sdk/python && uv publish` 与 `cd admin/client && uv publish`。

## 4. 约定与约束

| 规则 | 来源/证据 |
|---|---|
| Go 后端必须使用 `go 1.26.4`（由 `go.mod` 指定） | `go.mod` 首行 |
| Python 必须使用 3.13（≥3.13 且 <3.14） | `pyproject.toml` `[project] requires-python` |
| 原生库版本必须与脚本中硬编码常量一致（office_oxide 0.1.9、pdfium 7809、pdf_oxide 0.3.73、onnxruntime 1.23.2） | `build.sh` 顶部常量 + 版本校验逻辑（`strings` 匹配） |
| macOS 不允许本地构建含 ORT 的服务端二进制（ld64 不支持 `--whole-archive`/`--export-dynamic`） | `build.sh` 中 `case "$(uname -s)" in Darwin) ... return 1` |
| Linux 下必须安装 `lld`（`ld.lld`）才能链接 pdfium | `build.sh` 中 `command -v ld.lld` 检查 |
| 构建缓存失效通过版本化符号链接实现（而非直接改 .a 路径） | `build.sh` 中 `ln -sf ... v${OFFICE_OXIDE_VERSION}/...` 与注释说明 |
| 生产镜像不包含 Node.js 与构建工具，仅含运行期依赖 | `Dockerfile_go` 注释与 `base` stage 未安装 nodejs |
| 镜像内依赖源可通过 `NEED_MIRROR=1` 切换至阿里云镜像 | `Dockerfile` 中多处 `if [ "$NEED_MIRROR" == "1" ]` 分支 |
| 发布产物 SHA256 校验和随 release 一起上传 | `release.yml` 中 `sha256sum * > SHA256SUMS` |
| 测试分为 unit / integration / e2e / manual / native 多个 tier，通过 `--test-*` 参数与 build tag 控制 | `build.sh` 中 `run_go_tests_tagged` 与各 `--test-*` 分支 |
| Helm chart 名称为 `ragflow`，chart 版本见 `helm/Chart.yaml` | `helm/Chart.yaml` |

## 5. 关键文件清单

- `build.sh` — Go/C++ 构建与测试统一入口
- `go.mod` / `go.sum` — Go 依赖与版本锁定
- `pyproject.toml` / `uv.lock` — Python 依赖与锁定
- `Dockerfile` — 全栈镜像（Go+Python+Web）
- `Dockerfile_go` — 纯 Go 后端镜像
- `Dockerfile_base` / `Dockerfile_ci` / `Dockerfile_deepdoc_oss` / `Dockerfile_tei` — 辅助镜像
- `docker-compose.yml` / `docker/docker-compose-*.yml` — 单机编排
- `docker/entrypoint.sh` / `docker/entrypoint-go.sh` — 容器启动脚本
- `docker/nginx/*.conf` — Nginx 反向代理配置
- `helm/Chart.yaml` / `helm/values.yaml` / `helm/templates/*` — Kubernetes 部署模板
- `.github/workflows/release.yml` — 发布流水线
- `web/package.json` — 前端依赖与构建脚本
- `ragflow_deps/Dockerfile` — 依赖预烘焙镜像
- `cmd/ragflow_server.go` / `cmd/ragflow-cli.go` — Go 服务与 CLI 入口
- `api/ragflow_server.py` / `admin/server/admin_server.py` — Python 服务入口（由 entrypoint 启动）
- `conf/service_conf.yaml.template` — 运行时配置模板
- `tools/scripts/install.sh` / `install.ps1` — 安装包脚本（随 release 发布）
