---
description: 关键规范自动核对（第⑤步编码收尾兜底闸门 + 验收复核）。触发词：规范核对/自动核对/check-standards/兜底核对/规范检查。用法：/check-standards <项目路径>。作用：用 grep/ast-grep 逐项扫描生成代码，核对关键规范是否执行到位，输出证据报告；HIGH ❌ 项升级人工核对。
---

# 关键规范自动核对（Check Standards）

编码收尾 / 验收复核的**兜底闸门**：规范条目多、分散在多个 skill，AI 编码可能漏执行。本命令用 **grep/ast-grep 实际扫描产出**，逐项核对关键规范，**禁止凭记忆答 ✅**。

## 模式判定与核对依据（执行前必读，先判模式再核对）

**核对判定以项目约束为准，规范 skill 默认值仅兜底**——与硬性约束 4 同一口径：

| 模式 | 判定依据（先读哪个文件） | 选型类核对项怎么判 |
| :--- | :--- | :--- |
| **标准模式**（新项目） | `docs/<模块>V<版本>-<时间戳>/2.1-项目约束.md`（技术架构表含 ★ 人确认的选型） | 按 2.1 人确认的选型判定（如 Log4j2 → 按 Log4j2 查，不强制 logback-spring.xml） |
| **存量适配模式**（老项目） | `docs/0.5-存量代码扫描.md` + `2.1-项目约束-存量适配.md`（老项目实际约定） | 按老项目约定判定（如老项目用 R<T> 返回体 → 不判 Response<T>；用 Apifox 不写注解 → 按老约定） |
| 判定不了 | 停止并向人确认项目模式（禁止按默认值猜） | — |

**选型敏感核对项**（#1 接口文档 / #2 日志框架 / #3 数据访问层 / #10 返回体）：判定标准随项目模式变化，**先读 2.1 约束/0.5 扫描再判**；其余核对项（SQL 注释/DDL 注释/事务/注入/WHERE/密码/分页等）为通用安全项，任何模式都按规范判定。

## 执行方式（核心规则）

1. **先判模式**（见上"模式判定与核对依据"），读对应约束文件，确定选型类核对项的判定口径
2. 对下方每一项，**实际执行「标准检查指令」**（grep/ast-grep），把**命令 + 命中行（文件:行号）粘贴为证据**
3. 无命中 → 记 `✅`；命中违规 → 记 `❌`（附证据 + 严重度）
4. **HIGH/代码规范/注释组 ❌ → 先向用户确认，再执行补齐**：把该项证据展示给人，问"是否按规范补齐？"——**用户确认后**才执行补齐 → 重跑该项指令 + 新证据（已修复）；补齐后仍无法满足 → **升级人工核对**（在报告中标注"需人工核对"）。**禁止未经用户确认自动改代码**（补齐全过程有人的确认闸门）
5. **INFO ❌** → 记录参考，不阻塞（如涉敏感/重复代码等可顺手处理项，同样先告知用户再动）
6. 输出《关键规范核对报告》（逐项 ✅/❌ + 证据），结果汇总进 5.2 验收报告"关键规范落地核对表"

> [!WARNING] 证据强制 + 用户确认闸门
> **每项必须给出实际执行的 grep/ast-grep 命令与命中行**。只写 ✅/❌ 无证据 = 未核对，打回重跑。禁止"没做就声称做了"。
> **发现未执行到位项 → 必须停下来让用户确认是否执行补齐**（人确认后才动手改代码），不自动默默补齐。

## 检查项清单

### HIGH（必须处理；❌ 补齐 → 重跑 → 仍 ❌ 升级人工核对）

