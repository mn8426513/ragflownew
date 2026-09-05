---
kind: external_dependency
name: Infinity 向量/全文检索引擎（Elasticsearch 替代）
slug: infinity
category: external_dependency
category_hints:
    - migration_status
    - client_constraint
scope:
    - '**'
---

RAGFlow 默认使用 Elasticsearch 存储全文与向量，但通过设置 `DOC_ENGINE=infinity` 可切换到 InfiniFlow 自研的 Infinity 引擎。切换需先停容器、改 docker/.env、再启动；Linux/arm64 平台尚未正式支持该切换。映射文件位于 `conf/doc_meta_infinity_mapping.json` 与 `conf/infinity_mapping.json`，过滤逻辑在 `common/metadata_infinity_filter.py`。当前代码库同时存在 ES 与 Infinity 两套 mapping/filter，属于可选迁移目标而非已强制替换。