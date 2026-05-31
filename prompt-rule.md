# 生产级提示词（Prompt）编写指南

本指南基于 GPT-5.5、Claude Opus 4.8、Gemini 3.5 Flash 三大最新模型的官方文档及最新学术研究，确立核心编写原则。目标：用最少的指令实现最高的输出确定性。

**更新测试：这行是测试更新功能添加的。**


---

## 核心规则速查

1. **结果导向，而非过程导向**：描述目标、成功标准、约束条件，让模型自行规划路径。不要在 prompt 中编排每一步骤。
2. **正向表述优先**：告诉模型"做什么"，而非"不要做什么"。必须设禁令时用陈述式（"格式：纯文本"）而非命令式（"千万不要用 Markdown"）。适用范围：内容与风格生成。安全红线与 Agent 流程流控允许保留否定剪枝（如"严禁重试认证错误"）。
3. **结构化锚定**：用 Markdown 标题或 XML 标签隔离指令区块。Claude 对 XML 标签响应最佳。
4. **示例按需添加**：推理模型默认 zero-shot，仅在 ablation 测试证明有效时添加示例。非推理模型推荐 3-5 个多样示例。示例数据必须脱敏，使用虚构但逼真的跨领域数据。
5. **用参数控制推理深度**：不要在 prompt 中写"think step by step"。用 `reasoning.effort`（GPT-5.5）、`effort`（Claude）、`thinking_level`（Gemini）参数替代。
6. **角色定义**：在 prompt 开头用声明式（"Role: Senior Engineer"）而非表演式（"Act as..."）声明角色。
7. **输出格式契约**：机器消费场景：100% 交由 API Structured Outputs 托管，prompt 内不描述 Schema 字段。人类阅读场景：Markdown 模板放 prompt 开头。
8. **消除逻辑二义性**：消灭"充分""详细""通俗"等主观形容词，替换为可验证的限制性条件。
9. **缓存友好排版**：静态内容放开头，对话历史居中（前缀稳定支持增量缓存），RAG 文档和用户输入放末尾（每轮变化）。注意：OpenAI 自动缓存需 1024+ tokens，低于此阈值缓存不触发。超长上下文（>32k tokens）场景下准确率优先于缓存，指令可后置。
10. **上下文管理**：关键指令放开头或结尾，中间放数据。控制 prompt 长度，避免冗余信息稀释信号。
11. **异常恢复路径**：Agentic 工作流必须为工具调用失败设计显式降级路径——重试、降级、兜底、人工介入。错误文本保持静态以迎合缓存，动态追溯参数通过 API metadata 传递，严禁进入 prompt 历史。
12. **Prompt 即代码**：对 Prompt 进行版本管理，配套构建黄金测试集（Golden Set），每次修改运行回归测试。

---

## 详细原则说明

### 1. 结果导向，而非过程导向

* **核心定义**：告诉模型"要什么"（目标、成功标准、约束），而非"怎么做"（每一步骤）。三大模型厂商均明确主张 outcome-first。
* **GPT-5.5 官方**："Shorter, outcome-first prompts usually work better than process-heavy prompt stacks." "Describe the destination rather than every step."
* **Claude Opus 4.8 官方**：effort 参数控制推理深度，prompt 中的逐步脚手架应删除，用参数替代。
* **Gemini 3.5 Flash 官方**："Verbose or complex prompt engineering techniques designed for older models may cause the model to over-analyze."
* **实操落地**：将"第一步检索痛点，第二步对应成因，第三步对比方案"重构为"产出一份痛点-成因-方案对照表，覆盖所有已知场景，结论先行"。

### 2. 正向表述优先

* **核心定义**：告诉模型"要做什么"，而非"不要做什么"。否定指令存在"启动效应"——提及被禁词汇反而激活该词汇的生成概率。
* **GPT-5.5 官方**："Avoid unnecessary absolute rules. Older prompts often use strict instructions like ALWAYS, NEVER, must, and only. Use those words for true invariants, such as safety rules, required output fields, or actions that should never happen. For judgment calls, prefer decision rules instead."
* **机制解释**：学术研究（arxiv 2601.08070）通过 40,000 个样本的机械可解释性分析发现，87.5% 的否定指令失败源于"启动效应"——模型对被禁词汇的注意力强于对否定词的注意力。
* **陈述式替代命令式**：当必须设禁令时，用陈述式/声明式句式（"格式限制：纯文本"）而非命令式（"千万不要使用 Markdown"）。研究表明命令式语气在跨语言环境下语义拓扑极不稳定，而陈述式极其稳定（arxiv 2603.25015）。
* **实操落地**：将"禁止使用虚假数据"重构为"仅使用可溯源的真实数据"；将"不要啰嗦"重构为"每段不超过 3 句"。ALWAYS/NEVER 仅用于安全规则等真正不变量。
* **适用边界**：正向表述适用于**内容与风格生成**。**安全红线与 Agent 流程流控**允许保留否定剪枝——涉及外部工具链的确定性对接时，"禁止""严禁"是触发模型在 RLHF 阶段写入的最高权重防御通路的必要开关。例如"严禁重试认证错误"是合理的流程约束。

