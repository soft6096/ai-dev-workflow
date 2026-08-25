---
name: ai-dev-workflow
description: Use when the user wants to develop a module or feature with AI following a structured spec-driven process — 按流程开发/按规格开发/生成功能清单/写技术方案/拆任务/契约测试/AI 编码收敛. Triggers on phrases like "按流程开发 XX 模块", "从需求到代码", "先生成功能清单", "写技术方案", "拆 tasks", "生成规范不跑偏的代码". Applies to Java service projects (Controller/Listener/Job) and similar layered backends.
---

# AI 开发完整流程（需求 → 上线）

## 核心原则

**不跑偏**：让 AI 生成规范、可验证、可维护的代码。五道防线：

1. **类型化模板** — 每种产物格式固定（Controller/Listener/Job），AI 没有发挥空间
2. **验收场景前置** — 方案阶段写死"怎么算对"，4.2 契约测试阶段翻译成测试
3. **约束硬规则** — 项目约束是"能检查对错"的规则，AI 逐条对照
4. **角色分离** — 写测试的标准 ≠ 实现代码的 AI，防止"骗绿"
5. **收敛兜底** — 实现后按类型逐条核对，防规格漂移

## 0.0 项目/工程初始化（脚手架，从零生成项目必做）

**适用范围**：从零生成/初始化 Spring Boot 工程（含 AI 一次生成整个项目）。**只开发既有项目里的模块则跳过本节**，直接走 1.1。

**为什么必须有这一步**：流程 1.1~5.2 假设"项目已存在"，但一次生成整个项目时，工程骨架缺基础设施会导致**启动即炸**。历史踩坑（真实事故）：

| 缺失/错误 | 启动/运行现象 |
|---|---|
| 只有单个 application.yml，无环境拆分 | dev 配置带到生产，连错库 |
| datasource 只配 url/username/password，无 HikariCP 池参数 | 高并发连接打满，难排查 |
| JDBC URL 写 `characterEncoding=utf8mb4` | `Unsupported character encoding 'utf8mb4'` 启动失败 |
| JDBC URL 端口默认 3306，实际环境非 3306（NAT/容器映射） | `Access denied for user 'xxx'@'localhost'` 连接被拒 |
| 实体 `@TableField(fill=...)` 但项目无 MetaObjectHandler | 一插即炸：`Column 'create_time' cannot be null` |
| 库/表沿用旧 schema，与代码 Entity 不匹配 | `Unknown column 'real_name' in 'field list'` |
| MySQL 8 配了 5.1 旧驱动 | 字符集/时区/加密协议一串兼容问题 |

**脚手架硬性检查清单（生成项目后逐项核对，缺一项即不合格）**：

- [ ] 配置文件四件套：`application.yml` + `-dev` + `-qa` + `-online`（公共/开发/测试/生产），profile 激活 `${SPRING_PROFILES_ACTIVE:dev}` 不写死
- [ ] HikariCP 显式池参数（maximum-pool-size / minimum-idle / pool-name 等），禁止 datasource 三件套裸奔
- [ ] JDBC URL 字符集：**不写 `characterEncoding=utf8mb4`**（8.x 报 Unsupported character encoding），推荐不写或 `characterEncoding=UTF-8`；`serverTimezone=Asia/Shanghai` 必带
- [ ] JDBC URL 端口与目标环境实际一致（禁止默认 3306 想当然）
- [ ] MyBatis-Plus 插件配置（分页 + 乐观锁，参考 java-code-standards `04-templates/ConfigTemplate.java`）
- [ ] MetaObjectHandler 实现（createTime/updateTime 自动填充，参考 `04-templates/MyMetaObjectHandler.java`）
- [ ] 建库/建表显式 utf8mb4；**库初始化按当前 schema.sql 全量重建，禁止沿用旧库旧表**
- [ ] MySQL 驱动 Connector/J 8.x（Spring Boot BOM 管理，无 5.1 旧驱动）

> 规范出处：配置文件/连接池/字符集 → java-code-standards `01-java/application-config-standards.md`；表/schema → database-standards `standards/table-design-standards.md`；依赖 → build-standards `standards/dependency-standards.md`。模板见 `templates/0.0-项目初始化.md`。

## 两种使用方式

### 配套规范 skill 触发矩阵（各步骤必须加载对应规范）

