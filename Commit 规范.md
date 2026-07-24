# Commit 规范

> 本规范基于 [Conventional Commits 1.0.0](https://www.conventionalcommits.org/)，并针对 APIJSON ORM 框架的代码结构（`newSQLConfig` 请求解析链路、`@combine` 条件表达式引擎、多数据库方言、`Operation` 鉴权等）做了领域化裁剪。

---

## 1. Commit Message 结构

每条 Commit Message 由三部分构成：

```
<type>(<scope>): <subject>

<body>

<footer>
```

- **Header（必填）**：单行，≤ 72 字符；`type` 小写、`scope` 小写、`subject` 动词原形开头，不加句号。
- **Body（可选）**：说明 **为什么 / 做了什么**，每行 ≤ 100 字符；与 Header 之间空一行。
- **Footer（可选）**：放破坏性变更（`BREAKING CHANGE:`）、Issue 关联（`Closes #123`）、签核（`Signed-off-by:`）。

示例：

```
feat(combine): support nested boolean expression in @having

Allow @having to reuse parseCombineExpression so that HAVING
clauses can express "(toId>0 | avg(id)<100000) & status=1".
Default connector remains OR unless @having& is used, keeping
backward compatibility controlled by IS_HAVING_DEFAULT_AND.

Closes #482
```

---

## 2. Type（必填）

| type | 说明 | 示例场景 |
|------|------|----------|
| `feat` | 新增用户可感知的能力 | 新增 `@window` 关键字、新方言 ClickHouse |
| `fix` | 修复 Bug | 修复 `@combine` 在 JOIN ON 中 prepared value 顺序错乱 |
| `perf` | 仅性能优化，无行为变更 | `id`/`userId` 强制 AND 条件前置；SQL 缓存键优化 |
| `refactor` | 重构，非功能非修复 | 把 `newSQLConfig` 中 id 处理抽为 `IdentityPolicy` |
| `docs` | 仅文档变更 | 更新 `APIJSON 通用文档.md`、README、Javadoc |
| `style` | 格式化、分号、import 顺序，不改逻辑 | 对齐缩进、去除尾随空格 |
| `test` | 新增/修正测试，不改生产代码 | 为 `parseCombineExpression` 增加参数化测试 |
| `build` | 构建系统、依赖、CI | 升级 `maven-compiler-plugin`、增加 Testcontainer profile |
| `ci` | CI 配置与脚本 | GitHub Actions 增加 MySQL 8 / PG 16 矩阵 |
| `chore` | 杂项（不涉及 src/test） | 更新 `.gitignore`、LICENSE 年份 |
| `revert` | 回滚历史 commit | `revert: feat(combine): ...`，正文写 `This reverts commit <sha>` |
| `security` | 安全修复（可独立于 fix） | 收紧 `ALLOW_MISSING_KEY_4_COMBINE` 默认值 |

---

## 3. Scope（推荐填）

scope 与 [APIJSONORM/src/main/java/apijson/orm](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm) 下的核心模块对齐：

| scope | 对应代码 |
|-------|----------|
| `parser` | [AbstractParser](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractParser.java) |
| `object-parser` | [AbstractObjectParser](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractObjectParser.java) |
| `sql-config` | [AbstractSQLConfig.newSQLConfig](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L5477) |
| `combine` | [parseCombineExpression](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L3364) |
| `having` | `@having`/`@having&` 相关 |
| `join` | [Join](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/Join.java) / `parseJoin` |
| `subquery` | [Subquery](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/Subquery.java) / `@from` |
| `executor` | [AbstractSQLExecutor](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLExecutor.java) |
| `verifier` | [AbstractVerifier](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractVerifier.java) / [Operation](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/Operation.java) |
| `function` | [AbstractFunctionParser](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractFunctionParser.java) / 远程函数 |
| `script` | [script 包](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/script)（JSR223/JS） |
| `json` | [apijson.JSON](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/JSON.java) / fastjson 适配 |
| `dialect-mysql` / `dialect-pg` / `dialect-oracle` / ... | 具体数据库方言实现 |
| `model` | [model 包](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/model)（Access/Request/Document 等） |
| `exception` | [exception 包](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/exception) |
| `deps` | 依赖升级 |
| `release` | 版本发布（`chore(release): 8.3.0`） |

跨模块时用 `,` 分隔：`feat(sql-config,combine): ...`；无合适 scope 可省略但需在 body 说明。

---

## 4. Subject 写作约定

- 使用祈使句、动词原形（"add"、"fix"、"remove"），不用过去式。
- 首字母小写，结尾不加句号。
- 尽量 ≤ 50 字符；超过时考虑拆分 commit。
- 不使用模糊词："update stuff"、"fix bug"、"misc"。
- 对 `@combine` / `@having` / `@column` 等关键字用反引号包裹：`` fix(combine): disallow `id` key in `@combine` expression ``。
- 涉及安全闸常量（`MAX_COMBINE_*`、`ALLOW_MISSING_KEY_4_COMBINE`）时显式点名。

✅ 好例子：

- `fix(combine): preserve prepared value order under JOIN ON`
- `feat(sql-config): add @window keyword for window functions`
- `perf(sql-config): force id/userId AND condition before other keys`
- `security: set ALLOW_MISSING_KEY_4_COMBINE default to false`

❌ 反例：

- `fix bug`（无 type/scope/语义）
- `Fixed stuff.`（过去式、句号、模糊）
- `feat: new feature`（无 scope、无信息量）

---

## 5. Body 写作约定

- 回答 **"为什么这么改"**，而不是复述 diff。
- 涉及 `newSQLConfig` 主链路的改动，说明在 10 步流水线中的哪一步（参考 [APIJSON 规划及路线图.md](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSON%20%E8%A7%84%E5%88%92%E5%8F%8A%E8%B7%AF%E7%BA%BF%E5%9B%BE.md) 1.2 节）。
- 涉及 `@combine` 的改动，说明对布尔表达式语法、安全闸（depth/count/ratio）的影响。
- 列出副作用：行为兼容性、性能、对其他方言的连带影响。
- 可使用 Markdown 列表，但不要塞截图或外部链接（除 Issue/PR）。

---

## 6. Footer 约定

### 6.1 破坏性变更

```
BREAKING CHANGE: <subject 级简述>

<影响面与迁移说明>
```

破坏性变更触发规则（SemVer major bump）：

- 修改 `@combine` / `@having` 语法或默认连接符（如 [IS_HAVING_DEFAULT_AND](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L35) 默认值翻转）。
- 收紧安全闸默认值导致旧请求被拒。
- 删除或重命名 `@` 关键字、`Operation` 枚举值。
- 抽象方法签名变更（[SQLConfig](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/SQLConfig.java) / [Parser](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/Parser.java) / [Callback](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L6575)）。
- 提升 Java baseline（如 8 → 17）。

### 6.2 Issue 关联

- `Closes #123`、`Fixes #123`：合并后自动关闭。
- `Refs #456`：仅引用。
- `Implements #789`：对应需求。

### 6.3 签核

- DCO：`Signed-off-by: Name <email>`
- 若有共同作者：`Co-authored-by: Name <email>`

---

## 7. 特殊场景模板

### 7.1 新数据库方言

```
feat(dialect-<name>): add <Database> SQLConfig support

- implement gainXxxString for `<db>` quoting/limit/pagination
- register DATABASE_<NAME> in SQLConfig interface
- add TCK test profile with Testcontainer image <image>:<tag>

Implements #<issue>
```

### 7.2 修复 `@combine` 表达式 Bug

```
fix(combine): <一句话>

Root cause: 在 parseCombineExpression 中，<触发条件> 导致
<prepared value 顺序 / depth 计数 / key 重复> 与预期不一致。

Fix: <改动点>，并补充 <合法/非法> 用例。

Backward compat: <兼容说明>

Closes #<issue>
```

### 7.3 安全加固

```
security(combine): tighten MAX_COMBINE_RATIO default

Change MAX_COMBINE_RATIO from 1.0f to 0.5f to prevent a single
request from forcing exponential key evaluation. A new static
setter remains for operators who need the old behavior.

BREAKING CHANGE: requests whose @combine references more than
half of WHERE keys will now be rejected.
```

### 7.4 回滚

```
revert: refactor(sql-config): extract IdentityPolicy

This reverts commit 1a2b3c4d5e6f...

Reason: the introduced SPI broke binary compat for third-party
SQLConfig subclasses that override newSQLConfig.
```

---

## 8. 分支与 PR 约定

- 分支：`feat/<scope>-<short-kebab>` / `fix/<scope>-<short-kebab>` / `chore/<short>`。
- 一个 PR 内 commit 数量建议 ≤ 8，过多则 squash。
- PR 标题与最终 squash commit 的 Header 一致。
- CI 绿（编译、TCK、Checkstyle）+ 至少 1 名 Reviewer 通过才能合并。
- 合并策略：普通 PR 用 **Squash and merge**；Release PR 用 **Merge commit** 保留历史。

---

## 9. 自动化校验（建议）

- **commitlint**：`@commitlint/config-conventional` + scope 白名单（见 §3）。
- **husky/commit-msg**：本地拦截不合规提交。
- **PR 标题检查**：GitHub Action `amannn/action-semantic-pull-request`。
- **CHANGELOG**：用 `standard-version` 或 `release-please` 根据 type 自动分组生成：
  - `feat` → Features
  - `fix` / `security` → Bug Fixes / Security
  - `perf` → Performance
  - `BREAKING CHANGE` → ⚠ BREAKING CHANGES

---

## 10. 速查 Cheat Sheet

```
feat(sql-config): add @window keyword for OLAP functions
^───^ ^─────────^  ^───────────────────────────────────^
|     |            └─ subject：祈使句、≤50字符、无句号
|     └────────────── scope：来自 §3 白名单
└──────────────────── type：来自 §2 枚举
```

记住三句话：

1. **Header 一行说清"改了什么"**；
2. **Body 说清"为什么这么改、影响谁"**；
3. **破坏性变更必须写 `BREAKING CHANGE:` 并附迁移说明**。
