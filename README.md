# Extensibility Skills

> Agent Skills that move "hard to extend later" problems **before** code generation — for any Agent-Skills-compatible AI coding agent (Claude Code, OpenCode, Cursor, Codex, Copilot, Gemini CLI, Goose, Amp, and 40+ others).

两个前后哨 skill,把过去依赖资深开发者经验才能发现的"未来不好扩展"问题,前移到代码生成之前解决:

```
需求方案
  ↓
【前哨 extensibility-design】变化点分析 → 扩展性设计(YAGNI 裁决) → 可执行门禁落仓
  ↓
AI 编码(依赖规则测试已在仓库,编码期即被约束)
  ↓
【后哨 extensibility-audit】扩展性检视(双轨) → 黑盒验证(功能验收 + 变更模拟波及面度量) → 报告修复
  ↓
合并 / PR
```

核心机制:**扩展性的客观度量 = 实施一个预判变更的波及面**(新增文件数 / 是否修改既有文件 / 既有测试是否被破坏)。经验判断变成可验证指标。

## 它解决什么问题

AI 生成的代码普遍不考虑未来扩展性:接口写死、分支散落、缺少隔离层,问题在评审甚至上线后才暴露。本项目的做法:

- **编码前**:结构化枚举未来变化点(8 维度 × 24 信号分类库),对每条做显式裁决——T1 立即实现 / T2 预留扩展点(从 11 个预设模式中按抽象成本最低选用)/ T3 仅记录(防过度设计)
- **编码中**:扩展性约束以**可执行门禁**(import-linter / ArchUnit / dependency-cruiser 依赖规则测试)持久化进仓库——skill 缺席时依然生效
- **编码后**:双轨检视(确定性规则 + LLM 语义回查/坏味道扫描)+ H 级发现双代理对抗复核 + **变更模拟**(在独立 worktree 真实实施预判变更、度量四指标波及面、默认回滚)+ 变异抽查防"永远绿"假测试

## 两个 Skill

| Skill | 触发时机 | 产物 |
|-------|---------|------|
| `extensibility-design` | 需求/设计文档就绪、编码之前 | 变化点登记表(CP-NN)、决策记录(ADR-NN)、编码约束(CON-NN)、依赖规则测试(RULE-NN)、AGENTS.md 约束段 |
| `extensibility-audit` | 实现完成、合并/PR 之前 | 审计报告(F-NN 发现清单 + 证据链 + 对抗复核)、变更模拟四指标、变异抽查记录、登记表状态更新 |

两个 skill 可独立安装;未装前哨时后哨以降级模式运行(自底向上归纳变化点,标注 `[推断]` 并请用户确认)。

## 安装

把 `extensibility-design/` 与 `extensibility-audit/` 复制到你所用 AI 编码助手的 skills 目录:

- OpenCode: `~/.config/opencode/skills/`
- Claude Code: `~/.claude/skills/`(全局)或项目内 `.claude/skills/`
- 其它兼容客户端参见其文档或 [agentskills.io/clients](https://agentskills.io/clients)

## 依赖与降级(全部可选)

| 能力 | 无该能力时 |
|------|-----------|
| 子代理 | H 级发现单代理复核,报告标注"未交叉验证" |
| git | 变更模拟在临时目录副本上做 |
| 测试框架 | 静态检视 + 变更模拟只统计波及面 |
| 依赖规则工具 | AGENTS.md 自然语言约束 + 每次运行 LLM 复查 |

## 设计依据与验证

- 业界实践基础:Change Cases (Scott Ambler)、Evolutionary Architecture fitness functions (Ford/Parsons)、ADR (Nygard)、Change Preventers 坏味道 (Fowler)、ATAM 场景化架构评估 (CMU SEI)、Architecture Drift Reduction with LLMs (Thoughtworks Radar 2026)。详见 [docs/spec.md](docs/spec.md)
- 端到端验证:无 skill 基线(设计零变化点讨论,RED 红)→ 装前哨后全过(GREEN);植入的 shotgun surgery 被后哨全部检出且经双代理对抗 CONFIRMED;变更模拟实测"新增支付渠道需改 2 个既有文件"判超标;变异抽查咬合。详见 [docs/verification.md](docs/verification.md)

## 仓库结构

```
├── extensibility-design/     # 前哨(SKILL.md + 3 references + 3 templates)
├── extensibility-audit/      # 后哨(SKILL.md + 3 references + 2 templates)
└── docs/                     # 设计 spec + 验证证据
```

## License

[MIT](LICENSE)