| 流程步骤 | 必须加载的规范 skill | 覆盖内容 |
|---|---|---|
| 0.0 项目/工程初始化 | **build-standards** + **java-code-standards**（application-config + 04-templates）+ **database-standards**（table-design） | 脚手架基础设施：多环境配置/HikariCP/JDBC 字符集/端口/MyBatis-Plus 插件/MetaObjectHandler/schema 一致性/驱动版本 |
| 1.1~5.2 所有 md 产物 | **`templates/00-中间产物-MD样式规范.md`** | 速览框 / 对齐表格 / mermaid 配色 / callout / 自检（生成任何中间产物 md 前必读） |
| 2.1 `/constraints` | **build-standards** | 技术架构选型、依赖/版本管理、多模块结构 |
| 3.x `/design` | **database-standards** + **java-code-standards** | DDL/表设计/索引（db）；Controller/Service 分层接口规范（java） |
| 4.1 `/task-breakdown` | **database-standards** | Mapper XML 触发条件（简单查询禁 XML） |
| 4.2 `/contract-tests` | **test-standards** | 契约测试规范（三态/先红后绿/方法名英文驼峰） |
| 5.1 `/implement` | **java-code-standards**（必读）+ **database-standards** + **build-standards** + **comment-standards** | 写代码前先加载：Java → java-code-standards（命名/分层/注释引用）+ comment-standards（注释）；SQL/MyBatis-Plus → database-standards；pom/依赖 → build-standards |
| 5.2 `/accept` | **comment-standards** + 各规范自检清单 | 注释核对 + 命名/分层/公共组件核对 |

> 规范 skill 与流程 skill 的关系：**流程管「怎么走」，规范管「生成物长什么样」**。编码 Agent 写代码前必须先加载对应规范 skill，保证**生成物符合规范**（兜底在流程 skill，不靠项目 AGENTS.md）。

### 方式一：分步执行（推荐，可随时介入）

用斜杠命令按步骤执行，每步可独立触发、独立检查：

| 编号 | 命令 | 用途 |
|---|---|---|
| 1.1 | `/feature-list` | 功能清单 |
| 1.2 | `/clarify` | 盘问歧义/入口复杂度/边界/依赖（方案前必做） |
| 2.1 | `/constraints` | 项目约束（含公共组件规范） |
| 3.x | `/design` | 技术方案（3.0 通用骨架 + 类型模板组合；含公共组件识别） |
| 4.1 | `/task-breakdown` | 任务拆解（公共组件入 Phase 0.5） |
| 4.2 | `/contract-tests` | 接口契约测试（先红） |
| 5.1 | `/implement` | AI 编码（让测试变绿） |
| 5.2 | `/accept` | 验收报告（含重复代码核对） |
| 附加 | `/gen-comments` | 存量代码补注释（有 spec 派生 / 无 spec 事实注释，不猜意图） |

命令文件在 `commands/` 目录，每个命令自带输入/执行/完成标准。`/gen-comments` 为独立命令，不属于五步流程，按需用于历史代码注释补全。

### 落盘规则（所有命令统一遵守）

| 产物类型 | 落盘位置 |
|---|---|
| 文档（md） | `docs/<模块名>V<版本号>-<YYYYMMDDHHMMSS>/` 目录下（每模块每版本一个目录） |
| 测试代码 | `src/test/java/<对应包>/` |
| 业务代码 | `src/main/java/<对应包>/` |
| 项目约束（全局共享） | `docs/2.1-项目约束.md`（跨模块，仅一份） |

> **docs 模块目录命名**：`<模块名>V<版本号>-<YYYYMMDDHHMMSS>`——模块名 + V 开头版本号（**从 PRD/需求文档获取**，如 V1.2.0）+ 连字符 + 功能清单生成时间戳（YYYYMMDDHHMMSS）。例：`docs/订单模块V1.2.0-20260824103000/`。目录在 1.1 功能清单步骤创建（`/feature-list` 从需求文档读取版本号），后续步骤产物全部落入该目录，跨版本开发互不覆盖。

示例（订单模块 V1.2.0）：

```
docs/
├── 2.1-项目约束.md                # 全局约束
└── 订单模块V1.2.0-20260824103000/
    ├── 1.1-功能清单.md
    ├── 1.2-澄清问题清单.md
    ├── 3.1-订单创建-技术方案.md
    ├── 4.1-订单创建-任务拆解.md
    └── 5.2-订单创建-验收报告.md
src/
├── main/java/com/example/order/    # 业务代码
└── test/java/com/example/order/    # 测试代码
```

### 方式二：一句话全流程

