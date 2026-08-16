# 简历"核心设计"逐条解释（面试应对手册）

> 配套 `RESUME.md`。每一条设计都按同一结构展开：
> **一句话解释 → 为什么这样设计 → 涉及文件 → 相关代码 → 面试可能问的问题 → 可能的追问**。
> 面试策略：先讲"是什么 + 为什么"，再等面试官追问；追问到代码层再引用下面的代码片段。


---
AgentScope 核心 API、多 Agent 之间通信方式、会话状态怎么保存、Agent 失败如何重试。

Trace 存储选型、存储量、回放的实现逻辑

Agent 调用 LLM 超时、JSON 解析失败如何处理

多 Agent 之间怎么通信？AgentScope 是进程内还是多进程？会话状态为什么不由 Agent 自己维护？

如果某个 Agent 调用大模型超时 / 返回 JSON 格式错乱，怎么容错处理？

Trace 落库，大量请求下如何避免数据库压力过大？是否做异步落盘？

LLM Judge 做评估，如何解决 LLM 本身打分不稳定的问题？

槽位澄清 Agent 的逻辑，如果用户多次回答缺失参数，你的编排中枢如何控制会话流转？

多 Agent 架构对比单 Prompt 实现推荐，优势是什么？缺点是什么？

## 设计点 1：多 Agent 编排架构

### 一句话解释
把"理解用户 → 追问澄清 → 生成推荐"拆成 4 个专职 Agent（意图识别、槽位澄清、推荐生成、多餐规划），由一个编排中枢（Orchestrator）统一控制流程与状态；每个 Agent 就像"一个用 LLM 实现的纯函数"——输入文本，输出 JSON，不碰业务状态。

### 为什么这样设计
- 最初是一个大 prompt 让 LLM 同时干三件事（分类+抽槽+写回复），输出互相干扰、无法单独优化和评估；
- 拆开后每个 Agent 职责单一：意图识别可以单独测准确率、推荐话术可以单独调 prompt；
- 状态必须收敛：如果让每个 Agent 自己决定下一步，同一句话可能这次走 A 分支、下次走 B 分支，不可测试、不可回放。所以由 Orchestrator 做唯一的路由与状态写入方。

### 涉及文件
- `service/orchestrator/DietOrchestratorService.java`：编排中枢，状态机 + 路由
- `agent/factory/AgentFactory.java`：按 sessionId 缓存一组 Agent（LRU 1000）
- `agent/builder/*.java`：4 个 Agent 的构建器（模型 + prompt）
- `service/intent/IntentAgentService.java`、`service/clarify/ClarifyAgentService.java`、`service/recommend/RecommendResponseAgentService.java`、`service/plan/PlanResponseAgentService.java`：各 Agent 的服务封装

### 相关代码

路由分发（Orchestrator 内，按意图 switch 到不同分支）：

```java
// DietOrchestratorService.handleTurn()
return switch (intent.intent()) {
    case MEAL_RECOMMENDATION, CLARIFY_NEEDED ->
            handleRecommendation(sessionId, userId, request.message(), traceId, state, intent);
    case MEAL_ADJUST -> handleAdjust(sessionId, userId, request.message(), traceId, state, intent);
    case MEAL_PLAN -> handlePlan(sessionId, userId, request.message(), traceId, state, intent);
    case HEALTH_RISK -> handleHealthRisk(sessionId, traceId, state);
    case OTHER -> handleChitchat(sessionId, traceId, state);
};
```

Agent 按会话缓存（AgentFactory，LRU 上限 1000，防止不同用户串线 / 内存泄漏）：

```java
private final Map<String, AgentSet> cache = Collections.synchronizedMap(
        new LinkedHashMap<>(16, 0.75f, true) {
            @Override
            protected boolean removeEldestEntry(Map.Entry<String, AgentSet> eldest) {
                return size() > MAX_AGENT_SETS;
            }
        });

public AgentSet get(String sessionId) {
    return cache.computeIfAbsent(cacheKey(sessionId), ignored -> new AgentSet(
            intentBuilder.build(), clarifyBuilder.build(),
            recommendResponseBuilder.build(), planResponseBuilder.build()));
}
```

Agent 构建示例（IntentAgent 用轻量模型 + 独立 system prompt）：

