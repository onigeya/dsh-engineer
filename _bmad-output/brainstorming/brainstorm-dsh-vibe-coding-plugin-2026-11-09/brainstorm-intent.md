# Intent: DSH Vibe Coding 插件 —— "仪表盘与刹车片"

一句话定位：给 DeepSeek Harness 补上它唯一缺失的验证反馈层（V 层）——声明式反馈传感器 + 证据门 + 方法论薄包装——而不是再造一套 harness。

## 命题改写

用户原始选题是「一套运行在 DSH 上的 Vibe Coding 插件：Tools + Skills + Hooks 稳定推动工作流 + 自动注入系统提示词与任务上下文」。对 DSH 发行版 307 个 @deepseek-ai 包逐包源码考古后，命题从「造一套 Vibe Coding harness」改写为「给现有 harness 装仪表盘（反馈传感器）和刹车片（证据门），外面包一层方法论」。

支撑这个改写的三条证据：

1. **四要件中三件已被原生解决**：注入面（systemPrompt.section order+variable+refreshContext、persona 行、AGENTS.md 链去重+预算+发现）、编排面（goal+round-driver、ralph、workflow JS 编排、plan-mode、subagent 多后端、worktree、schedule）、上下文面（compaction、spill=pass-by-reference、repeat-tool-reminder、skill 渐进披露）在 DSH 源码中都已完整存在（memlog 第 13、19 行）。
2. **唯一缺口在 V 层**：Awesome-Agent-Harness 给 DSH 的 V=◐（半满）评级与本地源码逐包核实一致；原生仅有 auto-review + observation-policy，无 guides/sensors 体系（memlog 第 14 行）。全 170+ 项目调研无一做了传感器体系（NUF 终选 N=9 的依据，memlog 第 24 行）。
3. **执行期拦截的证据强于提示词**：GABench 显示 system prompt 对强模型几乎无效（+0.3~+0.4pp），而执行期干预 +2.8~+7.7pp（memlog 第 17 行）——说明插件的真价值应投在 hooks 的"验证反馈"半边，而非提示词工程。三轮独立证据（Claude Code ultracode / DSH repeat-tool-reminder / superagent guard）共同指向「advisory 优先，blocking 要证据背书」（memlog 第 25 行）。

## 选定方向（In）

### 组件 1：Feedback Sensor Registry（反馈传感器注册表）

- **是什么**：声明式传感器注册——每个传感器声明「命令 + 适用 glob + 触发时机 + computational/inferential 类型 + shift-left 档位」。computational 型跑确定命令（linter/test/结构检查），inferential 型跑 LLM 判读；分类与三域（Maintainability / Architectural Fitness / Behaviour）取自 harness engineering 学科框架（Boeckeler, martinfowler.com, 2026-04）。
- **挂在哪**：DSH 原生 `tools/post-execute` 扩展点（memlog 第 14、24 行）。
- **对标谁**：Harness Engineering 的 Sensors 半边（前馈 Guides / 反馈 Sensors 二分法）；反向对标是市面无一做传感器体系这一空白（N=9）。
- **MVP 判据**：注册一个 lint/test 传感器后，工具执行失败信息能自动回流进 agent 上下文，无需提示词引导；附带 LLM 优化错误消息（正面提示注入，memlog 第 19 行）。
- **为什么**：V 层是 DSH 唯一净增量所在，且 GABench 证明执行期拦截的收益（+2.8~+7.7pp）远大于提示词（+0.3~+0.4pp）。

### 组件 2：Evidence Gate（证据门）

- **是什么**：挂在 `agent/turn-stopping` 上的门——agent 声明「ready/修好了」必须附可核查证据（命令输出、测试通过、diff 事实），否则拦截并要求补齐。等人字段（等待人工确认的槽位）只接受结构化数据，**永不从自由文本推断**。
- **挂在哪**：DSH 原生 turn-stopping hook 面（memlog 第 24 行）。
- **对标谁**：dsh-superagent-prototype 的 guard 机制（off/observe/auto 三模式）；UTBoost 的教训——verifier 是除模型、harness 外的第三项，且误判可覆盖 SWE-bench Lite 40.9% 条目，所以门必须保守（证据缺失才拦，不猜测）。
- **MVP 判据**：装上后「模型说修好了」必须过证据门才能结束回合（memlog 第 24 行明示）。
- **为什么**：这是用户原始四要件中 Hooks「稳定推动工作流」的验证半边，也是全插件的行为核心。

### 组件 3：阶段机薄包装（Clarify → Plan → Build → Verify → Learn）