用户说"按流程开发 XX 模块"时，按下方五步顺序执行。**每步产物必须给人确认后才能进入下一步**（见下方"中间产物确认闸门"）。中途用户可要求跳到任一 `/命令`。

## 中间产物确认闸门（强制，所有步骤适用）

**每一步产出中间产物后，AI 必须停下来，等人在场确认，禁止连续执行到下一步。**

确认流程（每步通用）：

```
1. 该步产物落盘（md/代码/测试）
2. AI 对照该步"完成标准"逐项自检，向人展示：
   - 产物位置（文件路径）
   - 完成标准核对结果（每项 ✅/❌）
   - 关键内容摘要（如功能清单表格、约束条目、方案要点、测试结果）
3. 人检查产物 → 确认通过 / 提出修改意见
4. 人确认后 → 才进入下一步；有修改意见 → AI 修改后重新展示确认
```

- **每步一个闸门，禁止跨步**：1.1 确认后才能 1.2；1.2 确认后才能 2.1……依次类推
- 关键闸门（必须人明确确认，不能默认通过）：
  - **1.1 功能清单**：类型/验收标准/依赖是否齐全
  - **1.2 澄清问题清单**：所有"待确认"项必须人逐项确认（已有问题清单模板）
  - **2.1 项目约束**：技术选型与硬规则是否符合预期
  - **3.x 技术方案**：接口/DDL/验收场景无"待定"
  - **4.1 任务拆解 + 4.2 契约测试**：任务粒度 + 测试红（先红证据）
  - **5.1 编码结果**：测试绿证据 + 代码抽查
  - **5.2 验收报告**：无未解决差异 + quickstart 调通证据
- 人未确认前，AI 不得继续下一步（这是"防跑偏"的第零道闸门：人有权在任意一步叫停修正）

## 五步流程

每步的详细指令见对应 `commands/` 命令文件，模板见 `templates/`。

### 1.1 功能清单（人主导）→ `/feature-list`

- **输入**：需求文档
- **输出**：功能清单总览文件（表格），每项含：类型（Controller/Listener/Job）、优先级、依赖、**验收标准**（可验证，不能写"实现 XX 功能"）
- **完成标准**：每项有类型 + 可验证验收标准 + 依赖标注

### 1.2 澄清（人 + AI）→ `/clarify` ★ 方案前必做

- **输入**：功能清单
- **输出**：澄清问题清单（人确认答案）+ 更新后的功能清单（补边界/风险列、依赖）
- **动作**：逐项盘问四类问题——**歧义**（模糊词/未定义字段/不可验证标准）、**入口形态与复杂度**（有/无入口、简单/复杂、跨功能项规则归属）、**边界用例**（重复/并发/空值/超长/状态流转）、**依赖**（功能间/外部系统/数据）
- **完成标准**：无"待确认"残留；每个功能项边界用例已列出；依赖完整

> 这是防跑偏的第一道闸门：歧义带进方案，后面全偏。宁可在这里多盘问，不要带歧义进入 `/design-*`。

### 2.1 项目约束（人主导）→ `/constraints`

- **输入**：功能清单
- **输出**：技术架构 + 项目约束（硬规则，能检查对错，禁止"尽量/建议"）+ **注释规范**（全量注释规则：所有类/变量/方法必须注释 + 方法体步骤注释 + // WHY: 约定，含正反例）+ **公共组件规范**（包结构约定 common/util|base、第 2 个使用点必须抽公共、禁止复制粘贴）
- **完成标准**：每条约束可检查，版本号写死，AI 可直接读取

### 3.x 技术方案（人 + AI 辅助）→ `/design`

- **输入**：功能清单
- **动作**：读功能项的类型列，选模板组合——**3.0 通用骨架必用**（基本信息/验收场景/公共组件识别/注释要求/自检），类型特有内容按四段式（入口定义/数据契约/核心逻辑/关注点）取自对应类型模板：Controller → `3.1`、Listener → `3.2`、Job → `3.3`；无入口形态（配置/工具/纯数据）→ 仅用 3.0，跳过类型段；每功能项做**架构自检**（识别重复逻辑，判定抽工具类 common/util 还是抽象基类 common/base，结果写入方案"公共组件识别"小节）
- **输出**：每功能项一份完整技术方案 md（3.0 通用章节 + 类型四段内容合并）
- **完成标准（质量闸门）**：第零段定性正确；通用章节完整（验收场景三态/公共组件识别/注释要求）；类型四段齐全；接口明细逐 URL 且失败示例与错误码一一对应；核心逻辑含**调用骨架**（分层 + `→` 调用标注 + 中文步骤 + Mapper 内联 SQL，不写实现代码）；DDL 完整且业务 SQL 全量在方案内（无独立 sql 目录、无"DDL 外推 schema.sql"）；类型特有横切关注点（权限事务/幂等死信/防重入批处理）已写明

