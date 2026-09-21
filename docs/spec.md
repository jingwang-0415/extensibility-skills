# 扩展性前后哨 Skills 设计文档

- 日期:2026-09-20
- 状态:已确认(设计经用户分段确认)
- 产物:两个 opencode skills(`extensibility-design` / `extensibility-audit`),遵循 agentskills.io 开放标准,可跨平台共享

## 1. 背景与问题

AI 生成的代码普遍不考虑未来扩展性:接口写死、分支散落、缺少隔离层。这类"未来不好扩展"的问题传统上依赖资深开发人员的经验,在代码评审时才被发现,修复成本高。本设计通过两个前后哨 skill,把该问题**前移到代码生成之前解决**:

```
需求方案 (brainstorming 产出 spec)
   ↓
【前哨】extensibility-design —— 变化点分析 → 扩展性设计(YAGNI 裁决) → 门禁落仓
   ↓
writing-plans → AI编码 (TDD; 依赖规则测试已在仓, 编码期即被约束)
   ↓
【后哨】extensibility-audit —— 扩展性检视 → 黑盒验证(功能+变更模拟) → 报告修复
   ↓
finishing-a-development-branch (merge/PR)
```

核心机制:**扩展性的客观度量 = 实施一个预判变更的波及面**(新增文件数、是否修改既有代码、既有测试是否被破坏)。把经验性判断转化为可验证指标。

## 2. 目标与非目标

### 目标

1. 在编码前结构化枚举未来变化点,并对每个变化点做出显式设计裁决(实现 / 预留扩展点 / 明确不设计)
2. 扩展性约束以可执行门禁(依赖规则测试)形式持久化到仓库,skill 缺席时依然生效
3. 编码后以双轨(确定性工具 + LLM 语义)检视扩展性,并以变更模拟实测波及面
4. 遵循 agentskills.io 开放标准,可直接共享到 Claude Code / Cursor / Codex / Copilot 等 40+ 兼容客户端
5. 通用多栈,运行时校准;所有降级路径显式标注
6. 最终交付两个发行包:独立版(仅两个新 skill,零 superpowers 依赖)与捆绑版(附 superpowers 整套),见 §8

### 非目标

- 不替代 `design-implementation-consistency`(代码 vs 设计文档一致性);后哨聚焦"代码 vs 未来变化"
- 不做性能/安全等非扩展性架构属性评估
- 不追求预测所有未来变化(承认概率×影响评估的主观性,靠用户确认环节缓解)
- 不修改任何现有 superpowers skills

## 3. 业界实践基础

| 设计元素 | 业界出处 | 拓展点 |
|---------|---------|--------|
| 变化点三元组(变化+可能性+影响) | Change Cases (Scott Ambler, Agile Modeling) | 8 维度×信号结构化分类库 + Layer1/2 演进模型 |
| T1 直接实现 | Change Cases:"极高概率变化写成正式需求" | 无,照搬 |
| T3 仅记录 | Speculative Generality 反模式 (Fowler);YAGNI (XP) | 无,照搬 |
| T2 预留扩展点+成本上限 | 进化架构"最后责任时刻"原则 (Ford/Parsons) | "每点 ≤ 一层抽象 + 文档"为工程化操作化,业界无量化标准,需实践校准 |
| 扩展机制选择 | OCP (Martin)、防腐层 (Evans DDD)、ports&adapters (Cockburn)、GoF 封装变化 | 无,经典 |
| 轻量决策记录 | ADR (Nygard/MADR) | 简化为单条目模板 |
| 依赖规则测试落仓 | Fitness Functions (Ford/Parsons);TW Radar 2026 "Architecture drift reduction with LLMs"(ArchUnit 等 + LLM 双轨) | 设计期而非编码后生成("架构约束先行");落仓后脱离 skill 独立生效 |
| 检视信号 | Change Preventers 坏味道 (Fowler/refactoring.guru):Shotgun Surgery / Divergent Change / Parallel Inheritance Hierarchies | 无,照搬 |
| 变更模拟 | SEI ATAM/SAAM 场景化架构评估 (CMU) | **核心拓展**:从纸面场景推演变为真实实施 + 波及面度量 |
| skill 格式 | agentskills.io 开放标准(渐进披露) | 无,遵循 |
| 变异抽查防假测试 | Mutation Testing 实践(TW Radar Trial) | 简化为 1-2 处人工变异抽查 |
| 预设模式目录 | GoF 目录思想;TW Radar "Mapping code smells to refactoring techniques"(Trial):问题→预定义解法映射提升 agent 一致性 | 11 个模式映射 8 维度 + 抽象成本标注 + 选择协议(成本最低优先 + `[自定义机制]` 逃生舱 + Layer 2 演进) |