- **是什么**：五阶段方法论包装，**寄生在 dsh-goal 状态上，不造新状态机**；accept mode 默认 observe（advisory），不是 auto。Learn 阶段沉淀 lessons（对应 memlog 第 15 轮 1 的 E 段想法）。
- **挂在哪**：复用 goal + goal-round-driver + plan-mode + subagent + worktree 既有原生面（memlog 第 24 行「②阶段机包装（次做，复用goal/plan-mode/worktree）」）。
- **对标谁**：BMAD 阶段循环、Spec Kit 收敛环、Superpowers 三层 bootstrap——但只取其方法论外壳。
- **MVP 判据**：阶段推进不新增任何持久化状态；关掉包装后 DSH 原有 goal 流程完全不受影响。
- **为什么**：planning 对弱模型 +11.6pp、对强模型省约 30% 成本（arXiv:2609.20804）——值得做，但危险组合已警示：做重 = 造 BMAD 轮子，做薄 = 寄生 goal 状态即可（memlog 第 25 行）。

### 安全默认值四条（随组件 1-3 顺手交付）

1. **技能文本进数据区渲染**，不进指令区——promptinject-agent-skills 证明技能即注入面。
2. **凭据外置**——技能生态审计显示安装技能 ≈ 给 agent sudo，9% 技能带 L3 关键风险。
3. **fail-closed**——传感器/门异常时默认拦截而非放行；执行期拦截的 +2.8~+7.7pp 收益不能建立在误放上。
4. **judge 异源**——inferential 传感器的判读模型与执行模型不同源，避免 verifier 误判放大（UTBoost：误判可覆盖 SWE-bench Lite 40.9% 条目）。

## 明确不做（Out）

1. **不自建 prompt 装配/编译器**——DSH 原生 systemPrompt.section（order+variable+refreshContext+agent 作用域覆盖）已是完整注入面；「提示词编译工件」簇在收敛步 2 被判定「并入①②」，不单独成波（memlog 第 23 行）。且 GABench 证据表明对强模型提示词收益近零。
2. **不自建编排运行时**——goal/ralph/workflow/subagent/plan-mode 原生俱全（memlog 第 13、19 行）；纯编排类骨架在轮 5 源码证据下已降级为薄包装（memlog 第 22 行）。
3. **不做技能 mega-collection**——技能生态审计：46.3% 重复、500 行/5k token 红线；技能数 202 时选择准确率 -21%，候选 >30 时准确率仅 13.6% vs <30 时 >90%（arXiv:2602.08004 等）。堆技能数量是负收益。
4. **阶段机不做重**——见上；独立状态机会与原生 goal 冲突，重造 BMAD 轮子无差异化。
5. **inferential 传感器不做单点 blocking**——LLM 判读有误判率（UTBoost 40.9% 教训），inferential 型默认 advisory；blocking 权限只给 computational 型（确定命令输出）或有证据背面的场景，与「advisory 优先，blocking 要证据背书」的跨轮暗线一致（memlog 第 25 行）。

## 第二差异化（Parking lot，高价值待验证）

**调用率仪表盘 + harness coverage 度量**：统计每个传感器在真实会话里的触发率与覆盖率，回答一个别人没问的问题——「某 sensor 从不触发，是代码质量好，还是探测不足？」（memlog 第 19 行 harness coverage、第 25 行 wildcard，编号 79 + 121）。

佐证：arXiv:2609.20804 的实证里 recall_event 在 56.3% 的设置中从未被调用——说明「装了不等于用了」是行业普遍盲区；Stop Comparing LLM Agents Without Disclosing the Harness（arXiv:2605.23950）则给了披露卡先例，可把 coverage 度量做成对标其披露格式的插件自带报告。

状态：第三波（高影响高成本），本期只留 API 钩子（memlog 第 23 行⑦簇落位），不进 MVP。

## 关键证据附录

供 PRD 的 constraints / assumptions 直接引用。全部数字带出处。

| # | 证据 | 出处 |
|---|---|---|
| E1 | harness 效应 27.4pp ≈ 模型效应 29.4pp：换 harness 与换模型同量级 | arXiv:2605.23950《Stop Comparing LLM Agents Without Disclosing the Harness》 |
| E2 | 同模型换 adapter：19.1% → 73.4% | 同上 |
| E3 | system prompt 对强模型几乎无效（+0.3~+0.4pp）；执行期拦截 +2.8~+7.7pp | GABench, arXiv:2607.20982 |
| E4 | context management 价值随窗口增长坍缩：32k 窗差 35.7pp → 128k 窗差 2.7pp；T4（先截断后摘要）性价比最优 | arXiv:2609.20804 harness 组件实证 |
| E5 | recall_event 在 56.3% 的实验设置中从未被调用（组件装了≠有用） | arXiv:2609.20804 |
| E6 | planning 对弱模型 +11.6pp；对强模型准确率 -2pp 但省约 30% 成本 | arXiv:2609.20804 |
| E7 | leaderboard 分差 <3pp 不构成发现：MVP 判据必须绕开此量级 | 网搜轮（对标评测方法学，memlog 轮 3/4） |
| E8 | verifier 误判可覆盖 SWE-bench Lite 40.9% 条目：verifier 是模型、harness 之外的第三项 | UTBoost（memlog 第 18 行） |
| E9 | 技能生态：46.3% 重复；9% 带 L3 关键风险；安装技能 ≈ 给 agent sudo；SKILL.md 500 行 / 5k token 红线 | skills 生态审计 arXiv:2602.08004（memlog 第 17 行） |
| E10 | 技能选择准确率：候选 >30 时 13.6%，<30 时 >90%；技能数 202 时 -21% | 同上 |
| E11 | Guides（前馈）× Sensors（反馈）× Computational/Inferential × 三域（Maintainability / Architectural Fitness / Behaviour）分类框架；harnessability 与 harness templates 概念 | martinfowler.com harness engineering, Boeckeler 2026-04 |
| E12 | ETCLOVG 七层 survey：DSH 为四个 anchor 系统之一；Awesome-Agent-Harness 给 DSH V 层 ◐ 评级，与 307 包源码核实一致 | Agent Harness Engineering Survey + 本地源码考古 |

