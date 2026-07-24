# APIJSON 规划及路线图（Roadmap）

> 版本基线：`APIJSONORM 8.2.0`（Java 1.8）
> 定位：以 `AbstractParser → AbstractObjectParser → AbstractSQLConfig` 为核心的自动化 ORM 引擎，让客户端用一个 JSON 描述请求即可完成增删改查与关联查询。

---

## 一、产品定位与设计目标

APIJSON 的核心价值是**用一份 JSON 协议替代大量手写接口**：客户端发什么 JSON，服务端就返回结构对应的 JSON，无需为每个查询单独写后端代码。围绕这一目标，引擎需要持续在四个维度演进：

| 维度 | 目标 | 支撑的核心代码 |
| --- | --- | --- |
| **表达能力** | 让 JSON 能表达复杂 SQL（多表 JOIN、子查询、聚合、逻辑组合） | `@combine` 表达式引擎、`parseJoin`、`Subquery` |
| **安全性** | 默认拒绝、按角色/权限放行，防注入、防深度攻击 | `AbstractVerifier`、`parseCorrectRequest`、`MAX_COMBINE_*` / `MAX_QUERY_DEPTH` 限制 |
| **可扩展性** | 支持多数据库方言、远程函数、脚本、自定义回调 | `Callback`/`SimpleCallback`、`AbstractFunctionParser`、`ScriptExecutor` |
| **性能** | 减少 SQL 次数、缓存复用、预编译 | 数组主表缓存、`preparedValueList`、`@cache` |

---

## 二、当前架构现状（Baseline）

```
HTTP 层(应用侧)
   │  parser.parse(requestString)
   ▼
AbstractParser.parseResponse(M)          ← 全局关键字剥离 / 登录·内容·角色校验 / 事务
   │  onObjectParse()
   ▼
AbstractObjectParser.parse()             ← 单对象拆解：sqlRequest / customMap / functionMap / childMap
   │  setSQLConfig() → newSQLConfig()
   ▼
AbstractSQLConfig.newSQLConfig()(静态)   ← 请求 → SQLConfig（列/条件/组合/分组/排序/值）
   │  gainSQL()
   ▼
SQLExecutor.execute()                    ← JDBC 执行
   │
   ▼
response() 逐层回填 → extendSuccessResult() → JSON 响应
```

**已具备的能力**
- 全量条件操作符：`key$`(LIKE)、`key~`/`key*~`(REGEXP)、`key%`(BETWEEN)、`key{}`(IN/范围)、`key}{`(EXISTS)、`key<>`(JSON contains)、`key>=/<=/>/<`(比较)、`key!`(取反)。
- `@combine` 双形态：新版布尔表达式（`&`/`|`/`!` + 括号）与旧版逗号列表，见 [AbstractSQLConfig.java](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Bulbasaur/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java) 的 `parseCombineExpression`。
- 深度/数量限制：`MAX_COMBINE_DEPTH=2`、`MAX_COMBINE_COUNT=5`、`MAX_QUERY_DEPTH` 等安全阈值。
- 远程函数（`key()`）、脚本执行（JSR223 / JavaScript）、多数据库方言分支。

---

## 三、演进路线图（Roadmap）

### 阶段 1 — 巩固与文档化（近期）
- [ ] 完善 `newSQLConfig` 与 `parseCombineExpression` 的内联注释与开发者文档（本次输出）。
- [ ] 为条件操作符建立完整语法对照表与单元测试基线（对照 http://apijson.cn/unit）。
- [ ] 梳理 `Callback` 扩展点文档，明确应用侧需实现的抽象方法（`getSQLConfig`、`createObjectParser`、`createSQLConfig`）。

### 阶段 2 — 表达式引擎增强（中期）
- [ ] `@combine` 表达式支持可配置深度/数量上限（当前为静态常量，改为按角色/表配置）。
- [ ] 表达式解析器从「单遍扫描直出 SQL」演进为可选 AST 模式，便于做 SQL 优化与更精确的报错定位。
- [ ] `@having` 与 `@combine` 语法统一，减少两套代码路径的维护成本。

### 阶段 3 — 性能与可观测（中期）
- [ ] 强化 `@cache` 策略：分级缓存（RAM/ROM/远程）与主表结果复用统计。
- [ ] 完善调试元数据（`sql:generate|cache|execute`、`depth`、`time`、`trace`）的可视化与采样开关。
- [ ] 预编译 `preparedValueList` 复用池，降低高并发下的重复解析开销。

### 阶段 4 — 生态与多端（远期）
- [ ] 扩展更多数据库方言（当前已具备方言分支，如 PostgreSQL `~*`、`@>`）。
- [ ] 与 FASTJSON2 分支能力对齐（见 README 中 fastjson2 分支）。
- [ ] 提供更强的自动化文档/Mock/权限可视化工具链。

---

## 四、风险与技术债

| 项 | 说明 | 建议 |
| --- | --- | --- |
| 请求 map 复用副作用 | `newSQLConfig` 在 `finally` 中把移除的关键字**还原**回 request（供数组逐项复用），逻辑隐蔽 | 补充注释 + 测试覆盖数组多项场景 |
| 两套 combine 代码路径 | 新表达式与旧 `combineMap` 并存，`getWhereString` 需分支判断 | 阶段 2 统一 |
| 抽象方法散落应用侧 | 具体 `SQLConfig`/`ObjectParser`/`Parser` 在使用方实现，ORM 只定义抽象/接口 | 提供参考实现与脚手架 |
| 安全阈值为静态常量 | `MAX_COMBINE_*` 全局生效，难以按业务分级 | 阶段 2 配置化 |

---

## 五、里程碑衡量指标
- **接口替代率**：JSON 协议可覆盖的查询占比。
- **SQL 次数**：单请求平均生成 SQL 条数（缓存命中率）。
- **安全拦截率**：非法/越权/超深请求的拦截比例。
- **方言覆盖**：支持的数据库类型数量。