```java
// IntentAgentBuilder.build()
return ReActAgent.builder()
        .name("diet_intent_agent")
        .model(lightModel)                      // qwen-turbo
        .sysPrompt(promptLoader.load("diet/prompts/intent.txt"))
        .memory(new InMemoryMemory())
        .build();
```

### 面试可能问的问题
1. **为什么要拆多 Agent，一个 Agent 不行吗？**
   → 单一职责、可独立优化/评估/降级；一个 prompt 干多件事会互相干扰。
2. **Agent 之间怎么通信？**
   → 不走消息总线，就是 Java 方法调用 + 结构化对象（IntentResult / ClarifyResult / RecommendResult），这样可单测、可 Trace。
3. **为什么不让 Agent 自己决定下一步？**
   → 状态机会失控；编排层收口后，相同状态必然走相同分支，可测试可回放。

### 可能的追问
- **Agent 有记忆吗？多轮上下文怎么传？**
  → 显式上下文：每轮从 `diet_messages` 取最近 3 轮摘要拼进 prompt；Agent 自带 memory 每次调用前 `clear()`，避免跨轮污染。
- **缓存用 LRU 会不会有问题？**
  → 单机内存态，用户量大时不可扩展；面试时主动说"下一步可改为无状态或 Redis 会话化"会加分。
- **如果新增一个 Agent 要动哪些代码？**
  → 加 Builder → 加 Service 封装（含兜底）→ Orchestrator 加路由/事件 → Trace 埋点 → 评估指标。体现"架构可扩展"。

---

## 设计点 2：检索重排在 LLM 之前（防幻觉）

### 一句话解释
不把餐食库丢给 LLM 让它"选菜"，而是先由 Java 代码用标签精确召回和打分重排，选出 top10（再取 top3）候选，LLM 只负责"为已经选好的菜写推荐理由和口语回复"。

### 为什么这样设计
- 早期版本把整张餐食表塞给 LLM 挑选，出现三类问题：菜多了 prompt 塞不下、LLM 会选库里没有的菜（幻觉）、推荐理由和菜的特征对不上；
- 规则检索有确定性：`mealId` 一定来自数据库，`matchScore` 可解释、可落 Trace、可评估；
- LLM 擅长"表达"（写理由）而不擅长"精确检索"，所以职责切分：**规则选菜，LLM 解释**。

### 涉及文件
- `service/meal/MealService.java` + `mapper/MealMapper.xml`：JSON_OVERLAPS 标签召回
- `service/meal/MealRankService.java`：7 维槽位重叠率打分、排除已推荐、top10
- `service/recommend/RecommendResponseAgentService.java`：取 top3 给 LLM、解析输出、模板兜底
- `resources/diet/prompts/recommend-response.txt`：prompt 强制"mealId 必须来自候选"

### 相关代码

标签召回（MealMapper.xml，MySQL 8 JSON_OVERLAPS，任一维度命中即召回）：

```xml
<select id="search" resultMap="MealItemRowMap">
    SELECT <include refid="MealColumns"/> FROM meal_item
    <where>
        <choose>
            <when test="sourceMode.name() == 'PERSONAL'">
                source_type = 'PERSONAL' AND owner_user_id = #{userId}
            </when>
            <otherwise>
                source_type = 'PUBLIC' AND owner_user_id IS NULL
            </otherwise>
        </choose>
        AND (#{mealTimeJson} = '[]' OR JSON_OVERLAPS(meal_time, #{mealTimeJson}))
        AND (#{moodJson} = '[]' OR JSON_OVERLAPS(mood, #{moodJson}))
        <!-- ...其余 5 维同理 -->
    </where>
    ORDER BY updated_at DESC LIMIT #{limit}
</select>
```

规则重排打分（MealRankService，7 维命中率均值归一化到 0~1）：

```java
private double slotScore(SlotBundle item, SlotBundle query) {
    double total = overlap(item.mealTime(), query.mealTime())
            + overlap(item.mood(), query.mood())
            /* ...其余 5 维 ... */
            + overlap(item.convenience(), query.convenience());
    return clamp(total / 7.0);
}

private double overlap(List<String> itemValues, List<String> queryValues) {
    Set<String> itemSet = Set.copyOf(itemValues == null ? List.of() : itemValues);
    long hits = queryValues.stream().filter(itemSet::contains).count();
    return hits * 1.0 / queryValues.size();   // 用户标签命中占比
}
```

