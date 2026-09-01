---
description: 为存量代码补全/完善日志（对已完成但日志不完善的项目做日志补全）。触发词：补日志/完善日志/日志补全/存量日志/日志不完善。用法：/gen-logs <代码路径或模块名>。核心：按 java-code-standards 日志规范补全——全类 @Slf4j、删 System.out、入口入参+耗时、关键节点 INFO、异常 ERROR 带堆栈；只加日志不改业务逻辑。
---

# 存量日志补全

为已有代码补全或完善日志。核心原则：**日志是"证据"，不是"装饰"——补的日志要能回答"这个请求发生了什么、为什么失败"**；只加日志，禁止改业务逻辑。

> 完整执行规则与日志规范见 **java-code-standards** skill（`00-common/04-logging-standards.md`），本命令为流程入口。

## 输入

- 目标代码路径（文件/目录/模块）
- 技术方案 md（存在时自动走 `--from-spec`，否则 `--from-code`）

## 两种模式

### `--from-spec`（有技术方案，优先）

代码有对应 3.x 技术方案 md → 从 spec 派生日志点：

1. 读技术方案 md（入口定义/核心逻辑/关注点/验收场景）
2. 按方案标注的关键业务节点确定日志点：状态变更、下单/支付/取消、消费完成、定时任务开始/结束
3. 从方案抄业务上下文 ID（订单号/用户 ID）与关键字段
4. 匹配不上 spec 的代码 → 对该部分降级为 `--from-code` 处理

### `--from-code`（无 spec，按日志规范补全）

无技术方案 → 按 java-code-standards `00-common/04-logging-standards.md` 的强制规则与最佳实践逐类补全：

| 日志点 | 规则 | 来源 |
|---|---|---|
| 类声明 | 全类 `@Slf4j`（Lombok），禁止散落 `private static final Logger` 混用 | 04-logging §1 |
| System.out | 一律删除，替换为 log 输出 | 04-logging §1 |
| 请求入口（Controller） | 记入参摘要 + 耗时，出口记结果 | 04-logging §最佳实践 |
| 关键业务节点 | 状态变更/下单/支付/取消/消费处理完成/定时任务开始结束 → INFO | 04-logging §2 |
| 异常处 | ERROR 且带堆栈（`log.error("msg, orderId={}", orderId, e)`，异常对象作最后参数）；catch 块禁止空吞 | 04-logging §4 |
| 日志内容 | 占位符 `{}` 禁止字符串拼接；关键日志带业务上下文 ID；不记敏感信息（密码/token/手机号脱敏）；大对象只记关键字段 | 04-logging §3/§5 |

## 执行规则（硬性约束，不可协商）

```
1. 遵守 java-code-standards `00-common/04-logging-standards.md` 全部强制规则
2. 只增删日志，禁止修改业务逻辑（改一行业务代码即违规）
3. 已有合格日志不覆盖，只补缺失或不合格的（如 System.out、无堆栈的 error）
4. 关键业务节点必须有 INFO；异常处必须有 ERROR 带堆栈——该记的地方必须记
5. 拿不准该不该记日志的逻辑 → 按"能否回答请求发生了什么/为什么失败"判断；仍拿不准 → 询问，不自行发挥
6. 每个文件完成后独立提交（commit message = 补日志: 文件路径）
7. 完成后对照 04-logging-standards 自检清单逐项核对
```

## 输出

- 日志补全后的代码
- 日志核对结果（按 04-logging-standards 自检清单：@Slf4j/无 System.out/占位符/异常带堆栈/上下文 ID/无敏感信息/关键节点覆盖）

## 完成标准

- [ ] 全类 @Slf4j（Controller/Service/Job/Listener 一律带），无散落 Logger 声明、无 System.out
- [ ] 请求入口（Controller）有入参摘要 + 耗时日志
- [ ] 关键业务节点（状态变更/下单/消费完成/定时任务）有 INFO
- [ ] 异常处有 ERROR 且带堆栈（无空 catch、无只记 message 丢堆栈）
- [ ] 占位符替代字符串拼接；关键日志带业务上下文 ID；无敏感信息
- [ ] 未修改任何业务逻辑；已有合格日志未被破坏
- [ ] 每个文件独立提交
