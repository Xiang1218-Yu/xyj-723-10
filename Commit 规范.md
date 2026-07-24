# APIJSON Commit 规范

> 基于 Conventional Commits 规范，针对 APIJSON ORM 项目定制
>
> `[规范]` 表示本文档定义的约定；`[源码]` 表示有源码依据的事实；`[示例]` 表示示例。

---

## 一、Commit Message 格式

`[规范]` 每条 commit message 由 **Header**、**Body**、**Footer** 三部分组成：

```
<type>(<scope>): <subject>

<body>

<footer>
```

### 格式要求

- **Header** 必填，Body 和 Footer 可选
- 每行不超过 72 个字符
- 使用英文编写（核心代码），中文注释可用于 Body 中的详细说明

---

## 二、Header 规范

### 2.1 Type 类型

| Type | 说明 | 示例 |
|------|------|------|
| `feat` | `[规范]` 新功能 | `feat(combine): support IN literal in @combine expression` |
| `fix` | `[规范]` Bug 修复 | `fix(mysql): fix MySQL8 REGEXP syntax incompatibility` |
| `perf` | `[规范]` 性能优化 | `perf(sql): optimize combineMap lookup from O(n) to O(1)` |
| `refactor` | `[规范]` 重构（非新增功能/非修bug） | `refactor(config): split AbstractSQLConfig condition builder` |
| `docs` | `[规范]` 文档变更 | `docs(readme): update @combine expression syntax` |
| `style` | `[规范]` 代码格式（不影响逻辑） | `style: unify indentation to Tab` |
| `test` | `[规范]` 测试相关 | `test(combine): add @combine nested parenthesis boundary test` |
| `chore` | `[规范]` 构建/工具/依赖 | `chore: upgrade maven-compiler-plugin to 3.12.1` |
| `ci` | `[规范]` CI/CD 配置 | `ci: add JDK 8/11/17 matrix build` |
| `revert` | `[规范]` 回滚提交 | `revert: revert "feat: xxx"` |
| `security` | `[规范]` 安全修复 | `security(verifier): enhance @raw SQL injection detection` |
| `db` | `[规范]` 数据库适配器 | `db(duckdb): add DuckDB adapter support` |
| `breaking` | `[规范]` 破坏性变更 | `breaking(api): remove deprecated legacy @combine comma format` |

### 2.2 Scope 作用域

`[规范]` Scope 指定本次变更影响的模块，`[源码]` 下列文件/模块均在代码库中存在：

| Scope | 对应文件/模块 |
|-------|---------------|
| `parser` | [AbstractParser.java](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractParser.java) — 请求解析入口 |
| `config` | [AbstractSQLConfig.java](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java) — SQL 配置/生成 |
| `combine` | @combine 条件表达式引擎（`[源码]` 核心在 parseCombineExpression L3364） |
| `oparser` | [AbstractObjectParser.java](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractObjectParser.java) — 对象解析 |
| `executor` | [AbstractSQLExecutor.java](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLExecutor.java) — SQL 执行 |
| `verifier` | [AbstractVerifier.java](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractVerifier.java) — 权限验证（6种角色 L76-L96） |
| `fparser` | [AbstractFunctionParser.java](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractFunctionParser.java) — 函数解析 |
| `join` | [Join.java](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/Join.java) — 12种JOIN连表（L21） |
| `subquery` | [Subquery.java](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/Subquery.java) — 子查询 |
| `logic` | [Logic.java](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/Logic.java) — 逻辑运算 &/\|/!（L15-L22） |
| `operation` | [Operation.java](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/Operation.java) — 13种操作枚举 |
| `model` | model/ 下的系统表模型（Access/Request/Table/Column...） |
| `script` | script/ 脚本引擎（JSR223/JavaScript） |
| `sql` | [SQL.java](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/SQL.java) — SQL 工具类 |
| `json` | JSON/JSONMap/JSONRequest 等基础类 |
| `mysql` | MySQL 数据库适配 |
| `pgsql` | PostgreSQL 适配 |
| `mssql` | SQL Server 适配 |
| `oracle` | Oracle 适配 |
| `clickhouse` | ClickHouse 适配 |
| `mongodb` | MongoDB 适配 |
| `redis` | Redis 适配 |
| `*` | 跨模块/全局变更 |

> `[源码]` 其余 DATABASE_LIST 中的 35 种数据库（如 DuckDB/StarRocks/Doris/TiDB 等）可直接使用数据库名小写作为 scope。

### 2.3 Subject 规则

- 使用祈使句（imperative mood），英文小写开头
- 结尾不加句号 `.`
- 简明扼要，描述"做了什么"而非"怎么做的"
- 长度建议 ≤ 50 字符

```
✅ feat(combine): support nested IN expression in @combine
❌ feat(combine): Added support for nested IN expression in @combine.
❌ feat(combine): 增加了对 @combine 中嵌套 IN 表达式的支持
```

---

## 三、Body 规范

`[规范]` Body 用于详细描述本次变更的 **动机**、**实现方式** 和 **影响范围**。

### 3.1 结构建议

```
<subject>

## Motivation
<为什么需要这个变更，解决了什么问题>

## Solution
<核心实现思路，关键算法/设计决策>

## Impact
<影响范围：兼容性/性能/安全性/数据库>
```

### 3.2 示例