## 4. 前哨 `extensibility-design` 详细设计

### 4.1 触发条件

frontmatter description 覆盖:需求/设计文档已存在、即将进入编码计划(writing-plans 之前)、新功能设计、变化点分析、扩展性设计、预留扩展点等场景(中英文关键词)。

### 4.2 工作流

**Phase 0 输入与校准**

- 定位需求/设计文档:任何来源——用户提供的文档、仓库内 docs(常见如 docs/specs/、docs/design/、`*-design.md`)、或用户口述要点;skill 内容不得写死特定环境路径(本环境常见输入为 brainstorming 产物 `docs/superpowers/specs/*-design.md`,仅作说明,不进 skill 文档)
- 识别技术栈与目录结构(为 Phase C 门禁生成做准备)
- 无 spec 时引导用户先完成需求设计,不凭空臆测需求

**Phase A 变化点分析(三层模型)**

- **Layer 0 通用分类库**(skill 自带,8 维度约 24 信号,每信号附启发问题,源自 Change Cases 七问):
  1. 业务规则与流程:定价/费率调整、审批流变化、状态机新增状态、规则参数化、新业务对象类型
  2. 集成与依赖:第三方服务替换/新增、外部协议变化、新数据源接入、消息中间件更换
  3. 数据演进:schema 变更、存储引擎更换、缓存策略变化、数据量级增长
  4. 接口与契约:API 版本升级、新客户端类型/多端、开放 API 给第三方、同步转异步
  5. 规模与性能:流量增长、并发提升、时延要求收紧、批量处理能力
  6. 部署与配置:多环境差异、多租户/SaaS 化、私有化部署、feature flag 增减
  7. 用户与场景:国际化/多语言、新用户角色、权限模型变化、新交互形态
  8. 技术栈与合规:框架大版本升级、运行时更换、法规(数据留存/审计)、安全要求
- **Layer 1 项目登记表**(落仓 `<repo>/.extensibility/change-register.md`,活文档):
  每条变化点 = `ID | 变化描述 | 维度 | 可能性 H/M/L | 影响面 H/M/L | 依据(需求原文/领域常识/[推断]) | 裁决(T1/T2/T3) | 状态`
- **Layer 2 自扩展**:项目中遇到分类库未覆盖的变化点 → 写入项目登记表;领域无关 → 提升回 Layer 0(skill 维护者操作)
- 产出后**一次性过表请用户增删确认**(多选问题交互)

**Phase B 扩展性设计与 YAGNI 裁决**

按 `可能性 × 影响面` 矩阵分级(先看影响面,再看可能性):

| 可能性 | 影响面 | 裁决 | 动作 |
|-------|-------|------|------|
| H | H | **T1 立即实现** | 直接进需求/设计,不留扩展机制 |
| M | H | **T2 预留扩展点** | 明确扩展机制;轻量 ADR;编码约束 |
| H 或 M | M | **T2 预留扩展点** | 同上 |
| L | H 或 M | **T3 仅记录** | 登记表一句话理由,零设计投入 |
| 任意(含 H) | L | **T3 仅记录** | 同上 |

矩阵规则:**自上而下首个命中行生效**。即影响面为 L 一律 T3;影响面为 H 且可能性为 H 才升 T1,可能性为 M 即 T2;影响面为 M 且可能性 ≥ M 即 T2;可能性为 L 为 T3。

**T2 扩展机制必须从预设模式目录中选用(选择协议):**

预设模式目录(references/extension-patterns.md,11 个模式,每个附语言中立描述 + 适用维度 + 抽象成本 + 反例 + 代码骨架):