### 3. 结构化锚定

* **核心定义**：利用大模型对特定标记的结构敏感度，拉起注意力的物理隔离墙。
* **Claude Opus 4.8 官方**："XML tags help Claude parse complex prompts unambiguously, especially when your prompt mixes instructions, context, examples, and variable inputs." Claude 被专门训练识别 XML 标签作为结构标记。
* **GPT-5.5 官方**：推荐 Markdown 标题 + XML 标签组合使用。
* **上下文位置**：研究表明（Liu et al., 2024）开头和结尾的注意力最强，中间区域准确率下降 30%+。关键指令放开头，关键数据放结尾。

### 4. 示例按需添加

* **核心定义**：示例是工具，不是默认配置。是否使用、使用多少，取决于模型类型和任务类型。
* **推理模型（GPT-5.5 Thinking、Claude Extended Thinking、Gemini Thinking）**：
  * 默认 zero-shot。OpenAI 官方："Try zero shot first, then few shot if needed. Reasoning models often don't need few-shot examples to produce good results."
  * 过多样本可能有害——推理模型会花推理预算模仿示例结构，而非解决实际问题。独立实测显示 zero-shot 在数学和编码任务上一致优于 few-shot。
  * 若需示例，one-shot 对推理模型的数学/编码任务可能最优（OpenReview 2026），但需 ablation 验证。
* **非推理模型（GPT-4o、Claude Sonnet、Gemini Flash 等）**：
  * Claude 官方推荐 3-5 个示例，用 `<example>` 标签包裹，确保覆盖多样输入类型。
  * GPT-5.5 官方："Few-shot learning lets you steer a large language model toward a new task... try to show a diverse range of possible inputs with the desired outputs."
  * Gemini 实践："One good example beats five paragraphs of instructions."
* **通用原则**：示例的多样性和输入分布比标签正确性更重要（Min et al., 2022）。超过 5 个后收益递减，上下文成本持续攀升。每个示例长度不超过任务输入的 10%。
* **具象脱敏原则**：示例中的数据、人名、实体必须与真实业务隔离。优先采用"业务完全不相关、结构高度相似"的跨领域真实数据（如金融审查任务使用烹饪菜谱步骤展示结构），既断绝数据泄漏，又避免符号污染。若必须使用虚构数据，使用逼真的虚构人名（如"李明"、"王芳"）而非机械占位符——模型可能将 `[USER_NAME_A]` 作为字面量打印出来。研究表明 few-shot 示例中的真实数据会通过"具象记忆泄漏"进入模型输出（tianpan.co, 2026），在多租户场景下构成跨租户数据泄漏风险。
* **决策规则**：任务可用规则描述 → 跳过示例。任务需要展示风格/品味 → 加 1-2 个脱敏示例。始终用 ablation 测试验证。
* **机器消费场景的注入规范**：当使用 Structured Outputs（GPT-5.5）或 tool-use（Claude）时，Few-Shot 示例严禁以明文 JSON 写入 System Prompt 文本——这是对 Schema 的变相描述，会与 API 的输出约束产生协议解析冲突。正确做法：通过 `messages` 数组注入 mock message pairs（模拟 `user` 消息 → `assistant` 的结构化响应对象），确保 Prompt 文本层面的绝对纯净。

### 5. 用参数控制推理深度

* **核心定义**：不要在 prompt 中编排推理过程，用 API 参数控制模型的思考深度。
* **三大模型的参数**：
  * GPT-5.5：`reasoning.effort`（none / low / medium / high / xhigh），默认 medium
  * Claude Opus 4.8：`effort`（low / medium / high / xhigh / max），默认 high
  * Gemini 3.5 Flash：`thinking_level`（minimal / low / medium / high），默认 medium