LLM 只拿到 top3 候选 + 禁止编造（RecommendResponseAgentService）：

```java
List<MealItem> topMeals = rankedMeals.stream().limit(3).toList();   // 只给 LLM 前 3 个

// buildUserPrompt()
"""
用户原话：%s
数据源模式：%s
本轮槽位：%s
候选餐食：%s
请输出 JSON，包含 recommendations 数组（每项 mealId + reason）和 speechText，不要编造候选之外的餐食。
""".formatted(userInput, sourceMode, slots, topMeals);
```

### 面试可能问的问题
1. **为什么不让 LLM 直接选菜？**
   → 幻觉（选库外菜）+ 成本（整库塞 prompt）+ 不可解释；规则检索保证候选确定、可审计。
2. **召回和重排分别解决什么问题？**
   → 召回要"宽"（JSON_OVERLAPS 任一维命中，防漏）；重排要"准"（7 维重叠率排序，排除已推荐）。
3. **怎么防止 LLM 推荐理由与菜不符？**
   → 双保险：prompt 强约束 + 只给 top3 候选（LLM 没有机会看到库外菜）。评估里还有 `hallucinationControl` 指标校验卡片 ID 是否都在候选内。

### 可能的追问
- **JSON_OVERLAPS 性能怎么样？**
  → 标签是数组，无法走普通 B+ 树索引，数据量大时全表扫描；面试可答"可加生成列 + 函数索引，或换标签关联表/向量检索"。主动暴露边界反而加分。
- **matchScore 为什么不加权各维度？**
  → 当前各维等权、简单可解释；以后可按业务调权重（如 healthGoal 优先），打分逻辑在 Java 里改起来比 SQL 灵活。
- **top3 是怎么定的？为什么不是 top5？**
  → 前端展示 2~3 个卡片最合适；top3 也控制了 prompt 长度和 Token 成本。

---

## 设计点 3：全链路 Trace 可观测

### 一句话解释
每一轮请求生成唯一 `traceId`，在链路关键节点（HTTP 入口、意图识别、槽位合并、检索、重排、每次 Agent 调用、守卫检查、响应出口）记录"事件"，每个事件包含输入、输出、耗时、Token 等字段，整条链路落库，可按 traceId/sessionId 回放排查。

### 为什么这样设计
- LLM 是黑盒且输出不可复现：线上用户说"昨天推荐得很差"，没有输入输出快照就没法定位；
- 一条链路有 4 个 Agent 调用，报错不知道卡在哪一环；
- 事件化设计的价值：**加新环节 = 加一个新事件**，不需要改表结构（全部塞进 `trace_json`）；且 Trace 数据同时是离线评估的输入，一次埋点两处复用。

### 涉及文件
- `service/trace/AgentTraceService.java`：ThreadLocal TraceScope、recordEvent/recordError/callAgent、close 落库
- `service/orchestrator/DietOrchestratorService.java`：各业务节点埋点（20+ 事件类型）
- `mapper/AgentTraceMapper.java` + `mapper/AgentTraceMapper.xml`：`diet_request_trace` 表读写
- `controller/trace/AgentTraceController.java`：Trace 查询与人工标注接口
- `model/RequestTraceRow.java`：Trace 行模型（含标注字段）

### 相关代码

ThreadLocal 绑定当前请求的 Trace 上下文：

```java
// AgentTraceService
private final ThreadLocal<TraceScope> currentScope = new ThreadLocal<>();

public TraceScope openTrace(String traceId, String sessionId, Long userId) {
    TraceScope scope = new TraceScope(traceId, sessionId, userId);
    currentScope.set(scope);
    return scope;      // 配合 try-with-resources，close() 时统一落库并清理 ThreadLocal
}
```

业务节点埋点（Orchestrator 内调用，一行一个事件）：