| 模式 | 适用变化维度 | 抽象成本 |
|------|------------|---------|
| 策略 (Strategy) | 新渠道/算法/规则(支付、计费、排序) | 1 接口 |
| 插件注册表 (Plugin Registry) | 同类能力可扩展注册,配置/第三方驱动 | 1 注册点 |
| 防腐层 (ACL) | 第三方依赖替换、外部协议变化 | 1 边界层 |
| 适配器 (Adapter) | 多端/接口适配 | 1 包装层 |
| 配置外置 | 参数变化、环境差异 | 0 |
| 特性开关 (Feature Flag) | 灰度、部署形态切换 | 0 |
| 模板方法 (Template Method) | 流程骨架稳定、步骤可变(审批流) | 1 基类 |
| 事件/观察者 | 新增旁路行为不改主流程(通知/审计) | 1 事件契约 |
| 密封接口+工厂 | 新业务对象类型(sum types) | 1 接口 |
| 数据版本迁移 | schema 演进 | 1 版本机制 |
| 管道/中间件 | 处理链新增环节(校验/过滤链) | 1 链契约 |

选择协议(防"模式强迫症"):
1. 每个 T2 点先查目录的**维度→候选模式**映射(通常 2-3 个候选),**抽象成本最低者优先**(与 T2 成本上限"≤一层抽象"对齐)
2. 目录不合身 → 允许自由设计,登记表标注 `[自定义机制]` + 理由;后哨对自定义机制**单独复核**
3. 自定义机制多次复用 → 沉淀进项目模式清单(Layer 1);领域通用 → 提升回预设目录(Layer 2,skill 维护者操作)——与变化点分类库同构演进

其余 Phase B 规则:
- T2 成本上限:**每个扩展点 ≤ 一层抽象 + 文档**,超出即过度设计
- 每个 T2 以轻量 ADR 记录到 `design-decisions.md`:`决策(含选用模式) | 理由 | 代价(抽象成本) | 迁移触发条件(当X发生时如何演进)`
- 产出**编码约束段**(可执行表述,如"新增支付渠道只允许新增 `PaymentProvider` 实现,禁止修改 `PaymentService`"),写入 `design-constraints.md` 供 writing-plans 与 AI 编码引用

**Phase C 门禁落仓**

- 目录结构已定:按栈生成依赖规则测试(Java→ArchUnit / Python→import-linter / JS/TS→dependency-cruiser),并入测试命令
- 目录结构未定:以结构化规则规格写入登记表(语义完整、路径占位),由后哨在代码成形后物化为可执行测试
- 无工具适配的栈:降级为 AGENTS.md 自然语言约束段,登记表标注 `门禁:语言约束(降级)`
- 写入 `.extensibility/` 产物;AGENTS.md 增补扩展性约束段(引用登记表,不重复内容)

### 4.3 前哨产出物

| 产物 | 位置 |
|------|------|
| 变化点登记表(含裁决) | `<repo>/.extensibility/change-register.md` |
| 决策记录(轻量 ADR) | `<repo>/.extensibility/design-decisions.md` |
| 编码约束段 | `<repo>/.extensibility/design-constraints.md` |
| 依赖规则测试(或规格) | 各栈惯例位置 / 登记表内 |
| AGENTS.md 扩展性约束段 | `<repo>/AGENTS.md` |

## 5. 后哨 `extensibility-audit` 详细设计

### 5.1 触发条件

frontmatter description 覆盖:实现完成待合并前、扩展性检视、扩展性评审、不好扩展、变更模拟、波及面度量等场景。

### 5.2 工作流

**Phase 0 校准**

- 识别栈/测试框架/测试命令
- 定位 `.extensibility/` 登记表
- 登记表缺失 → **降级模式**:自底向上从代码归纳变化点(全部标注 `[推断]`),请用户确认后重建登记表再继续

**Phase A 扩展性检视(双轨)**

