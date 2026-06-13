# TencentDB Agent Memory —— 架构分析与赛题映射

> 项目：TencentDB-Agent-Memory v0.3.4 (MIT License)  
> 作者：腾讯 DB 团队  
> 仓库：https://github.com/Tencent/TencentDB-Agent-Memory  
> 分析时间：2026年5月16日

---

## 一、项目概览

```
一句话：腾讯开源的 Agent 记忆系统，四层语义金字塔 + 符号化压缩，
已经在 Hermes 和 OpenClaw 上验证过，Token 节省 61%，PersonaMem 准确率 48%→76%。
```

### 核心指标（官方Benchmark）

| 记忆能力 | Benchmark | 原版 | 加插件后 | 提升 |
|----------|-----------|------|---------|------|
| 短期记忆 | WideSearch | 33% | **50%** | +51.52% |
| 短期记忆 | SWE-bench | 58.4% | **64.2%** | +9.93% |
| 短期记忆 | AA-LCR | 44.0% | **47.5%** | +7.95% |
| 长期记忆 | PersonaMem | 48% | **76%** | +59% |
| Token消耗 | WideSearch | 221M | **85M** | **-61.38%** |
| Token消耗 | SWE-bench | 3474M | **2375M** | **-33.09%** |

---

## 二、核心架构：四层语义金字塔

```
L3 Persona (用户画像)
  ↑ 每50条新记忆触发一次生成
L2 Scenario (场景块)
  ↑ 每15分钟/跨场景触发聚合
L1 Atom (原子事实)
  ↑ 每5轮对话触发提取
L0 Conversation (原始对话)
  ↑ 每轮对话自动捕获
```

### 每层详解

**L0 — Conversation（原始对话存储）**
- 完整保留 user↔assistant 的原始对话
- 存为 JSONL + 可选 SQLite/TCVDB
- 支持全文本检索（BM25 + 向量）

**L1 — Atom（结构化事实提取）**
- 每 N 轮对话触发一次 LLM 提取
- 从对话中抽取出离散的原子事实
- 向量去重——相同事实不重复存储
- 示例：从"我习惯用 Markdown" → `{type: preference, fact: "用户偏好Markdown格式"}`

**L2 — Scenario（场景聚合）**
- 将相关 L1 Atoms 聚合成场景块
- 存在 Markdown 文件中，人类可读
- 示例：将"Markdown偏好"+"下午高效"+"安全敏感"→ 工作场景画像

**L3 — Persona（用户画像）**
- 所有 L2 Scenarios 的顶层总结
- 输出为 `persona.md`，包含：
  - 用户核心偏好
  - 工作习惯
  - 沟通风格
  - 长期目标

### 下钻追溯链（审计/调试）

```
Persona（高层）→ Scenario（中层索引）→ Atom → Conversation（原文）
```

任何高层抽象都能追溯到原始对话证据——不是黑盒。

---

## 三、短期记忆：符号化压缩（Mermaid Canvas）

除了长期记忆的四层金字塔，还有短期（任务内）记忆的符号化压缩：

```
工具调用日志（几十万Token）
  ↓ L1.1: 写入 refs/*.md（完整保留）
  ↓ L1: 提取步骤摘要 → JSONL
  ↓ L1.5 + L2: 生成 Mermaid 任务画布
  ↓ 注入 Agent 上下文（仅几百Token）
```

Agent 只看轻量级 Mermaid 图，需要查证时通过 `node_id` 下钻到 `refs/*.md`。

---

## 四、存储与检索架构

### 存储后端（零配置可用）

```
默认：SQLite + sqlite-vec（本地，零配置）
可选：Tencent Cloud VectorDB（云端）
```

### 检索策略

```
hybrid = BM25（关键词）+ 向量（语义）+ RRF 融合
支持：keyword / embedding / hybrid 三种模式
BM25 分词：支持中文（jieba）和英文
```

### 检索路径

```
Agent 收到新消息
  → POST /recall (query="用户最新消息")
  → L3 Persona 注入（顶层偏好）
  → L1/L2 语义检索（相关知识）
  → 结果注入 Agent 上下文
  → Agent 回复时比无记忆多出关键偏好/知识
```

---

## 五、与 Hermes 的集成方式

### 架构图

```
Hermes Agent (Python)
  └─ MemoryManager
       └─ MemoryTencentdbProvider        ← hermes-plugin 目录
            ├─ GatewaySupervisor          ← 启动/监控 Node.js 侧车
            └─ MemoryTencentdbSdkClient   ← HTTP 客户端
                    │
                    ▼  HTTP (127.0.0.1:8420)
            memory-tencentdb Gateway (Node.js)
               └─ TdaiCore (核心引擎)
                    ├─ L0 Conversation store
                    ├─ L1 Episodic extraction
                    ├─ L2 Scene blocks
                    └─ L3 Persona synthesis
```

### 生命周期映射

| Hermes 事件 | Gateway API | 行为 |
|------------|-------------|------|
| 每次对话前 | POST /recall | 同步检索记忆，返回上下文注入 |
| 每轮对话后 | POST /capture | 异步捕获对话，后台线程处理 |
| 会话结束 | POST /session/end | 刷新待处理流水线 |
| Agent 工具 | memory_search / conversation_search | LLM 主动搜索记忆 |

### 环境变量（最少配置）

```bash
# 必需：LLM API Key（用于 L1/L2/L3 提取）
export MEMORY_TENCENTDB_LLM_API_KEY="sk-..."

# 可选：覆盖默认模型
export MEMORY_TENCENTDB_LLM_BASE_URL="https://api.openai.com/v1"
export MEMORY_TENCENTDB_LLM_MODEL="gpt-4o"
```

