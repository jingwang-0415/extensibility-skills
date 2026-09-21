# 扩展性坏味道目录(Change Preventers)

每信号含:检测方法(命令模板)+ 证据要求(`file:line` + 片段)+ 分级。

## SM-1 Shotgun Surgery(霰弹式修改)
同一业务概念/枚举散落多文件,改一处须改多处。
检测:概念名或渠道名在 ≥3 个非测试文件出现:
`grep -rn "{概念名}" --include="*.py" --include="*.java" --include="*.ts" --include="*.js" --include="*.go" src/ | grep -v test`
(逐栈多个 --include;勿用引号包裹的 `*.{py,java}` 花括号形式——GNU grep 不支持,会静默空结果)
证据:每个命中点 file:line + 命中行。分级:H(新增一个变体需改 >1 既有文件)。

## SM-2 类型分支链
业务逻辑按类型字符串/枚举做 ≥3 分支的 if/elif/switch。
检测:`grep -rn 'if.*channel\|elif.*type\|case .*:' src/`(按项目语言调整)。
证据:分支所在函数全貌。分级:登记表已声明模式 → H(模式偏离);未声明 → M,若同时命中 SM-1 则升 H。

## SM-3 Divergent Change(发散式变化)
一个类/文件因 ≥2 个不同维度被反复修改。
检测:`git log --oneline --follow {file} | wc -l` 异常高(如 >10 次或显著高于仓库中位)且提交主题跨维度;或多职责命名(XxxManager/Util 结尾承载多域)。
证据:提交列表 + 文件职责清单。分级:M。

## SM-4 Parallel Inheritance Hierarchies
新增一类需同步新增另一体系类。
检测:两棵继承树节点数 1:1 同步增长(人工比对)。
证据:两树对照表。分级:M。

## SM-5 无防腐层直连
核心业务模块直接 import 第三方 SDK,或直接 import 横切实现模块(通知/审计/日志的具体实现)而非其接口。
检测:`grep -rn "from {sdk}\|import {sdk}" src/`(SDK 名来自依赖清单;横切实现模块名按项目分层定义;路径按项目布局调整;无行首锚点,以捕获函数内延迟导入)。
证据:import 行 + 使用行。分级:H(核心域)/M(边缘模块)。

## SM-6 硬编码配置
业务逻辑内字面量阈值/URL/渠道名/魔法数。
检测:`grep -rn '"\(http\|alipay\|wechat\)"' src/` 及数值字面量抽查。
证据:命中行。分级:M(登记表 P3 相关时升级 H)。

## 使用规则
1. 逐信号扫描,宁报后核,不跳过;结论必须回落到 file:line 证据
2. 命中后关联登记表变化点:该坏味道会在哪个 CP 变化时引爆
3. 疑似但不成立的 → 记录为已排除 + 排除理由,不进入发现清单