* **GPT-5.5 官方**："A reasoning model is like a senior co-worker. You can give them a goal to achieve and trust them to work out the details."
* **Gemini 3.5 Flash 官方**："Simplify prompts. If you used chain-of-thought prompt engineering to force reasoning, try thinking_level: medium or high with simpler prompts instead."
* **Claude Opus 4.8 实践**："Delete the scaffolding and raise the effort level instead... You pay for the reasoning you select, not for a paragraph of instructions begging the model to try harder."
* **何时仍需显式推理引导**：对需要可审查推理过程的场景（如合规、审计），可在 Claude 中启用 thinking tags 要求可见推理。但控制深度用参数，不用 prompt 文字。

### 6. 角色定义

* **核心定义**：在 prompt 开头用声明式句式声明角色，校准模型的专业深度、用词习惯与表达风格。三大模型均推荐。
* **声明式 vs 表演式**："Role: Senior Software Engineer"（声明式）优于 "You are a senior software engineer who..."（表演式）。声明式是事实描述，表演式是行为指令，在跨语言环境下稳定性不同（arxiv 2603.25015）。
* **GPT-5.5 官方**：推荐 prompt 结构以 Role 开头，区分 Personality（听起来怎样）和 Collaboration Style（怎样工作）。
* **注意事项**：角色定义对风格类任务（写作、总结）效果显著，对分类、事实问答等任务效果有限。不要在角色中强制规定自信程度——Claude Opus 4.8 的诚实校准机制会与强制自信指令冲突，导致模型忽略角色指令或产生虚假自信。

### 7. 输出格式契约

* **核心定义**：明确指定输出的结构、字段、类型、顺序，消除模型"自由发挥"的空间。
* **机器消费场景**：100% 交由 API 结构化输出能力托管。GPT-5.5 的 Structured Outputs、Claude 的 tool-use 强制输出、Gemini 的 `responseSchema` 均在编译器层面约束输出格式。Prompt 内禁止描述 Schema 字段——既浪费 token，又存在 API 升级时 prompt 与代码同步失调的风险。
* **GPT-5.5 官方**："Remove output schema definitions from the prompt where possible. Use Structured Outputs instead."
* **人类阅读场景**：用 Markdown 模板描述期望格式，放 prompt 开头。明确字数、段落数、是否使用项目符号。模型对前置格式指令的遵从度更高。
* **Gemini 3.5 Flash 实践**："Specify the output format up front. 'Return JSON with keys X, Y, Z' before the actual task, not after."

### 8. 消除逻辑二义性

* **核心定义**：坚决消灭主观、模糊的描述性形容词（如"充分"、"详细"、"通俗"），进行算法化重构。
* **Claude Opus 4.8 官方**：模型"interprets prompts literally and explicitly"。模糊指令不会被泛化，只会被按字面理解或忽略。
* **Gemini 3.5 Flash 实践**："Be concise... Verbose or complex prompt engineering techniques designed for older models may cause the model to over-analyze."
* **实操落地**：将"充分展开细节"重构为"沿'核心观点 → 支撑论据 → 核心细节'的逻辑链条正序向下解码"。将"Be concise"重构为"Max 80 words"。

### 9. 缓存友好排版

* **核心定义**：Prompt 的物理排版顺序直接决定 50%-90% 的调用成本与响应延迟。三大厂商均支持 prompt caching，缓存匹配基于前缀精确匹配。
* **排版规范**：按稳定性从高到低排列：
  1. 系统级全局指令（完全静态）
  2. 工具定义 / API Schema（完全静态）
  3. 静态示例（Few-shot，完全静态）
  4. 对话历史（半静态递增——前缀稳定，支持增量缓存）
  5. RAG 检索文档 / 动态参考上下文（每轮变化，必须后置）
  6. 当前用户输入（完全动态）