## 对标融合矩阵

| 来源 | 吸收的具体机制 | 落到哪个组件 |
|---|---|---|
| dsh-superagent-prototype（官方 MIT） | off/observe/auto 三模式、guard、9 步 prompt 装配末位 policy、单写者 Engine | 组件 3 accept mode 默认 observe；组件 2 门的形态 |
| GitHub Spec Kit 1.3 | 收敛环（spec→plan→tasks 循环对齐） | 组件 3 Verify/Learn 段 |
| obra Superpowers 7 | 三层 bootstrap、技能作为方法论载体 | 组件 3 阶段命名；Out 3 反面对标（不做 mega-collection） |
| BMAD-METHOD V7 | 阶段循环 + agent 角色 | 组件 3 只取外壳；Out 4 明确不重造 |
| oh-my-openagent 5 | Kibitzer/顾问位 | 组件 1 inferential advisory 形态 |
| OpenClaw | 10 阶段 hook 提案、subagent announce/steer | 组件 1 触发时机词表 |
| Claude Code dynamic workflows | plan 进代码、ultracode 关键词触发、mods 进程内 JS hook | 组件 3 交付形态（Code-as-Workflow 簇并入②）；暗线「advisory 优先」证据 |
| martinfowler.com harness engineering（Boeckeler 2026-04） | Guides/Sensors 二分 × Computational/Inferential × 三域 × harnessability | 组件 1 声明模型字段 |
| ETCLOVG survey + Awesome-Agent-Harness | 七层缺口分析、V=◐ 评级 | 命题改写的定位依据 |
| arXiv:2609.20804 | 组件实证（E4-E6）、90 行最小 harness | 附录证据 + Out 1/2 的成本纪律 |
| arXiv:2605.23950 | harness 披露卡 | Parking lot 报告格式先例 |
| GABench arXiv:2607.20982 | 执行期干预 > 提示词（E3） | 组件 1/2 的存在理由 |
| skills 生态审计 arXiv:2602.08004 | 重复率/风险/规模阈值（E9-E10） | 安全默认值 1/2 + Out 3 |
| UTBoost | verifier 是第三项、误判覆盖 40.9%（E8） | 组件 2 保守设计 + 安全默认值 3/4 |
| promptinject-agent-skills | 技能即注入面 | 安全默认值 1 |

## 下游建议

1. **首选 bmad-product-brief**：概念已收敛（NUF 终选 N9/U10/F9）但未压测——brief 阶段正好验证「V 层传感器体系」的单一价值主张能否对外讲通。
2. **若跳过 brief 则直接 bmad-spec**：In/Out 边界与 MVP 判据已足够成形，spec 可从组件 1/2/3 + 安全默认值四条直接展开。
3. **两者都必须携带本文件作为输入**：证据附录 E1-E12 供 PRD constraints/assumptions 引用；「明确不做」五条应原样进入 spec 的 Out of Scope，防止下游技能把命题漂回「再造 harness」。

## 会话出处

- 信息源 memlog：`D:\workspace\dsh-engineer\_bmad-output\brainstorming\brainstorm-dsh-vibe-coding-plugin-2026-11-09\.memlog.md`（135 条想法 + 6 条 insight/decision，status: complete, mode: autonomous）
- 研究方式（5 轮）：轮 1 QMD 种子六对标（idea 1-35）/ 轮 2 网络检索一：superagent-prototype、Claude Code、OpenClaw（idea 36-70）/ 轮 3-4 学科文献：harness engineering、ETCLOVG、arXiv 系（idea 71-105）/ 轮 5 DSH 发行版 307 个 @deepseek-ai 包源码考古（idea 106-135）/ 收敛步 1-3：亲和聚类 7 簇 → 影响×成本落位三波 → NUF 终选主骨架
- 检索通道：QMD 本地知识库（621 文档）+ OpenSERP 网络多轮检索 + 本地源码直读
- 日期：2026-11-09
- 参与者：用户（选题与 Ideate-for-me 模式指定）、头脑风暴 coach（qmd 检索、轮 1 对标、NUF 终选主持）、文档生成代理（本文件）