| # | 核对项 | 标准检查指令（实际执行） | 判定标准（全部满足才 ✅） | 规范出处 |
| :---: | :--- | :--- | :--- | :--- |
| 1 | 接口文档支持（OpenAPI/Swagger）⚠️选型敏感 | `grep -n "springdoc\|knife4j\|springfox\|swagger" pom.xml`；`grep -rln "@Tag\|@Operation\|@Api\|@ApiOperation" src/main/java`；`grep -rln "@Schema\|@ApiModelProperty" src/main/java` | **按 2.1 选型/老约定判定**：标准模式 springdoc/knife4j → pom 有依赖 + Controller 有 @Tag/@Operation + DTO/VO 有 @Schema（Apifox 约定注解照写）；老项目 Swagger2 → 按 @Api/@ApiOperation/@ApiModelProperty 体系；老约定不写注解（纯 Apifox 在线文档）→ 按老约定，记"按老约定" | build-standards `dependency-standards.md` 4.6；java-code-standards `api-doc-standards.md` |
| 2 | 日志框架支持 ⚠️选型敏感 | `ls src/main/resources/logback*.xml src/main/resources/log4j2*.xml`；`grep -rln "@Slf4j" src/main/java`；`grep -rn "System.out" src/main/java`；`grep -n "log4j2\|logback" pom.xml` | **按 2.1 选型/老约定判定**：Logback → logback-spring.xml 存在（控制台+滚动文件+环境级 level）；Log4j2 → log4j2 配置存在且 pom 无 logback 并存；任何模式：全类 @Slf4j、无 System.out | java-code-standards `04-logging-standards.md`；build-standards 4.7 |
| 3 | SQL 在 XML（手写 SQL 统一收拢）⚠️选型敏感 | `grep -rn "@Select\|@Insert\|@Update\|@Delete\|<script>" src/main/java`；`ls src/main/resources/mapper/*.xml` | **按 2.1/老约定判定**：标准模式 → Java 无注解 SQL，手写 SQL 全在 XML，namespace 一致；老项目 ORM/约定不同（注解 SQL/JPA）→ 按 0.5 扫描的数据访问约定判定（2.1 存量适配约束内要求的仍遵守） | database-standards `mybatis-plus/mybatis-xml-standards.md` |
| 4 | 详细设计 SQL 注释 | `grep -rn "CREATE TABLE\|SELECT \|INSERT INTO\|UPDATE " docs/*/3.*-技术方案*.md` | 技术方案《数据模型与 SQL》所有 SQL 带注释：DDL 每字段 COMMENT；查询/DML 每条 `--` 注释（用途+归属 Mapper+关键条件） | ai-dev-workflow `3.0-技术方案-通用骨架.md`；database-standards `sql-standards.md` 1.5 |
| 5 | DDL 字段注释 | `grep -A 25 "CREATE TABLE" src/main/resources/db/schema.sql docs/*/3.*-技术方案*.md` | CREATE TABLE **每个字段**带 `COMMENT`（含义/枚举/单位/时区）+ 表级 COMMENT；无裸字段无注释 | database-standards `table-design-standards.md` |
| 6 | JSON 入参/出参产物 | `ls docs/*/3.*.2-接口清单（前后端通用）.md`；`grep -n "入参\|出参" docs/*/3.*.2-接口清单*.md` | 每个 Controller 功能项的 `3.<功能项序号>.2-<功能名>-接口清单（前后端通用）.md` 已生成；每接口有 URL+方法+**JSON 入参示例**+**JSON 出参示例（成功/失败）**；与技术方案一致（无 HTTP 接口功能项标注跳过） | ai-dev-workflow `templates/3.4-接口清单-前后端通用.md` |
| 7 | 事务 rollbackFor | `grep -rn "@Transactional" src/main/java` | 每个 `@Transactional` 均带 `rollbackFor = Exception.class`（无 rollbackFor → ❌） | java-code-standards `service-impl-standards.md` |
| 8 | SQL 注入（拼接/`${}`） | `grep -rn '\${' src/main/resources/mapper/*.xml`；`grep -rn '"SELECT \|"INSERT \|"UPDATE \|"DELETE ' src/main/java` | XML 无 `${}` 拼接值（仅白名单排序/表名，需人工确认）；Java 无字符串拼接 SQL；无 `apply()/last()` 传用户输入 | database-standards `sql-standards.md`；java-code-standards `security-standards.md` |
| 9 | UPDATE/DELETE 带 WHERE | `grep -rn -i "^[[:space:]]*UPDATE \|^[[:space:]]*DELETE " src/main/resources/mapper/*.xml` | 所有 UPDATE/DELETE 语句带 WHERE（无 WHERE → HIGH ❌，全表变更风险） | database-standards `data-safety.md` |
| 10 | 统一返回体 ⚠️选型敏感 | `grep -rn "Map<" src/main/java/*/controller/*.java`；`grep -rln "class R\b\|class Result\b" src/main/java` | **按 2.1/老约定判定**：标准模式 → Controller 返回 `Response<T>`/`PageResult<T>`，无 Map 裸返回，无 R/Result 类；存量模式 → 按老项目返回体（如 `R<T>`）判定，无 Map 裸返回 | java-code-standards `01-naming-standards.md` / `controller-standards.md` |
| 11 | 密码加密 | `grep -rn -i "md5\|sha1\|DigestUtils" src/main/java` | 无 MD5/SHA1 用于密码存储（用 BCrypt 等慢哈希）；命中需人工确认场景 | java-code-standards `security-standards.md` |
| 12 | 分页上限 | `grep -rn "pageSize\|PageQuery" src/main/java/*/dto/*.java` | 分页入参 pageSize 有 `@Max`/上限校验 | java-code-standards `controller-standards.md` |