```java
agentTraceService.recordEvent("REQUEST_RECEIVED",  "HTTP",   request, initialState);
agentTraceService.recordEvent("INTENT_RECOGNIZED", "INTENT", request.message(), rawIntent);
agentTraceService.recordEvent("SLOTS_MERGED",      "SLOT",   Map.of("stateSlots", state.slots(), "intentSlots", intent.slots()), mergedSlots);
agentTraceService.recordEvent("MEAL_SEARCHED",     "SEARCH", state.slots(), Map.of("candidateCount", candidates.size()));
agentTraceService.recordEvent("MEAL_RANKED",       "RANK",   excludeMealIds, Map.of("rankedCount", ranked.size(), "ranked", ranked));
agentTraceService.recordEvent("REQUEST_FINISHED",  "HTTP",   request, response, elapsedMs(startedAt));
```

统一 Agent 调用入口（模型名、耗时、Token 自动进 Trace，业务不用重复埋点）：

```java
public Msg callAgent(String sessionId, String agentName, String modelName,
                     ReActAgent agent, String inputText) {
    long startedAt = System.nanoTime();
    try {
        Msg response = agent.call(Msg.builder()
                .role(MsgRole.USER).textContent(inputText).build()).block();
        recordAgentCall(sessionId, agentName, modelName, inputText,
                response, elapsedMs(startedAt), null);
        return response;
    } catch (RuntimeException error) {
        recordAgentCall(sessionId, agentName, modelName, inputText,
                null, elapsedMs(startedAt), error);
        throw error;   // 调用方 catch 后走各自的规则兜底
    }
}
```

### 面试可能问的问题
1. **为什么 LLM 项目需要 Trace，普通日志不够吗？**
   → 普通日志只有"发生了什么"，Trace 记录的是"每一轮每个节点的输入/输出/耗时/Token"，LLM 输出不可重放，没有快照就无法复现定位。
2. **Trace 是怎么实现的？ThreadLocal 有什么用？**
   → openTrace 把 TraceScope 放进 ThreadLocal，同一线程内埋点都写进当前 scope，请求结束 close() 统一落库并清理（防止线程池复用污染）。
3. **落库失败会影响业务吗？**
   → 不会，close() 里 catch 掉只打 warn。可观测是增强，不能成为可用性的反向依赖。

### 可能的追问
- **ThreadLocal 的坑？**
  → 必须 finally 清理（否则线程池复用串数据）；异步线程拿不到。本项目同步处理所以可用；将来异步化要改成显式传递 traceId 或 TraceContext 透传。
- **Trace 数据量大怎么办？**
  → 目前同步写 MySQL；可改 MQ 异步 + ES/ClickHouse 独立存储，payload 按需截断（当前单条 20k 字符上限）。
- **Trace 事件有哪些类型？**
  → 能说出 5 个以上即可：REQUEST_RECEIVED、INTENT_RECOGNIZED、INTENT_REVISED、SLOTS_MERGED、ROUTE_SELECTED、MEAL_SEARCHED、MEAL_RANKED、AGENT_CALL、NUTRITION_GUARD_CHECKED、RESPONSE_READY、REQUEST_FINISHED。

---

## 设计点 4：离线评估闭环

### 一句话解释
不靠人肉看对话效果，而是把落库的 Trace 当"测试集"批量打分：规则算客观指标（意图准确率、Token、耗时、幻觉等 10+ 项），LLM Judge 给回复质量打分，再叠加用户反馈，三者按 **60 / 10 / 30** 加权输出百分制报告；低分样本人工标注后回流，驱动 prompt 和规则迭代。

### 为什么这样设计
- 人不可能逐条看完所有对话，也没有量化标准；改一次 prompt 是好是坏说不清；
- 三源互补：规则分客观可复现（主干 60%）、用户反馈是真实信号但稀疏（30%）、LLM Judge 覆盖主观体验但本身不可靠（只给 10%）；
- 评估要和迭代打通：低分样本 → 标注（gold label）→ 改 prompt/规则 → 再评估，形成闭环。

### 涉及文件
- `service/evaluation/EvaluationService.java`：拉 Trace → 解析事件 → 算指标 → 加权总分 → 报告
- `service/evaluation/EvaluationJudgeService.java` + `agent/builder/EvaluationJudgeAgentBuilder.java`：LLM Judge 打分
- `controller/evaluation/EvaluationController.java`：评估入口
- `controller/trace/AgentTraceController.java`：人工标注接口（gold label）
- `mapper/FeedbackMapper.java`：用户反馈查询
- `resources/diet/prompts/evaluation-judge.txt`：Judge 的 system prompt
- `model/EvaluationReport.java`、`model/TraceEvaluationResult.java`：报告与单条结果