- **轨 1 确定性**:若登记表存在未物化的规则规格(前哨 Phase C 目录未定时落下的),先物化为可执行测试;然后运行全部落仓依赖规则测试,违规即发现(证据:测试输出)
- **轨 2 LLM 语义**:
  - 登记表回查:每个 T2 扩展点在代码中真实存在、位置正确(证据 `file:line`),且**采用了登记表 ADR 声明的预设模式**(声明策略模式却写成 if-else 链 = H 级发现);`[自定义机制]` 项单独复核(机制是否成立、抽象是否失控)
  - 坏味道扫描(Change Preventers):shotgun surgery(同一业务概念/枚举散落多文件)、类型 switch/if-else 分支链、divergent change、外部依赖无防腐层直连、硬编码配置
- 发现分级:
  - **H**:T2 扩展点缺失/名存实亡;shotgun surgery 实锤;依赖规则违规
  - **M**:坏味道嫌疑(局部耦合);T3 变化点出现未登记的扩散修改迹象
  - **L**:文档/命名/轻微重复
- **H 级双代理对抗复核**:两个独立子代理盲验同一发现(给定相同的登记表条目+代码位置输入),结论一致才确认;不一致 → 标记 `AMBIGUOUS` 请用户裁决。M/L 单代理附证据链即可
- 无子代理平台:H 级降级为单代理复核,报告标注"未交叉验证"

**Phase B 黑盒验证**

- **B1 功能验收**:确认 T1 变化点对应功能有验收测试覆盖;**变异抽查**防"永远绿"假测试——对核心逻辑做 1-2 处变异,确认测试变红后还原;测试缺失则补写
- **B2 变更模拟(波及面度量)**:
  - 抽样:`可能性×影响面` 加权排序取 top 2~3 个 T2 变化点(用户可增删;大项目可降至 1)
  - 在独立 git worktree 上真实实施该变更
  - 度量四个指标:`新增文件数 | 修改既有文件数 | 修改行数 | 既有测试破坏数`
  - 判定:T2 点变更应"只新增、不修改"(OCP 目标);任何"修改既有文件"必须落在该点 ADR 声明的迁移触发条件内;超出 = H 级发现
  - **默认回滚** worktree;用户可选保留为真实演进
- **B3 门禁固化**:修复后重跑全部依赖规则 + 验收测试;登记表各条目更新状态(`已验证 YYYY-MM-DD`)

**Phase C 报告与修复**

- 修复 H/M 发现(改代码为主)
- 若发现**登记表裁决本身有误**(变化点漏判/误判):标记请示用户——登记表是活文档,经确认后演进(区别于 design-implementation-consistency 的"设计不可改"约定)
- 产出 `.extensibility/audit-reports/audit-NN.md`:发现清单(等级/证据链/裁决)+ 变更模拟数据 + 修复记录 + 降级标注汇总

**完成门(全部满足)**:

1. 登记表 100% 回查(T2 全部有代码证据,T1 全部有验收测试)
2. 依赖规则测试全绿(降级模式:LLM 检查通过并标注)
3. 验收测试全绿
4. 抽样变更模拟全部达标
5. H 级发现全部修复;无未处置的 AMBIGUOUS

## 6. Skill 目录结构(agentskills.io 标准)

```
~/.config/opencode/skills/extensibility-design/
├── SKILL.md                       # 主流程(中文)
├── references/
│   ├── change-taxonomy.md         # Layer 0 分类库(8 维度 24 信号+启发问题)
│   ├── extension-patterns.md      # 预设模式目录(11 模式×维度映射+成本+反例+骨架)与选择协议
│   └── fitness-rules-guide.md     # 各栈依赖规则编写指南
└── templates/
    ├── change-register.md
    ├── decision-record.md         # 轻量 ADR
    └── design-constraints.md

~/.config/opencode/skills/extensibility-audit/
├── SKILL.md                       # 主流程(中文)
├── references/
│   ├── smells-catalog.md          # Change Preventers 检测信号(检测什么)
│   ├── blast-radius-metrics.md    # 波及面度量定义与判定阈值(怎么验证)
│   └── adversarial-lite.md        # H 级双代理复核协议(怎么复核)
└── templates/
    ├── audit-report.md
    └── finding-record.md
```

## 7. 降级与错误处理

