---
kind: external_dependency
name: RAGFlow Cloud 云服务入口
slug: ragflow-cloud
category: external_dependency
category_hints:
    - vendor_identity
scope:
    - '**'
---

RAGFlow 官方托管云服务，提供与开源版本一致的 RAG + Agent 能力。仓库 README 将其作为「Get Started」入口（https://cloud.ragflow.io），企业版功能文档也与之配套发布。部署时可通过环境变量 `VITE_RAGFLOW_ENTERPRISE` 切换前端是否展示企业版管理后台与水印等能力；生产环境通常以自托管方式运行，Cloud 仅作为试用/演示入口。