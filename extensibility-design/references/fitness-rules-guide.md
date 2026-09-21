# 依赖规则门禁指南

目标:把扩展性约束转成**可执行测试**,编码期即被约束。规则语义来自 T2 的 CON-NN 约束(如"payment 禁止依赖 notification")。

## Python — import-linter

`pyproject.toml`(或 `.importlinter`):

```toml
[tool.importlinter]
root_package = "shop"

[[tool.importlinter.contracts]]
name = "CON-01 payment-not-notify"
type = "forbidden"
source_modules = ["shop.payment"]
forbidden_modules = ["shop.notification"]
```

运行:`lint-imports`(并入 CI/测试命令)。分层约束用 `type = "layers"`。

## Java — ArchUnit

```java
@AnalyzeClasses(packages = "com.shop")
class ExtensibilityRulesTest {
    @ArchTest
    static final ArchRule con01_payment_not_notify = noClasses()
        .that().resideInAPackage("..payment..")
        .should().dependOnClassesThat()
        .resideInAPackage("..notification..");
}
```

运行:`mvn test`(规则测试放 `src/test/java/.../arch/`)。

## JS/TS — dependency-cruiser

`.dependency-cruiser.cjs`:

```javascript
module.exports = {
  forbidden: [
    { name: "con01-payment-not-notify", severity: "error",
      from: { path: "^src/payment" }, to: { path: "^src/notification" } },
  ],
};
```

运行:`npx dependency-cruiser src`。

## 规则规格格式(目录未定时的降级落盘)

写入登记表"规则规格"区:RULE-NN | 关联 CON | 语义(源模块 禁止/只允许 依赖 目标)| 路径占位 | 状态(未物化)。后哨物化为上述任一形态。

## 语言约束降级(栈无适配工具)

AGENTS.md 追加:

```markdown
## 扩展性约束(来源: .extensibility/change-register.md)
- CON-02: 新增支付渠道只允许新增 shop/payments/*Provider 实现,禁止修改 PaymentService。
  依据: CP-01/ADR-01。修改此约束须先更新登记表并经用户确认。
```

登记表对应条目标注 `门禁:语言约束(降级)`;后哨每次运行需 LLM 复查并在报告标注"约束力降级"。