| 场景 | 降级策略 | 标注 |
|------|---------|------|
| 无测试框架/不可执行 | 静态回查 + 变更模拟只统计波及面不跑测试 | 报告标注"降级" |
| 无 git/worktree | 变更模拟在临时目录副本上做 | 报告标注 |
| 栈无依赖规则工具 | AGENTS.md 自然语言约束 + 每次运行 LLM 检查 | "约束力降级" |
| 大项目变更模拟成本高 | 抽样数降至 1,或用户指定单点 | 报告标注 |
| 登记表缺失 | 自底向上归纳变化点,全标 `[推断]`,用户确认后重建 | "降级模式" |
| 无子代理能力 | H 级发现单代理复核 | "未交叉验证" |
| 中断续跑 | 登记表/报告条目带状态字段,支持断点恢复 | - |

## 8. 依赖分层与跨平台策略

并非所有目标用户都装有 superpowers——skill 不得强依赖任何环境专属组件。

### 依赖分层

- **Tier 0 自带**:`references/` 与 `templates/` 随 skill 目录分发,永为可用
- **Tier 1 探测式衔接**:兄弟 skill(extensibility-design/audit)与外部流程(需求设计、实施计划、分支收尾)**仅条件式提及**("若可用则衔接");两个 skill 各自可独立安装、独立运行
- **Tier 2 能力降级**:子代理 / git / 测试框架 / 依赖规则工具按 §7 降级表处理
- **禁止**:skill 文件内出现 superpowers 内部 skill 名(brainstorming / writing-plans / executing-plans / finishing-a-development-branch / subagent-driven-development 等)、平台专有工具名、本地专属 skill 名(design-implementation-consistency)、写死的特定环境路径

### 与本环境现有流程的关系(仅说明,不进 skill 文档)

- 前哨在 brainstorming 之后、writing-plans 之前;后哨在实现完成、finishing-a-development-branch 之前
- `design-implementation-consistency`:管"代码 vs 设计文档",与本设计互补;其对抗验证思想被后哨的 H 级复核简化继承

### 发行包(最终交付)

两个 skill 为**单源内容**,两包共享同一份文件,仅打包范围不同:

| 发行包 | 内容 |
|--------|------|
| 独立版 `extensibility-skills-standalone.zip` | extensibility-design/ + extensibility-audit/ + README(安装与降级说明) |
| 捆绑版 `extensibility-skills-with-superpowers.zip` | 独立版全部内容 + superpowers/ 整目录原样(含 LICENSE)+ README(完整链路说明) |

## 9. 验证方式(skill 自身验收)

用示例小项目(订单 + 支付模块)端到端演练:

1. 前哨跑出登记表(应包含"新增支付渠道"等典型变化点,且为其从预设目录选用**策略模式**而非自由发明机制)+ 依赖规则测试落仓
2. 人工编码,**故意植入一个 shotgun surgery**(如把渠道判断 if-else 散落两处)
3. 后哨必须:发现该 H 级发现(含"声明策略模式却写成 if-else 链"的模式偏离检查);变更模拟(新增一个支付渠道)度量出"修改既有文件数 ≥ 1"并判为超标;变异抽查生效
4. 执行 writing-skills 的校验清单(含 SKILL.md 格式、渐进披露、无内部依赖)
5. **跨平台终检**:对两个 skill 全部文件执行 grep,模式 `opencode|superpowers|brainstorming|writing-plans|executing-plans|finishing-a-development|subagent-driven|design-implementation-consistency` 零命中(兄弟 skill 名 extensibility-* 仅允许条件式提及)
6. **发行包验证**:两个 zip 解压后 frontmatter 可解析、引用文件齐全、可独立运行

## 10. 风险与已知局限

| 风险 | 缓解 |
|------|------|
| T2 成本上限"≤一层抽象"为操作化猜测,或紧或松 | 实践中按 ADR 记录偏差,Layer 2 机制回收校准 |
| 变更模拟成本高 | 抽样策略 + 默认回滚 + 大项目降为 1 个 |
| 概率×影响评估主观 | 用户过表确认 + `[推断]` 标注区分实证与推断 |
| 语言约束降级时约束力下降 | 显式标注"约束力降级",不假装等价 |
| 变化点枚举永远不全 | 定位为"前移"而非"穷尽";登记表为活文档持续演进 |