### 相关代码

三源加权（EvaluationService，缺失项自动归一，不误伤无标注样本）：

```java
private Double weightedScore(Double ruleScore, Double judgeScore, Double feedbackScore) {
    double weighted = 0.0;
    double weight = 0.0;
    if (ruleScore != null)     { weighted += ruleScore * 0.6;     weight += 0.6; }
    if (judgeScore != null)    { weighted += judgeScore * 0.1;    weight += 0.1; }
    if (feedbackScore != null) { weighted += feedbackScore * 0.3; weight += 0.3; }
    return weight == 0.0 ? null : weighted / weight;
}
```

规则指标采样（从 Trace 事件里解析出评估事实）：

```java
metrics.put("intentAccuracy",  intentAccuracy(row.getExpectedIntent(), snapshot.intent()));
metrics.put("slotAccuracy",    slotAccuracy(row.getExpectedSlots(), snapshot.slots()));
metrics.put("tokenCost",       snapshot.tokenCost() == null ? null : snapshot.tokenCost().doubleValue());
metrics.put("latencyMs",       row.getDurationMs() == null ? null : row.getDurationMs().doubleValue());
metrics.put("fallbackRate",    snapshot.fallbackUsed() ? 1.0 : 0.0);
metrics.put("hallucinationControl", snapshot.hallucinationFree() ? 1.0 : 0.0);
metrics.put("safetyCompliance",     snapshot.safetyCompliance() ? 1.0 : 0.0);
```

幻觉控制指标怎么算（Trace 里对比"重排候选 ID"和"最终卡片 ID"）：

```java
// parseTrace(): rankedIds 来自 MEAL_RANKED 事件，responseIds 来自 RESPONSE_READY 事件
boolean hallucinationFree = responseIds.isEmpty() || rankedIds.containsAll(responseIds);
```

用户反馈映射（LIKE=1、DISLIKE=0、SWITCH=0.4）：

```java
return switch (action) {
    case "LIKE", "UP", "ADOPT", "ACCEPT"  -> 1.0;
    case "DISLIKE", "DOWN", "REJECT"      -> 0.0;
    case "SWITCH", "CHANGE", "REFRESH"    -> 0.4;
    default -> null;
};
```

### 面试可能问的问题
1. **为什么需要离线评估？**
   → 等价于"LLM 应用的回归测试"：人看不完所有对话，需要可复现的批量度量来验证每次 prompt/规则改动。
2. **60/10/30 权重怎么来的？**
   → 规则分客观可复现是主干；用户反馈真实但稀疏给 30%；LLM Judge 本身不可靠，只做主观体验补充给 10%。
3. **gold label（人工标注）起什么作用？**
   → 提供标准答案，驱动意图/槽位/澄清准确率指标；低分样本标注后回流，形成迭代闭环。

### 可能的追问
- **LLM Judge 打分不准怎么办？**
  → 只让它打"表达质量"两维（解释质量/自然度），准确率类交给规则；权重压到 10%；失败返回 null 不参与。
- **没有标注的 Trace 怎么算分？**
  → 缺失指标返回 null，不参与平均、不当 0 分误伤（归一化加权）。
- **怎么验证系统真的变好了？**
  → 三层证据：离线指标趋势（评估报告）、在线行为（反馈点赞率、换一批比例）、Bad-case 人工抽检。
- **反馈表为什么按 session 归属而不是 traceId？**
  → 当前反馈表没有 traceId 字段，是近似归属；改进方向是给 `recommend_feedback` 加 trace_id 精确关联（诚实说出已知边界，很加分）。

---

## 面试前 30 秒速记

- 4 个设计点一句话版：
  1. **多 Agent**：拆 4 个专职 Agent，编排层管状态和路由；
  2. **检索先行**：规则选菜（JSON_OVERLAPS + 7 维打分），LLM 只写理由，防幻觉；
  3. **Trace**：ThreadLocal + 事件模型，每轮每个节点输入输出耗时 Token 落库可回放；
  4. **评估**：规则 60% + Judge 10% + 反馈 30% 加权，标注回流形成迭代闭环。

- 高频追问的安全区：**"已知边界 + 改进方案"**（单机锁、同步写 Trace、反馈无 traceId、明文密钥），主动暴露边界比被问倒强。
