---
sidebar_position: 5
title: 将 RAGFlow 集成到传统 Java 系统
sidebar_label: Java 系统集成
slug: /java_ragflow_integration
---

# 将 RAGFlow 集成到传统 Java 系统

本文面向已有 Java 业务系统、计划增加企业知识检索与问答能力的团队。推荐将 RAGFlow 作为独立服务接入，而非嵌入现有应用：Java 保持业务规则、身份认证、权限与审计职责；RAGFlow 负责文件解析、索引、召回、引用和基于知识的生成。

```text
业务文档 → RAGFlow Dataset → Chat / Retrieval API → Java RAG Adapter → Web、App、客服工作台
```

## 推荐边界

在 Java 服务中增加一个轻量的 `rag-adapter` 模块或独立服务。它根据当前用户、租户和业务域选择允许访问的知识库，调用 RAGFlow，再把答案与引用转换为现有前端协议。浏览器只调用 Java 服务，RAGFlow API Key 必须保存在服务端密钥管理系统中。

首个版本应选择一个边界清晰的场景，例如内部制度问答、产品文档助手或售后知识检索。先准备 30–50 个真实问题，包含预期答案、可接受引用及必须拒答的问题；它们是调优和验收的基准。

## 两种调用方式

### 知识问答（首选 POC）

在 RAGFlow 中创建 Dataset、导入文档、完成解析后创建 Chat assistant。Java 调用 OpenAI 兼容接口：

```http
POST /api/v1/openai/{chat_id}/chat/completions
Authorization: Bearer <RAGFLOW_API_KEY>
Content-Type: application/json
```

请求携带 `messages`、`stream` 和可选的 `extra_body.reference`。Java 将返回的正文和 `references` 分开输出；前端必须显示文档名、页码/位置与引用片段。需要连续会话时，保存 RAGFlow 返回或管理的 `session_id`，并与本系统会话关联。

### 只做检索

若 Java 已经有模型网关、审批流或自定义提示词编排，可调用：

```http
POST /api/v1/retrieval
Authorization: Bearer <RAGFLOW_API_KEY>
```

请求提供 `question`、`dataset_ids`、`similarity_threshold`、`knn_top_k` 和可选 `metadata_condition`；Java 取得 chunks 后执行自身生成流程。该方式灵活，但需要自行承担引用格式、上下文裁剪与防幻觉控制。

## 上下文预算与防幻觉

RAGFlow 通过多层控制降低无关上下文和不可靠回答，但不保证事实绝对正确：

```text
文档分块 → Top-K 候选 → 阈值过滤/重排 → Top-N chunks
      → 知识预算（97%）→ 最终消息预算（95%）→ LLM
```

- **检索裁剪**：`similarity_threshold` 过滤低相关 chunks；`top_k` 控制初始候选池；reranker 调整排序；`top_n` 控制提供给模型的最终 chunks 数。增大 `top_n` 会增加上下文、时延、成本和噪声。
- **知识预算**：RAGFlow 按最终排序累计 chunks，达到模型上下文窗口约 97% 时停止，不会将已选 chunk 截断。较大的文档块可能挤掉后续证据，因此应结合评测调整分块大小。
- **消息预算**：若系统提示词、知识、历史和当前问题超出预算，RAGFlow 会优先保留系统消息与最后一条用户消息，再按 Token 截断内容。Java 不应无限透传会话历史；应限制轮数，并把订单号、用户状态等关键事实作为结构化业务上下文传入。
- **拒答控制**：必须配置 `empty_response`。完全没有可用检索内容时，RAGFlow 直接返回该固定文案；留空会允许模型按自身知识继续回答。注意：只要召回到任意可用内容，模型仍会生成，因此空响应不是“答案完整性”校验。
- **引用与审计**：保持 `quote` 开启，并在 Java 响应中单独返回和展示 `references`。引用提供可追溯证据，不等同于回答已被事实验证。

对于制度、合规和客服场景，建议 Java 再施加业务门禁：要求非拒答回答至少有一个属于当前用户授权范围的引用，并基于评测集确定可接受的最低引用分数；不满足时统一返回“无法从已授权知识中确认”。同时关闭不需要的网页搜索，防止外部内容突破知识范围。

## 知识更新与权限

文档应在业务侧“审核发布”后才进入同步任务：上传到目标 Dataset，轮询解析/索引完成状态，完成后才在 Java 的知识库映射中启用该版本。删除、失效或权限回收也必须同步删除或禁用对应文档，不能只依赖前端隐藏。

多租户优先为租户或强隔离业务域创建独立 Dataset/Chat assistant。共享知识库可使用元数据过滤，但 Java 仍必须先完成授权判定，不能把任意 `dataset_id` 直接交给调用方。

## 上线控制与验收

- 配置明确的空检索响应；没有可靠依据时拒答，不让模型补全事实。
- 使用检索测试集调节分块方式、`similarity_threshold`、`knn_top_k` 和 reranker，而不是只看主观体验。
- 记录请求 ID、租户、知识库版本、命中文档、引用分数、耗时和用户反馈；日志中避免保存 API Key 与不必要的原文。
- 为 Java 调用设置超时、限流和熔断；RAGFlow 不可用时返回可理解的降级提示。

## 建议的交付顺序

1. 部署隔离的 RAGFlow 环境，配置模型并导入一小批脱敏文档。
2. 建立 Dataset、Chat assistant 与评测问题集，完成检索和引用验收。
3. 实现 Java `rag-adapter` 的同步问答与引用返回，再加入 SSE 流式转发。
4. 接入文档发布同步、权限映射、审计和监控，按用户群灰度上线。

更多请求字段见 [HTTP API reference](../references/http_api_reference.md)；API Key 获取与保管方式见 [Acquire RAGFlow API Key](./acquire_ragflow_api_key.md)。