* **RAG 后置原则**：在多轮对话中，RAG 检索文档每轮都可能变化。若将其放在对话历史之前，任何文档变更都会破坏后续所有内容的前缀缓存，导致对话历史的增量缓存完全失效。将 RAG 文档放在对话历史之后、当前用户输入之前，可确保对话历史的前缀稳定，最大化增量缓存命中率。
* **进一步优化**：将 RAG 文档区分为"长期稳定参考文档"（如公司政策、产品手册，变化频率低）和"每轮动态检索片段"（随查询变化）。前者可放入缓存友好的静态区域，后者必须后置。
* **GPT-5.5**：缓存自动生效（1024+ tokens），前缀匹配。24 小时持久缓存为默认策略。静态内容放开头，动态内容放末尾。
* **Claude**：用 `cache_control` 显式标记断点（最多 4 个），支持 5 分钟 / 1 小时两种 TTL。严格处理层级：Tools → System → Messages。
* **Gemini**：支持隐式缓存（自动）和显式缓存对象（自定义 TTL）。
* **常见反模式**：在 system prompt 中嵌入时间戳、请求 ID、用户姓名等动态内容——这会使每个请求的前缀不同，完全破坏缓存。
* **阈值判定**：OpenAI 自动缓存需静态前缀 ≥ 1024 tokens。若系统指令 + 工具定义 + 静态示例总和低于此阈值，缓存不会触发，此时无需为缓存做排版妥协，可直接采用注意力优先排版。
* **容量分流**：缓存优先排版与注意力优先排版在超长上下文中存在互斥：
  * 上下文 < 32k tokens：强制执行缓存优先排版（静态内容绝对前置），成本优先。
  * 上下文 > 32k tokens 且使用 Gemini：考虑指令后置（放在数据之后、用户输入之前），准确率优先。Gemini 官方建议长上下文场景"place specific questions at the end, after your data context"。
  * 上下文 > 32k tokens 且使用 GPT-5.5 / Claude：缓存优先排版仍适用，但需确保系统指令不超过上下文的前 10-15%。
* **来源**：OpenAI Prompt Caching 文档、Anthropic 官方 GitHub caching 指南、AI Workflow Lab 2026 缓存指南。

### 10. 上下文管理

* **核心定义**：将 prompt 视为有限资源进行预算管理。
* **位置策略**：开头和结尾是黄金注意力区域，中间最弱。Gemini 官方建议长上下文场景将指令放在数据之后。
* **长度控制**：研究表明 prompt 超过 150-300 词后，附加内容通常稀释信号而非增强（Levy, Jacoby, Goldberg, 2024）。GPT-5.5 官方明确反对 legacy prompt stacks 的过度指定。
* **指令解释**：告诉模型"为什么"比告诉它"不要"更有效。例如"不要使用省略号，因为输出将由 TTS 引擎朗读"，模型可从解释中泛化出更多隐含规则。

### 11. 异常恢复路径

* **核心定义**：在 Agentic 工作流中，大模型频繁调用外部工具。生产级 Prompt 必须对工具报错、返回空值或网络超时设计显式的降级和纠错路径。
* **核心模式**：
  * **重试**：瞬态错误（5xx、超时、429）等待后重试，限制次数
  * **降级**：主工具失败后切换备用工具
  * **兜底**：所有工具失败时输出结构化错误码，交由人工介入
  * **快速失败**：认证错误（401/403）、配额耗尽等永久性错误不重试，直接报告
* **AWS Well-Architected 框架**："Develop agents with error handling in mind. Agents should classify responses as actionable or not, and take appropriate action to retry or gracefully fail."
* **反模式**：未配置恢复路径的 Prompt 会导致模型在工具故障时产生死循环或幻觉性填补。
* **实操落地示例**："若工具 A 调用返回超时或空值，等待 3 秒后重试一次；仍失败则切换至工具 B；若仍报错，停止自动规划，输出 ERROR_CODE 及原因，交由人工介入。"
* **异常可追溯原则**：模型在触发兜底或快速失败路径时，必须输出可追溯的错误信息，但**追溯元数据与上下文文本必须分离**：
  * **文本域**（进入对话上下文，保持静态以迎合缓存）：仅输出静态错误码与原因，如 `"STATUS: RECOVERY_FALLBACK_TRIGGERED. REASON: TOOL_TIMEOUT."`
  * **元数据域**（通过 API metadata 或日志传递，严禁进入 prompt 历史）：携带动态追溯参数，如 `trace_id`、`failed_tool_name`、`error_stage`、`input_payload_digest`
  * **陷阱警告**：动态哈希值（如 `input_payload_digest`）若进入对话历史，会因每轮变化而破坏后续所有增量缓存，导致高并发环境下的 token 成本雪崩。

### 12. Prompt 即代码

* **核心定义**：生产环境下的 Prompt 是高频调用的代码资产，应严禁依靠"个人体感"或"单次调试"进行修改。
* **版本管理**：每次 Prompt 改动作为 Git Commit 提交，附带变更说明和预期效果。
* **黄金测试集（Golden Set）**：为每个 Prompt 配套构建代表性输入 + 预期输出的测试集。每次修改后在测试集上运行回归测试，确保无性能退化。
* **评测指标**：正确性、完整性、格式合规率、安全合规率、延迟、成本。
* **GPT-5.5 官方**："Build evals that measure the behavior of your prompts so you can monitor prompt performance as you iterate, or when you change and upgrade model versions."
* **实操落地**：即使只有 20 个测试样本的黄金集，也远好过一周的"体感调试"。

