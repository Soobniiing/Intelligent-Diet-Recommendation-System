# diet-agent 系统设计

> 面向“不知道吃什么”的日常就餐决策场景，用多 Agent 协作 + 全链路 Trace 可观测 + 离线评估闭环，构建对话式智能饮食推荐系统。

---

## 1. 项目定位

一句话：**一个“聊着天帮你决定今天吃什么”的多 Agent 对话系统**。

核心命题是解决 LLM 黑盒带来的三个工程问题：

1. **不可控** —— LLM 输出不确定，可能错误路由、幻觉推荐、说风险话术；
2. **不可观测** —— 线上出问题无法回放、无法定位是哪一步 LLM 或哪条规则导致的；
3. **不可评估** —— 效果好坏没有量化指标，无法驱动迭代。

对应到项目里的三大设计支柱：

| 设计支柱 | 落地模块 | 解决什么问题 |
|---|---|---|
| 编排中枢 + 专职 Worker | `DietOrchestratorService` + 4 个 Agent | LLM 只做结构化推理，状态流转与路由由 Java 编排层管控 |
| Trace 全链路可观测 | `AgentTraceService` + `diet_request_trace` | 每一轮对话的每个关键节点输入/输出/耗时/Token 全部落库，可回放排查 |
| 离线评估闭环 | `EvaluationService` + LLM Judge + 用户反馈 | 规则 60% + Judge 10% + 用户反馈 30% 加权打分，低分样本回流迭代 |

---

## 2. 技术栈

| 层 | 选型 |
|---|---|
| 语言 / 框架 | Java 21 + Spring Boot 3.3.13 |
| ORM | MyBatis 3.0.4（XML Mapper） |
| 数据库 | MySQL 8（`JSON_OVERLAPS` 做 JSON 标签召回） |
| LLM 框架 | AgentScope（`io.agentscope:agentscope-spring-boot-starter 1.0.11`），底层接阿里云 DashScope |
| 模型 | 主模型 `qwen-max`（推荐理由生成）、轻量模型 `qwen-turbo`（意图/澄清/Judge） |
| 前端 | 原生 HTML/JS 单页应用（hash 路由），由 Spring Boot 静态资源直接托管 |
| 其他 | Lombok、Hutool、Jackson |