---

## 六、与赛题7项要求的逐条映射

| 赛题要求 | TencentDB 对应能力 | 适配程度 |
|----------|-------------------|---------|
| **(1) 多源数据整合** | L0 自动捕获所有对话+工具调用+配置。Capture hooks 覆盖 after-tool-call、before-prompt-build 等。 | ★★★★★ 天然支持 |
| **(2) 偏好动态捕捉** | L1→L2→L3 自动提取偏好。L1 从对话抽原子事实→L2 聚合场景→L3 生成用户画像。Pipeline 定时触发（可配）。 | ★★★★★ 核心功能 |
| **(3) 知识结构化整合** | L2 Scenario 块 + 向量去重（L1 dedup）。BM25+向量混合检索。Mermaid 符号化知识图谱。 | ★★★★☆ 知识图谱不是主存储，但 Mermaid Canvas 是图形化知识表示 |
| **(4) 端侧部署≤500ms** | SQLite+sqlite-vec 纯本地存储，零网络依赖。HNSW 索引+BM25 混合检索。Token 压缩降低 LLM 调用量。 | ★★★★☆ 本地检索快，但 LLM 提取（L1/L2/L3）需要 API 调用；可换本地小模型降本 |
| **(5) 安全与遗忘** | 未内置。需自建：PII 过滤层 + L0/L1 精准删除接口 + NL→删除指令解析 | ★★☆☆☆ 需自研扩展 |
| **(6) 记忆流转** | L0（短期）→ L1（中期桥）→ L2/L3（长期）天然分层。层间有明确索引和追溯。 | ★★★★★ 天然三层 |
| **(7) 量化评测** | 已有 Benchmark 框架（PersonaMem, SWE-bench）。自带的 `tdai_memory_search` 支持评测查询。 | ★★★★☆ 需扩展为赛题要求的评测标准 |

---

## 七、需要你们团队自研的部分

这个项目覆盖了赛题 70% 的需求，剩下 30% 就是你们的**创新空间**：

### 必须做的（赛题硬要求）

| 需求 | 实现思路 |
|------|---------|
| **麒麟 embedding SDK 替换** | 把 Gateway 的 embedding 模块（`src/core/store/embedding.ts`）的 OpenAI 接口换成麒麟 SDK 适配器 |
| **敏感信息识别+过滤** | 在 L0 capture 阶段加 PII 过滤层（可用 Presidio/正则） |
| **NL 精准遗忘** | 加一个 `/forget` 指令解析 → L0/L1 精确删除 API |
| **麒麟 OS 适配测试** | 在银河麒麟桌面环境跑通全栈，测延迟、资源占用 |

### 可以做来加分的（创新点）

| 方向 | 思路 | 对标论文 |
|------|------|---------|
| **自进化记忆** | 让检索策略随用户使用自我调优 | SAGE, EvolveMem |
| **双模态冲突消解** | 偏好冲突+知识冲突统一框架 | Rashomon Memory, Entropic Claim |
| **推理感知检索** | 重排序引入推理能力 | MemReranker |
| **Mermaid 知识图谱→麒麟OS原生UI** | 把 Mermaid Canvas 渲染成麒麟OS原生图形界面 | 自研 |

---

## 八、技术风险与对策

| 风险 | 对策 |
|------|------|
| LLM API 调用成本/Latency | L1/L2/L3 用本地 7B 模型（Qwen2.5-7B Q4 量化），取代云端 LLM |
| 麒麟 SDK 能力未知 | 保留 BM25 兜底检索，麒麟 SDK 只做语义补充 |
| SQLite 性能在大数据量下未知 | 基准测试 + HNSW 索引优化 + 数据分区 |
| Node.js Gateway 在麒麟OS上运行未知 | 提前做麒麟桌面OS兼容性测试 |
| 团队 TypeScript/Node.js 能力 | Gateway 代码量不大（~50个源文件），Python Hermes Plugin 更少（~3个文件），可集中学习 |

---

## 九、建议的开发路线

```
阶段1（理解期，1-2周）
  ├─ 通读 TdaiCore 源码（src/core/tdai-core.ts）
  ├─ 本地跑通 Hermes + Gateway
  └─ 理解 L0→L1→L2→L3 Pipeline

阶段2（适配期，2-3周）
  ├─ 替换 Embedding 为麒麟 SDK 适配器
  ├─ 添加敏感信息过滤层
  └─ 添加 NL 遗忘指令

阶段3（优化期，2-3周）
  ├─ 麒麟OS部署测试
  ├─ 延迟优化（目标 ≤500ms）
  ├─ 评测数据集构建
  └─ 性能指标验证

阶段4（打磨期，2-3周）
  ├─ 文档编写
  ├─ 演示视频
  └─ 测试报告
```

---

## 十、总结

**TencentDB Agent Memory 是目前能找到的最契合这个赛题的开源方案**：

- ✅ MIT 协议，商用/比赛无限制
- ✅ 已与 Hermes 集成（你们可以直接用）
- ✅ 四层记忆金字塔天然覆盖赛题核心需求
- ✅ 有 Benchmark 数据支撑技术报告
- ✅ 本地 SQLite 存储适合端侧部署
- ✅ 腾讯团队维护，社区活跃（Discord + 微信群）
- ⚠️ 需要你们自研 30%：麒麟SDK适配 + 安全遗忘 + 评测框架

**跟之前搜的那25篇SOTA论文的关系**：这个项目是**工程实现**，那些论文提供**创新方向**。你们的差异化在于——基于这个成熟的工程底座，把论文中的最新思想（自进化、双模态冲突消解、推理感知检索）加进去。