`[示例]`
```
fix(combine): resolve prepared value order mismatch in expression mode

## Motivation
When @combine uses expression mode (e.g., "a | (b & c)"), the prepared
value list was in reverse order compared to the generated SQL placeholders,
causing incorrect parameter binding for PreparedStatement.

## Solution
In parseCombineExpression(), split value collection into two phases:
1. First collect AND-only conditions (non-expression keys)
2. Then collect expression-referenced keys in the order they appear
This guarantees placeholder order matches the value list order.

## Impact
- Affects all databases using PreparedStatement (default enabled)
- No breaking change to public API
- Related: AbstractSQLConfig gainWhereItem/gainCondition logic
```

---

## 四、Footer 规范

### 4.1 破坏性变更 (BREAKING CHANGE)

`[规范]`
```
BREAKING CHANGE: <description of the breaking change>
<migration path>
```

`[示例]`
```
breaking(config): remove deprecated combine comma format

BREAKING CHANGE: The old @combine:"key0,&key1,!key2" comma-separated format
has been removed. Use the boolean expression format instead:
@combine:"key0 & key1 & !key2"

Migration: Replace comma separators with " & " in all @combine values.
```

### 4.2 关联 Issue

`[规范]`
```
Closes #123
Fixes #456
Refs #789
```

### 4.3 关联 PR

`[规范]`
```
PR #100
```

---

## 五、特殊场景规范

### 5.1 数据库适配器新增

`[规范]` 新增数据库适配器需：
1. 在 [SQLConfig.java](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/SQLConfig.java) 添加 DATABASE_xxx 常量
2. 在 [AbstractSQLConfig.java#L146](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L146) DATABASE_LIST 注册
3. 继承 AbstractSQLConfig 实现方言差异方法
4. 在 callback.getSQLConfig 中添加实例化分支

`[示例]`
```
db(starrocks): add StarRocks adapter

- Extend AbstractSQLConfig for StarRocks dialect
- Implement isStarRocks() database detection
- Add DATABASE_STARROCKS constant and register in DATABASE_LIST
- Tested with StarRocks 3.x

Closes #200
```

### 5.2 安全修复

`[示例]`
```
security(verifier): block @raw subquery injection

Prevent privilege escalation via crafted @raw values that inject
subqueries bypassing access control. Added whitelist validation
for raw SQL fragments and audit logging for @raw usage.

Credit: Security Researcher <researcher@example.com>
```

### 5.3 性能优化

`[示例]`
```
perf(config): optimize combine key lookup O(n²) → O(n)

The isKeyInCombineExpr() method used repeated indexOf() calls
causing O(n²) worst-case. Replaced with single-pass tokenization
using a precomputed Set of referenced keys.

Benchmark: 10 conditions @combine, 1000 iterations
- Before: 45ms
- After: 3ms (15x speedup)
```

### 5.4 条件表达式引擎变更

`[规范]` 涉及 @combine 引擎（`[源码]` parseCombineExpression L3364-L3654）的 commit 需额外说明：
- 是否影响 MAX_COMBINE_DEPTH/MAX_COMBINE_COUNT/MAX_COMBINE_KEY_COUNT 安全限制（`[源码]` L61-L67）
- 是否改变 preparedValueList 顺序
- 是否影响两种模式（简单列表/布尔表达式）的兼容性

`[示例]`
```
feat(combine): support BETWEEN literal in boolean expressions

## Motivation
Currently @combine can only reference where keys, not inline
comparison values. Users want: "price% & date>" inline.

## Solution
Extend the tokenizer to recognize key:value pairs within expressions,
where value follows the same type rules as where values.

## Syntax
@combine:"price%:[10,100] & date>:2024-01-01"

## Limits
- Max 2 inline values per expression (MAX_COMBINE_KEY_COUNT=2)
- Same depth/count limits apply (MAX_COMBINE_DEPTH=2, MAX_COMBINE_COUNT=5)
```

---

## 六、完整示例

`[示例]`
```
feat(combine): add parentheses-aware tokenizer for nested expressions

## Motivation
The original @combine parser used a simple character-by-character state
machine that could not handle deeply nested expressions efficiently.
When MAX_COMBINE_DEPTH was reached, it threw a generic error without
indicating which level caused the overflow.

## Solution
Introduce a two-pass approach:
1. First pass: tokenize the expression into Key/Op/Paren tokens
2. Second pass: evaluate the token tree with depth tracking

This enables:
- Better error messages (token position + expected syntax)
- Future extensibility for inline values
- O(n) parsing regardless of nesting complexity

## Impact
- Internal refactoring, no API change
- Error messages now include character position
- Prepared value ordering preserved (backward compatible)
- No performance regression (benchmarked ±2%)

Refs #150
PR #165
```

---

## 七、禁止事项

`[规范]`

1. **禁止** 一次 commit 混合多个不相关变更
2. **禁止** commit message 为空或仅写 "fix bug"、"update" 等模糊描述
3. **禁止** 提交编译不通过的代码（CI 必须通过）
4. **禁止** 在 commit message 中包含敏感信息（密钥、密码、内部URL）
5. **禁止** 用中文写 subject（核心代码仓库），但 Body 中可使用中文注释
6. **禁止** `git commit -m "xxx"` 跳过 Body 对于 feat/fix/perf/breaking 类型（这些必须有 Body 说明）

---

## 八、版本标签规范

`[规范]` Tag 命名遵循语义化版本，`[源码]` 当前版本为 8.2.0（[pom.xml#L8](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/pom.xml#L8)）：

```
v<major>.<minor>.<patch>
```

示例：
- `v8.2.0` — `[源码]` 当前版本
- `v8.3.0` — 新数据库适配器 + combine 增强
- `v8.2.1` — Bug修复补丁
- `v9.0.0` — 破坏性大版本升级（如重构泛型体系）
