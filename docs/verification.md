# 共享就绪终检记录(Task 14 Step 3)

记录日期:2026-09-21。对象:extensibility-design(7 文件)与 extensibility-audit(6 文件)两个 skill 目录(scratch 仓)。前置:两轮 GREEN 验证(Task 7/13)均通过;本记录为 REFACTOR 收尾(四项累计 Minor 修补 + 跨平台移植性终检)之后的共享就绪核对。

## 一、跨平台移植性终检(Step 2)

命令(workdir = scratch 仓 skills 目录):

```bash
grep -rinE 'opencode|superpowers|brainstorming|writing-plans|executing-plans|finishing-a-development|subagent-driven|design-implementation-consistency' extensibility-design extensibility-audit; echo "portability_exit=$?"
find extensibility-design extensibility-audit -name '*.md' | sort
```

输出(逐字):

```text
portability_exit=1
extensibility-audit/references/adversarial-lite.md
extensibility-audit/references/blast-radius-metrics.md
extensibility-audit/references/smells-catalog.md
extensibility-audit/SKILL.md
extensibility-audit/templates/audit-report.md
extensibility-audit/templates/finding-record.md
extensibility-design/references/change-taxonomy.md
extensibility-design/references/extension-patterns.md
extensibility-design/references/fitness-rules-guide.md
extensibility-design/SKILL.md
extensibility-design/templates/change-register.md
extensibility-design/templates/decision-record.md
extensibility-design/templates/design-constraints.md
```

结论:grep 零匹配(`portability_exit=1`),两 skill 目录不含任何专有工具/平台名;兄弟 skill 名相互仅以条件式提及、不构成安装前提(audit SKILL.md:21「若装有 extensibility-design 则交给它」;design SKILL.md:21、72「若装有 extensibility-audit 则…」);文件数 = **13**(design 7:SKILL.md + references/×3 + templates/×3;audit 6:SKILL.md + references/×3 + templates/×2)。**目录可直接 zip/复制到任何 agentskills.io 兼容客户端的 skills 目录。**

## 二、降级路径清单核对(5/5 显式存在)

| # | 降级场景 | 证据(file:line) | 内容 |
|---|---------|----------------|------|
| 1 | 无子代理 | `extensibility-audit/references/adversarial-lite.md:16`(另 audit SKILL.md:48) | 「无子代理能力 → 单代理复核,发现记录与报告标注"未交叉验证"」 |
| 2 | 无 git | `extensibility-audit/references/blast-radius-metrics.md:35`(另 audit SKILL.md:70) | 「无 git 环境:复制仓库到临时目录 `/tmp/...` 副本上模拟,报告标注降级;复制前先提交或清理基线脏文件,否则度量会把脏文件一并计入」 |
| 3 | 无测试框架 | `extensibility-audit/SKILL.md:70` | 降级速查「无测试框架→静态回查+只统计波及面」 |
| 4 | 无依赖规则工具 | `extensibility-design/references/fitness-rules-guide.md:56-66`(另 audit SKILL.md:70) | 「## 语言约束降级(栈无适配工具)」:AGENTS.md 语言约束模板 + 登记表标注 `门禁:语言约束(降级)` + 后哨每次运行 LLM 复查并在报告标注"约束力降级" |
| 5 | 无登记表 | `extensibility-audit/SKILL.md:39`(Phase 0;dot 图 SKILL.md:30、降级速查 SKILL.md:70 呼应) | 「无登记表 → 降级模式:自底向上从代码归纳变化点(全部标 `[推断]`),请用户确认后重建登记表再继续」 |

结论:5/5 降级路径在 skill 文档中显式存在,且均要求**降级标注**(未交叉验证 / 降级 / 约束力降级),降级运行不产生虚假置信。

## 三、最终文件清单与字数(13 个,`wc -w` 记录)

| 文件 | 字数(wc -w) |
|------|-------------|
| extensibility-design/SKILL.md | 372 |
| extensibility-design/references/change-taxonomy.md | 191 |
| extensibility-design/references/extension-patterns.md | 332 |
| extensibility-design/references/fitness-rules-guide.md | 134 |
| extensibility-design/templates/change-register.md | 116 |
| extensibility-design/templates/decision-record.md | 33 |
| extensibility-design/templates/design-constraints.md | 29 |
| extensibility-audit/SKILL.md | 301 |
| extensibility-audit/references/adversarial-lite.md | 59 |
| extensibility-audit/references/blast-radius-metrics.md | 156 |
| extensibility-audit/references/smells-catalog.md | 142 |
| extensibility-audit/templates/audit-report.md | 103 |
| extensibility-audit/templates/finding-record.md | 50 |
| **合计** | **2018**(design 1207 + audit 811) |

