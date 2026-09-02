---
description: 关键规范自动核对（第⑤步编码收尾兜底闸门 + 验收复核）。触发词：规范核对/自动核对/check-standards/兜底核对/规范检查。用法：/check-standards <项目路径>。作用：加载 check-standards skill，用 grep/ast-grep 逐项扫描生成代码，核对关键规范是否执行到位，输出证据报告；未到位项提示用户确认是否补齐。
---

# 关键规范自动核对（Check Standards）

编码收尾 / 验收复核的**兜底闸门**：规范条目多、分散在多个 skill，AI 编码可能漏执行。本命令加载 **check-standards** skill 用 grep/ast-grep 实际扫描产出，逐项核对关键规范，**禁止凭记忆答 ✅**。

> [!IMPORTANT] 本命令已独立为 **check-standards** skill
> 完整检查清单（12 项 HIGH + 代码规范组 C1-C6 + 注释组 N1-N4 + INFO + 场景化 + 标准 grep/ast-grep 指令 + 判定标准 + 用户确认闸门）见 **check-standards** skill 的 SKILL.md。
> 执行本命令时：**加载 check-standards skill**，按其中检查项清单逐项执行。

## 执行步骤

1. 加载 **check-standards** skill
2. 按 skill「模式判定与核对依据」先判模式（标准 / 存量适配，读 2.1 约束 / 0.5 扫描）
3. 逐项执行标准检查指令（grep/ast-grep），每项附证据（文件:行号）
4. 未执行到位项（含 INFO，不分重要与否）统一向用户确认是否补齐，确认后补齐并重跑
5. 输出《关键规范核对报告》，落盘 `docs/<模块名>V<版本号>-<YYYYMMDDHHMMSS>/5.2-<功能名>-规范核对报告.md`，汇总进 5.3 验收报告

## 完成标准

- [ ] 已加载 check-standards skill 并按其清单逐项核对（12 HIGH + C1-C6 + N1-N4 + INFO + 场景化）
- [ ] 每项附证据（文件:行号）；未到位项已向用户确认后补齐 / 标注"需人工核对"（不静默吞掉）
- [ ] 报告已落盘 `docs/<模块名>V<版本号>-<YYYYMMDDHHMMSS>/5.2-<功能名>-规范核对报告.md`，并汇总进 5.3 验收报告核对表