### INFO（参考项，❌ 记录不阻塞）

| # | 核对项 | 标准检查指令 | 判定标准 |
| :---: | :--- | :--- | :--- |
| 13 | 敏感信息进日志 | `grep -rn "password\|token\|secret" src/main/java` | 日志/异常/响应不含密码/token 明文（命中核对是否脱敏） |
| 14 | 集合命名 | `grep -rn "List<.*> records\|List<.*> codes\|Set<.*> values" src/main/java` | 集合字段用 `xxxList/xxxSet/xxxMap` 后缀 |
| 15 | 魔法值/缓存 key | 抽查常量类 | 无裸魔法值；缓存 key 集中常量定义 |

### 场景化（对应类型文件存在时才核）

| # | 核对项 | 触发条件 | 判定标准 |
| :---: | :--- | :--- | :--- |
| 16 | Job 防重入 + 批处理 | `grep -rln "@Scheduled" src/main/java` 有命中 | 有分布式锁/状态位防重入；批处理带 LIMIT；无长事务 |
| 17 | Listener 幂等 + 死信 | `grep -rln "@RabbitListener\|@KafkaListener" src/main/java` 有命中 | 消费幂等（唯一键/状态位）；重试有上限 + 死信队列；无 catch 静默 |
| 18 | 文件上传安全 | `grep -rln "MultipartFile" src/main/java` 有命中 | 扩展名+MIME 双白名单；UUID 重命名；大小限制 |

### 代码规范组（所有代码必核，❌ 先向用户确认再补齐）

> 规范出处：java-code-standards `00-common/*` + `01-java/*`（命名/分层/注入/异常/实体边界）。质量类项（重复代码）需人工抽查，机器只扫可查信号。

| # | 核对项 | 标准检查指令（实际执行） | 判定标准 | 严重度 |
| :---: | :--- | :--- | :--- | :--- |
| C1 | 构造器注入 | `grep -rn "@Autowired" src/main/java`（**应为空**） | 无字段注入（`@Autowired` 字段）；统一构造器注入（`@RequiredArgsConstructor` + final 字段） | HIGH |
| C2 | 分层边界 | `grep -rn "Mapper" src/main/java/*/controller/*.java`；Controller 方法体抽查 | Controller 不注入 Mapper、不写业务逻辑/SQL/事务（只收参→调 Service→返回） | HIGH |
| C3 | Entity 不暴露 | `grep -rn "Entity\|@TableName" src/main/java/*/controller/*.java`；接口方法出入参抽查 | 接口出入参用 DTO/VO，不暴露 Entity（禁 Entity 直接作请求/响应） | HIGH |
| C4 | 异常处理 | `grep -rn "throw new RuntimeException\|catch (.*) {}" src/main/java` | 无裸 `RuntimeException`（用 `BusinessException(ErrorCode, msg)`）；无空 catch 吞异常 | HIGH |
| C5 | 命名单字母/泛称 | `grep -rn "catch (.* e)\|throw new.*(.* e)" src/main/java`；抽查类/变量名 | 无单字母类名（R/Result）、异常参数非 `e`、无泛称变量（dto/data/obj） | INFO |
| C6 | 公共组件复用 | 扫描 `common/util`、`common/base` 与业务代码重复方法体（抽查 2-3 处） | Phase 0.5 公共组件已实现且被复用；无 ≥2 处相同/相似方法体（发现 → 提示抽公共，需人工确认） | INFO |