## 四、已接受的遗留项(设计取舍,不阻塞共享)

1. **C2/S3+D3/S1 关键词近似重叠**:分类库中 C2(同步转异步/批量)与 S3(批量处理能力)、D3(数据量级增长)与 S1(流量/并发量级增长)启发关键词近似。设计取舍:分类库定位是扫描启发而非互斥分类,由使用规则 1「宁多记后删,不凭感觉跳过维度」(change-taxonomy.md:47)缓解——同一变化点两维度命中只会多记后删,不会漏记。
2. **登记表两个子表无示例行**:change-register 模板的「新信号登记」与「变更记录」两个子表仅有表头(列语义已由表头定义,GREEN-1 实填验证通过);「规则规格」表含 RULE-01 占位行。设计取舍:示例行易与真实登记内容混淆;GREEN-1 实测代理填写正确。
3. **grep -c 口径备注**:blast-radius-metrics.md:16 已备注「grep -c 计数为 0 时退出码非零;在 set -e 下使用须对各命令追加 `|| true`」——保留为口径备注而非改写命令(命令本身语义最短正确)。

## 五、两轮 GREEN 验证结论摘要

- **GREEN-1(设计场景,有 extensibility-design;对照 RED-1 红)**:RED-1 基线命中失败(未枚举任何变化点、硬编码单渠道零扩展性讨论)。GREEN-1 6/6 断言通过:8 维度扫描产出 9 条变化点并经用户逐条过表确认(硬性确认门槛真实触发、两轮协议放行)、3 个 T2 扩展点(EP-A 策略+防腐 / EP-B 观察者 / EP-C 状态机收敛,每个 ≤1 层抽象)、CON-01..05 可执行编码约束、RULE-01..03 import-linter 门禁物化(RULE-01 精确捕获植入缺陷,TDD 式预期红);无投机抽象(唯一 `[自定义机制]` 按选择协议登记理由)。writing-skills 部署清单 6/6。详见 `docs/green-design.md`。
- **GREEN-2(检视场景,有 extensibility-audit;对照 RED-2 不红)**:RED-2 基线不红(通用代理已能定性指出耦合/OCP,故 GREEN 以更高标准衡量)。GREEN-2 6/6 断言通过,且四项能力全部超出基线:**变更模拟四指标度量**(3 个 T2 在独立 worktree 真实实施、逐 CP 度量、全部判不达标 H)、**变异抽查**(初测绿暴露通知零覆盖缺口→补写验收测试→复测红;修复后 2 处再验均红)、**完成门核对**(5 条逐项勾核,修复后 5/5)、**H 级对抗复核**(双代理输入字节级相同 4306 字节、互不可见、各自复跑门禁,F-01~F-04 全部 CONFIRMED)。修复阶段(经用户授权)TDD 实施 EP-A/B/C:lint-imports 3 kept 0 broken(RULE-01 预期红转绿)、pytest 14/14、植入缺陷全部消除。部署清单 6/6。详见 `docs/green-audit.md`。
- **复盘(Task 14 Step 1)**:重读两份 GREEN 记录,未发现评审未覆盖的新绕行/打折/合理化——用户门槛真实触发与放行、变更模拟真实实施非推演、对抗复核独立无倾向性措辞、无 T3 擅自升级设计;记录为「复盘无新漏洞(评审已覆盖)」。已发现的累计 Minor 共四项,本次收尾全部关闭:AMBIGUOUS 用户裁决后流转条款(adversarial-lite.md:12)、复核证据引用纪律(adversarial-lite.md:20)、CON-02 消歧(fitness-rules-guide.md:62)、GREEN-2 第二轮 prompt 字节标注 103→217 勘正(green-audit.md,实测 217 字节)。

## 六、共享结论

两个 skill 目录满足共享就绪:移植性零专有名依赖(grep exit=1)、13 文件结构完整、5/5 降级路径显式且带降级标注、遗留项均为已接受设计取舍、两轮 GREEN 验证分别证明「基线红→全过」与「基线不红→四能力超出」。**可直接 zip/复制到任何 agentskills.io 兼容客户端的 skills 目录。**