---

## 参考来源

### 一、模型厂商官方文档

| # | 来源 | 日期 | 链接 |
|---|------|------|------|
| 1 | OpenAI: GPT-5.5 Prompting Guide | 2026-04-23 | https://developers.openai.com/api/docs/guides/prompt-guidance |
| 2 | OpenAI: Using GPT-5.5 | 2026-04-23 | https://developers.openai.com/api/docs/guides/latest-model |
| 3 | OpenAI: Prompt Engineering | 持续更新 | https://developers.openai.com/api/docs/guides/prompt-engineering |
| 4 | OpenAI: Reasoning Models | 持续更新 | https://developers.openai.com/api/docs/guides/reasoning |
| 5 | OpenAI: Reasoning Best Practices | 持续更新 | https://developers.openai.com/api/docs/guides/reasoning-best-practices |
| 6 | OpenAI: Prompt Caching | 持续更新 | https://developers.openai.com/api/docs/guides/prompt-caching |
| 7 | Anthropic: Claude Prompting Best Practices | 持续更新 | https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices |
| 8 | Anthropic: Introducing Claude Opus 4.8 | 2026-05-28 | https://www.anthropic.com/news/claude-opus-4-8 |
| 9 | Anthropic: Prompt Caching Guide (GitHub) | 持续更新 | https://github.com/anthropics/skills/blob/main/skills/claude-api/shared/prompt-caching.md |
| 10 | Google: Gemini 3.5 Flash What's New | 2026-05-19 | https://ai.google.dev/gemini-api/docs/whats-new-gemini-3.5 |

### 二、学术研究与行业分析

| # | 论点 | 来源 | 链接 |
|---|------|------|------|
| 11 | 否定指令的启动效应机制：87.5%失败源于提及被禁词汇反而激活它 | arxiv 2601.08070, 2026 | https://arxiv.org/abs/2601.08070 |
| 12 | 命令式语气跨语言不稳，陈述式更稳定 | arxiv 2603.25015, 2026 | https://arxiv.org/abs/2603.25015 |
| 13 | Lost in the Middle 效应：上下文中间区域准确率下降 30%+ | Liu et al., 2024 | 被多篇 2026 年文章引用 |
| 14 | Few-shot 示例的输入分布比标签正确性更重要 | Min et al., 2022 | 被 Thomas Wiegold 2026 文中引用 |
| 15 | prompt 长度甜点区 150-300 词 | Levy, Jacoby, Goldberg, 2024 | 被 Thomas Wiegold 2026 文中引用 |
| 16 | 推理模型存在结构性过度思考（>60%冗余） | arxiv 2605.23926, 2026 | https://arxiv.org/abs/2605.23926 |
| 17 | CoT 的收益与成本理论分析：错误累积是结构性问题 | arxiv 2605.21260, 2026 | https://arxiv.org/abs/2605.21260 |
| 18 | 推理模型 zero-shot 优于 few-shot（数学/编码任务） | OpenReview, 2026 | https://openreview.net/pdf?id=5FtNTyaHp0 |
| 19 | Agentic 工作流的异常恢复模式与最佳实践 | AWS Well-Architected GenAI Lens | https://docs.aws.amazon.com/wellarchitected/latest/generative-ai-lens/genrel03-bp01.html |
| 20 | PALADIN: Agent 工具失败恢复框架，89.7%恢复率 | OpenReview, 2026 | https://openreview.net/forum?id=NVTtoO297p |
| 21 | Prompt Caching 生产实践：三方对比与排版优化 | AI Workflow Lab, 2026-04-20 | https://aiworkflowlab.dev/article/prompt-caching-production-claude-openai-gemini-cost-reduction |
| 22 | Few-shot 示例的数据泄漏风险与跨租户安全 | tianpan.co, 2026-05-13 | https://tianpan.co/blog/2026-05-13-tenancy-leaks-few-shot-examples-cross-customer-prompt-store |
| 23 | RAG 上下文排版对缓存命中率的影响 | FlowVerify, 2026-05-22 | https://www.flowverify.co/blog/prompt-caching-production-hit-rate-prompt-structure |
| 24 | 多轮对话中的增量缓存优化与 RAG 文档排版 | DEV Community, 2026-05-27 | https://dev.to/synthorai/llm-prompt-caching-the-complete-2026-guide-3mmb |