### 4.1 任务拆解 + 4.2 契约测试（人主导，AI 辅助）→ `/task-breakdown` `/contract-tests`

- **输入**：技术方案 md + 项目约束
- **输出**：任务拆解.md（公共组件拆入 **Phase 0.5**，先写契约测试先红 → 实现变绿；无重复则留空注明）+ 契约测试（**初始为红**）+ DDL
- **完成标准**：每任务有文件路径+验收方法；**测试先跑一遍确认是红的**；验收场景全部翻译成用例；公共组件复用点在功能任务中已写死（调用/继承，禁止复制）

### 5.1 AI 编码 + 5.2 收敛验收（AI 主导，人验证）→ `/implement` `/accept`

- **输入**：任务拆解.md + 红色测试 + 项目约束
- **输出**：绿色代码（含按注释规范生成的注释）+ 验收报告（含注释核对 + **重复代码核对** + **quickstart 调通证据**）
- **完成标准**：全部测试绿；验收报告无"未解决差异"；公共组件已实现且使用点复用、无 ≥2 处相同方法体；人工抽查 + **quickstart 调通证据齐全（必填：全新构建 + 真实启动日志 + 按验收场景逐条实测，仅测试绿不算通过）**

## 硬性约束（写入编码 Agent 提示词，不可协商）

```
1. 测试文件是验收标准：禁止修改断言、禁止删除测试、禁止 @Disabled
2. 严格按 Phase 顺序：先数据层 → 再公共组件（0.5）→ 再业务 → 再接口/消费/调度
3. 必须遵守《项目约束》的全部规则（逐条对照，含"注释规范"与"公共组件规范"章节）
4. **生成的代码必须遵守规范 skill 约定**：写任何代码前先加载对应规范（Java → java-code-standards + comment-standards；SQL/表/MyBatis-Plus → database-standards；pom/依赖 → build-standards），生成物不符合规范即不合格、需返工
5. 每个任务完成后独立提交（commit message = 任务 ID + 描述）
6. 测试红时先排查实现，禁止"改测试迁就实现"
7. 不实现任务拆解.md 之外的东西（不做"顺手优化"）；Phase 0.5 列出的公共组件必须实现并被复用点调用/继承。编码中发现设计外的重复逻辑：不擅自抽取，标注 `// TODO(收敛)` 由验收阶段统一处理
8. 写代码时同 pass 生成注释（内容从技术方案派生，禁止事后补/编造），遵守注释规范（见 comment-standards skill）
9. 收尾阶段核对注释、权限、事务、幂等、防重入
```

## 常见错误

| 错误 | 修正 |
|---|---|
| 跳过 3.x 直接让 AI 写代码 | 方案有歧义，后面全跑偏。3.x 是质量闸门，宁可多花时间 |
| 让实现 AI 顺便写测试 | 会"骗绿"（改断言、写假测试）。测试必须人写或人审 |
| 约束写成"尽量/建议" | 无法检查 = 无效约束。必须是能判对错的规则 |
| 验收场景只写正常路径 | 非法/边界场景缺失 = 测试有洞。三态都要有 |
| 实现完不核对方案 | 规格漂移悄悄发生。验收核对应是最后一关 |
| 让 AI 编码时"顺手"抽公共类 | 会跑偏（改任务外代码）。公共组件必须方案期识别（3.x 架构自检）+ 拆入 Phase 0.5，编码只实现不复抽 |
| 委派断链后直接编码却不加载规范 skill | 生成物不符合规范、验收返工。规范 skill 加载是写码前第一步，见硬性约束第 4 条 |
| 澄清阶段没确认入口/复杂度 | 第零段判定靠 AI 猜，方案四段选错、全链跑偏。入口形态/业务复杂度/状态机归属必须在 1.2 澄清盘问并由人确认 |
| 验收只跑 mock 测试就关闭（测试绿 ≠ 能启动） | 交付后启动即炸（配置错误/连接失败/旧 jar 过期配置/库表未初始化，mock 测试全部测不出）。5.2 必须附 quickstart 调通证据：全新构建 + 真实启动日志 + 按验收场景逐条实测，缺证据验收不通过 |