### 注释组（所有代码必核，❌ 先向用户确认再补齐）

> 规范出处：comment-standards `standards/comment-standards.md`（全量注释：所有类/变量/方法注释 + 步骤注释 + // WHY: + 禁翻译式）。机器查"覆盖率"，"质量"（业务含义/不翻译式）需人工抽查——覆盖率 ❌ 与质量可疑项都先向用户确认再处理。

| # | 核对项 | 标准检查指令（实际执行） | 判定标准 | 严重度 |
| :---: | :--- | :--- | :--- | :--- |
| N1 | 类注释覆盖率 | `grep -rn "public class\|public interface" src/main/java`，逐个核对类前有 `/** */` 类注释 | 所有类（Controller/Service/Impl/Mapper/Entity/DTO/VO/Config/Utils）有类注释，第一句概述职责 | HIGH |
| N2 | 方法 Javadoc 覆盖率 | `grep -rn "public " src/main/java`，逐个核对方法前有 `/** */`（含 @param/@return 业务含义） | 所有方法（含测试方法）有 Javadoc：功能 + @param/@return 写业务含义（不重复参数名） | HIGH |
| N3 | 步骤注释 + WHY | 抽查方法体 ≥2 逻辑步骤的方法（Service/Job/Listener） | 方法体 ≥2 个逻辑步骤有编号注释（`// 1.`）；复杂逻辑有 `// WHY:`；注释与代码一致 | INFO（抽查） |
| N4 | 禁翻译式注释 | 抽查注释（对照代码） | 无逐行翻译式注释（`// 价格乘以数量`）；注释写业务含义非复述代码 | INFO（人工） |

## 输出格式

```text
=== 关键规范核对报告 ===
项目: <路径>    时间: <时间戳>    模式: 标准/存量适配
[✅] 1 OpenAPI/Swagger    证据: pom.xml:12 springdoc-openapi; UserController.java:3 @Tag
[❌] 7 事务 rollbackFor  证据: OrderServiceImpl.java:45 @Transactional（无 rollbackFor）(HIGH)
[❌] C1 构造器注入       证据: OrderServiceImpl.java:12 @Autowired（字段注入）(HIGH)
[❌] N2 方法 Javadoc     证据: ProductServiceImpl.java:30 public 方法无 /** 注释 (HIGH)
...
结论: 12 HIGH + C/N 组：通过 X / 未通过 Y（INFO 命中 Z 条参考）
⚠️ 未执行到位项（请用户确认是否补齐）: #7、C1、N2 …
```

- 报告**落盘**：`docs/<模块名>V<版本号>-<YYYYMMDDHHMMSS>/check-standards-<功能名>.md`（产物落盘规则见 SKILL.md）；12 项 HIGH + C/N 组结论汇总进 5.2 验收报告「关键规范落地核对表」
- **未执行到位项必须向用户确认是否补齐**（展示证据 → 人确认 → 执行补齐 → 重跑）；全部通过才算过闸门；用户选择不补齐/补齐后仍 ❌ → 标注"需人工核对"

## 完成标准

- [ ] **模式已判定**（标准/存量适配，读对应约束文件），选型敏感项（#1/#2/#3/#10）按项目约束/老项目约定判定，非规范默认值一刀切
- [ ] 12 项 HIGH + 代码规范组（C1-C6）+ 注释组（N1-N4）全部实际执行检查指令并附证据（文件:行号）
- [ ] 通过项全 ✅；未到位项**已向用户确认是否补齐**（非自动默默补）——确认后补齐重跑通过，或用户选择不补/仍 ❌ → 已标注"需人工核对"（不静默吞掉）
- [ ] INFO/场景化项已核对并记录（命中项说明处理）
- [ ] 核对报告已落盘 `docs/<模块名>V<版本号>-<YYYYMMDDHHMMSS>/check-standards-<功能名>.md`
- [ ] 12 项 HIGH + C/N 组结果已汇总进 5.2 验收报告核对表
