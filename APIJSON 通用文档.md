# APIJSON 通用文档

> 本文档基于对 APIJSON ORM 8.2.0 源码的通读，以 [newSQLConfig](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L5477-L6288) 请求解析主链路与 [parseCombineExpression](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L3364-L3654) 条件表达式引擎为主线，系统性说明协议语义、关键字、扩展点与排错。

---

## 目录

1. [设计哲学与核心抽象](#1-设计哲学与核心抽象)
2. [请求方法与入口](#2-请求方法与入口)
3. [请求解析全貌：newSQLConfig 十步走](#3-请求解析全貌newsqlconfig-十步走)
4. [`@` 关键字参考](#4--关键字参考)
5. [条件操作符（Key 后缀）](#5-条件操作符key-后缀)
6. [`@combine` 条件表达式引擎](#6-combine-条件表达式引擎)
7. [JOIN 与子查询](#7-join-与子查询)
8. [分组、聚合、排序与分页](#8-分组聚合排序与分页)
9. [远程函数与脚本引擎](#9-远程函数与脚本引擎)
10. [安全、鉴权与 Operation](#10-安全鉴权与-operation)
11. [多数据源与方言](#11-多数据源与方言)
12. [扩展点](#12-扩展点)
13. [配置项与默认值](#13-配置项与默认值)
14. [错误与排错](#14-错误与排错)

---

## 1. 设计哲学与核心抽象

APIJSON 是腾讯开源的 **"零 CRUD 后端"** 协议：客户端用一段 JSON 描述"要什么数据 + 什么条件"，服务端将其翻译为 SQL 并返回结构化 JSON。核心模型：

- **一切请求皆 JSON 树**：一个 JSON 对象对应一张表/一次子查询；嵌套即关联；数组即一对多。
- **约定优于配置**：表名、字段名直接作为 key，条件用后缀表达（`id>`, `name~`），组合用 `@combine`。
- **多数据源一套协议**：[SQLConfig](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/SQLConfig.java) 中声明了 40+ 种数据库（MySQL/PG/Oracle/ClickHouse/Doris/StarRocks/MongoDB/Redis/Kafka/Milvus/InfluxDB…）。

### 1.1 分层架构

```
        ┌──────────────────────────────────────────────┐
HTTP →  │ Parser(AbstractParser.parseResponse)         │  JSON → 请求对象树
        │   └─ ObjectParser(AbstractObjectParser)      │  单表对象：setSQLConfig → executeSQL → response
        │        └─ SQLConfig(AbstractSQLConfig)       │  newSQLConfig 工厂：JSON 片段 → SQLConfig
        │             └─ @combine 引擎                  │  parseCombineExpression：布尔表达式 → WHERE
        │        └─ SQLExecutor(AbstractSQLExecutor)   │  JDBC 执行、缓存、事务、批处理
        │   └─ Verifier(AbstractVerifier)              │  角色/权限/Operation 校验
        │   └─ FunctionParser(AbstractFunctionParser)  │  远程函数、JSR223 脚本
        └──────────────────────────────────────────────┘
```

### 1.2 关键类型速查

| 类型 | 文件 | 角色 |
|------|------|------|
| Parser | [AbstractParser](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractParser.java) | 顶层入口、对象树递归、全局属性（version/tag/format/database…） |
| ObjectParser | [AbstractObjectParser](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractObjectParser.java) | 一张表对象的生命周期：解析成员 → setSQLConfig → executeSQL → response/子对象 |
| SQLConfig | [AbstractSQLConfig](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java) | 元数据（method/table/alias/where/content/column/group/having/order/limit）+ SQL 拼装 |
| SQLExecutor | [AbstractSQLExecutor](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLExecutor.java) | JDBC 执行、缓存（RAM/ROM/ALL）、批处理、事务 |
| Verifier | [AbstractVerifier](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractVerifier.java) | 访问控制、Operation（MUST/REFUSE/TYPE/VERIFY/EXIST/UNIQUE/INSERT…） |
| FunctionParser | [AbstractFunctionParser](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractFunctionParser.java) | `key()` 形式的远程函数/存储过程调用 |
| Callback | [AbstractSQLConfig.Callback](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L6575-L6597) | SPI：`getSQLConfig/newId/getIdKey/getUserIdKey/onMissingKey4Combine` |

---

## 2. 请求方法与入口

### 2.1 RequestMethod 枚举

定义见 [RequestMethod.java](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/RequestMethod.java#L14-L54)：

| 方法 | HTTP 映射 | 语义 | 写操作？ |
|------|-----------|------|----------|
| GET | GET | 查询（明文） | 否 |
| HEAD | HEAD | 查询总数（`count(*)`） | 否 |
| GETS | POST /gets | 安全 GET（请求体不回显、校验严格） | 否 |
| HEADS | POST /heads | 安全 HEAD | 否 |
| POST | POST | 新增 | 是 |
| PUT | PUT | 增量更新 | 是 |
| DELETE | DELETE | 删除（或软删除） | 是 |
| CRUD | POST /crud | 单请求多语句 | 混合 |

判定方法：[isQueryMethod](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/RequestMethod.java#L83-L85)（读）、[isUpdateMethod](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/RequestMethod.java#L91-L93)（写）、[isPublicMethod](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/RequestMethod.java#L99-L101)（明文）。**写操作必须带 WHERE 条件**，否则 [getWhereString](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L3344-L3346) 抛 `UnsupportedOperationException("写操作请求必须带条件！！！")`。

### 2.2 入口方法

- `Parser.parse(String json)` —— 字符串入口，内部 `parseRequest` 校验 JSON 合法性后进入 [parseResponse(M)](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractParser.java#L505)。
- 顶层关键字在进入对象解析前处理：`@format`、`@version`、`@tag`、`@database`/`@datasource`/`@schema`/`@namespace`/`@catalog`、`@explain`、`@cache` 等（见 [AbstractParser.java:512-540](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractParser.java#L512-L540)）。

---

## 3. 请求解析全貌：newSQLConfig 十步走

[AbstractSQLConfig.newSQLConfig](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L5477-L6288) 是"单表 JSON 片段 → SQLConfig"的工厂。下面是**带源码行号**的十步流水线。

### Step 1：前置校验
- `@explain:true` 在非 DEBUG 模式抛 `UnsupportedOperationException`（[L5485](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L5485)）。
- `@database` 必须命中 `DATABASE_LIST`，否则抛 `UnsupportedDataTypeException`（[L5490](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L5490)）。

### Step 2：实例化具体 SQLConfig
通过 [Callback.getSQLConfig](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L5500) 按 database 返回方言实现，设 alias、database、datasource、namespace、catalog、schema。`isProcedure` 直接返回（存储过程分支）。

### Step 3：JOIN 处理
[parseJoin](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L6300) 对每个 `Join` 递归调用 `newSQLConfig`，生成副表 config、ON config、OUTER config，并在 HEAD/HEADS 时强制改为 SELECT 关联键以优化性能。

### Step 4：主键与 userId 强制 AND 条件
- `id`/`id{}`/`userId`/`userId{}` 先被过滤：Number 必须 `>0`，String 不能空；Collection 形式去无效值、去重。
- `id` 与 `id{}` 同时出现时，`id` 必须 ∈ `id{}` 集合（[L5568-L5580](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L5568-L5580)）。
- POST 且未显式传 id 时，由 [Callback.newId](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L6608)（默认时间戳递增）生成。

### Step 5：抽取 @ 关键字
[L5639-L5690](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L5639-L5690) 一次性读出并从 request 中 `remove`：`@role/@explain/@cache/@database/@datasource/@namespace/@catalog/@schema/@from/@column/@null/@cast/@combine/@group/@having/@having&/@sample/@latest/@partition/@fill/@order/@key/@raw/@json/@method`。剩下的 key 要么是字段，要么是子对象。

### Step 6：@null 与 @cast 展开
- `@null:"tag,pictureList"` → 对每个名字 `request.put(name, null)`，若已存在非 null 值抛错（[L5694-L5707](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L5694-L5707)）。
- `@cast:"date:DATE,amount:DECIMAL"` → 登记到 `castMap` 用于后续 `CAST(... AS ...)`。

### Step 7：POST 分支
所有剩余 key 必须是合法标识符 [StringUtil.isName](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L5760)，拼装 `columns` 与 `values`（id、userId 追加到最前），调用 `config.setValues(valuess)` 走批量 INSERT。

### Step 8：非 POST 分支，构建 WHERE 与 CONTENT
核心在 [L5800-L5985](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L5800-L5985)：

1. `isWhere = method != PUT`：除 PUT 外其余方法的剩余键全部当条件。
2. 先把 id/userId 加入 `tableWhere` 与 `andList`（强制 AND 前置，利用索引）。
3. **软删除**：若 `enableFakeDelete`，对非 DELETE 自动追加 `deletedKey != deletedValue` / `deletedKey = notDeletedValue`。
4. 解析 `@combine`：
   - 若 `combine` 被切分后只剩一段（含 `&`/`|`/`!`/`(` 等运算符），进入布尔表达式模式；
   - 否则按逗号列表式，`&key`/`|key`/`!key`/`key`（默认 OR）入 andList/orList/notList。
5. 遍历剩余 key：
   - 非 PUT 的键全部作为 WHERE；
   - PUT 时若键名被 `@combine` 表达式引用（[isKeyInCombineExpr](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L6639)）也作为 WHERE；
   - 其余进入 `tableContent`（SET 内容）。
6. 键后缀（`>`, `<`, `~`, `{}`, `<>` 等）由 [gainWhereItem](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L3897) 分派（见 §5）。

### Step 9：DELETE 软删除改 PUT
[L5987-L6008](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L5987-L6008)：当 `AbstractVerifier.ACCESS_FAKE_DELETE_MAP` 对该表有配置，DELETE 被改写成 `PUT SET deletedKey=deletedValue`，并回调 `config.onFakeDelete(map)` 追加字段（如 deletedTime）。

### Step 10：列选择、聚合、分页与还原
- `@column` 解析：`DISTINCT ` 前缀 → `PREFIX_DISTINCT`；`fun(key)` 片段原样保留；普通字段按 `,`/`;` 切分；`@raw` 标记的片段走原始 SQL。
- `@having`/`@having&`：与 `@combine` 同引擎，但 `isHaving=true`；默认 OR、可用 `@having&` 强制 AND（[L6058-L6070](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L6058-L6070)，开关 [IS_HAVING_DEFAULT_AND](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L35)）。
- 最后把所有 `@` 关键字 `put` 回 request，保证后续对象解析还能读到（[L6246-L6284](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L6246-L6284)）。

最终 SQL 在 [gainSQL(boolean prepared)](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L4932) 中按方法拼出 SELECT/INSERT/UPDATE/DELETE，调用 [gainConditionString](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L5081) 完成 WHERE/HAVING/GROUP/ORDER/LIMIT 的组装。

---

## 4. `@` 关键字参考

完整常量见 [JSONMap.java](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/JSONMap.java#L166-L205)。**只有在表对象内出现的关键字会进入 `newSQLConfig`**；最外层关键字由 Parser 处理。

### 4.1 路由与元信息

| Key | 作用 | 示例 |
|-----|------|------|
| `@database` | 指定数据库类型（40+ 种），默认 MySQL | `"@database":"POSTGRESQL"` |
| `@datasource` | 数据源标识（多数据源路由） | `"@datasource":"ds_order"` |
| `@schema`/`@catalog`/`@namespace` | 命名空间，默认见 [DEFAULT_SCHEMA](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L72)（`sys`） | `"@schema":"analytics"` |
| `@role` | 角色，被 Verifier 用于鉴权 | `"@role":"ADMIN"` |
| `@explain` | 返回 SQL 与执行计划（仅 DEBUG） | `"@explain":true` |
| `@cache` | 缓存策略 `RAM`/`ROM`/`ALL` | `"@cache":"RAM"` |

### 4.2 查询结构

| Key | 作用 |
|-----|------|
| `@column` | 返回字段或 SQL 函数；`DISTINCT ` 前缀去重；`;` 分隔函数片段；`@raw` 允许原始片段 |
| `@from` | 子查询，值为 [Subquery](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/Subquery.java) 对象 |
| `@where`（隐式） | 表对象内所有"字段:值"即为 WHERE 条件（除 PUT SET） |
| `@group` | `GROUP BY` 字段 |
| `@having` | HAVING 条件，默认 OR，引擎与 `@combine` 同 |
| `@having&` | HAVING 条件，强制 AND |
| `@order` | `ORDER BY`，如 `"date-,id+"`（`-` DESC、`+` ASC） |
| `@combine` | 条件组合表达式，见 §6 |

### 4.3 特殊值与类型

| Key | 作用 |
|-----|------|
| `@null` | `"a,b"` 声明这些字段为 `IS NULL`/`SET ...=NULL` |
| `@cast` | `"date:DATE,amount:DECIMAL"` 类型转换 |
| `@json` | 把字段按 JSON 输出（PG JSONB/MySQL JSON） |
| `@string` | 字段按字符串输入 |
| `@trim` | trim 空白 |
| `@key` | 字段映射 / SQL 函数，如 `year:left(date,4)`；支持 `name_tag:(name,tag)` 多列 IN |
| `@raw` | 声明哪些字段值为原始 SQL 片段，value 为 `""` 时整段为 raw |
| `@method` / `@get` / `@post` ... | 在 CRUD 中给子对象单独指定方法 |

### 4.4 行为控制

| Key | 作用 |
|-----|------|
| `@try` | 该对象异常不影响整体 |
| `@drop` | 只作为条件提供者，结果丢弃 |
| `@default` | 为空时回填默认值 |
| `@sample` / `@latest` / `@partition` / `@fill` | 时序/采样/分区/填充专用（ClickHouse/InfluxDB/TDengine 等） |

---

## 5. 条件操作符（Key 后缀）

在 [gainWhereItem](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L3897-L3977) 中按后缀分派。每个后缀最终生成一个参数化条件片段。

| 后缀 | keyType | 含义 | 例 | 生成（近似） |
|------|--------:|------|----|--------------|
| (无) | 0 | 等值 / IS NULL | `"status":1` | `status = ?`；值为 null → `status IS NULL` |
| `!`（列名后） | 0 | 不等 / IS NOT NULL | `"status!":1` | `status != ?`；null → `status IS NOT NULL` |
| `>` | 9 | 大于 | `"id>":100` | `id > ?` |
| `<` | 10 | 小于 | `"age<":18` | `age < ?` |
| `>=` | 7 | 大于等于 | `"score>=":60` | `score >= ?` |
| `<=` | 8 | 小于等于 | `"score<=":100` | `score <= ?` |
| `{}` | 4 | IN / BETWEEN / 函数范围 | `"id{}":[1,2,3]` → `id IN (?,?,?)`；`"id{}":">0,<=100"` → 范围 |
| `}{` | 5 | EXISTS 子查询 | `"Comment{}` 子对象 | `EXISTS (SELECT 1 FROM ...)` |
| `<>`, `{&}`, `{\|}` | 6 | JSON/数组包含 | `"tags<>":['a','b']` | JSON 包含 |
| `~` | 2 | 正则匹配 | `"name~":"^A"` | `name REGEXP ?`（方言相关） |
| `*~` | -2 | 忽略大小写正则 | `"name*~":"^a"` | `LOWER(name) REGEXP LOWER(?)` |
| `%` | 3 | BETWEEN | `"date%":"2024-01-01,2024-12-31"` | `date BETWEEN ? AND ?` |
| `$` | 1 | 全文检索 | `"content$":"hello"` | `MATCH(content) AGAINST(? IN BOOLEAN MODE)`（MySQL） |

附加规则：
- **列长度函数**：列名后加 `[` 会被 [gainKey](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L4014) 包成 `length(column)`，支持 `"content[{}":">0,<=1000"`。
- **远程函数条件**：值里以函数名开头（如 `"length(name)>0"`）会被识别为 raw 表达式。
- **子查询值**：value 为 `Subquery` 时走 [gainSubqueryString](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L3996)。
- **安全过滤**：列名由 [gainRealKey](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L3947) 过白名单正则，禁止 `/* */`、`--`、`#`、`;`、空格等注入字符（见 [PATTERN_SCHEMA/RANGE/FUNCTION](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L75-L78)）。

---

## 6. `@combine` 条件表达式引擎

### 6.1 两种形态

**A. 逗号列表式（兼容/PUT 推荐）**

```json
"@combine": "status,&type,|category,!deleted"
```

- `&k` → AND；`|k` → OR；`!k` → NOT；无前缀默认 OR。
- PUT 请求**禁止** `|key` 与 `!key`（[L5901-L5912](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L5901-L5912)）。
- 逗号列表式不支持括号嵌套；需要嵌套请用表达式式。

**B. 布尔表达式式（5.0+）**

```json
"@combine": "status & (type | !category)"
```

语义即布尔逻辑，运算符：

| 运算符 | 含义 | 空白规则 |
|--------|------|----------|
| `&` | AND | **左右必须各一个空格** |
| `\|` | OR | **左右必须各一个空格** |
| `!` | NOT（单目） | **右侧紧贴键名，不接空格**；不可接 `)`/`&`/`\|`/`!` |
| `(` `)` | 分组 | `(` 右侧、`)` 左侧不允许空格 |

严格的空白规则由 [parseCombineExpression](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L3364) 逐条校验：首尾空格、连续空格、缺失连接符、括号不匹配都会抛 `IllegalArgumentException`。

### 6.2 求值语义

- 对表达式中的每个键名，先按 `:` 切出 `column:inlineExpr`（允许内联 raw 片段，见 [L3442-L3454](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L3442-L3454)）；否则从 `conditionMap` 取 value 走 `gainWhereItem`（WHERE）或 `gainHavingItem`（HAVING）生成条件片段 `wi`。
- 每个片段被包装成 `( wi )`，若前置 `!` 则为 [gainCondition(true, wi)](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L4800) → `NOT(wi)`。
- 运算符按出现位置串接为 `AND`/`OR`/`NOT`。
- **未在表达式中出现的条件**会以 AND 追加到尾部（WHERE 模式）或在 HAVING 中作为前缀；这样保证 `id/userId` 强制条件一定生效（[L3608-L3644](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L3608-L3644)）。

### 6.3 安全闸

所有阈值见 [AbstractSQLConfig.java:61-67](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L61-L67)，子类可覆盖对应 `getMaxXxx()`：

| 常量 | 默认 | 含义 |
|------|------|------|
| `MAX_WHERE_COUNT` | 10 | WHERE 条件总数上限（HAVING 用 `MAX_HAVING_COUNT=5`） |
| `MAX_COMBINE_DEPTH` | 2 | 括号嵌套深度上限 |
| `MAX_COMBINE_COUNT` | 5 | 表达式中可引用的条件 key 数量上限 |
| `MAX_COMBINE_KEY_COUNT` | 2 | 单个 key 在表达式中最多被引用次数 |
| `MAX_COMBINE_RATIO` | 1.0 | 表达式 key 数 / 条件总键数 的最大比值 |
| `ALLOW_MISSING_KEY_4_COMBINE` | true | 表达式中的 key 在 request 中缺失时是否放行（回调 [onMissingKey4Combine](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L6596)） |

### 6.4 硬规则

- `id`/`id{}`/`userId`/`userId{}` **禁止**出现在 `@combine` 表达式（它们被强制 AND 前置），见 [L5883-L5891](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L5883-L5891)。
- 空 key、空表达式、括号不匹配、连接符缺失都会立即抛错。
- prepared value 顺序：先 AND 追加项的占位值，后表达式内顺序值（[L3629-L3650](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L3629-L3650)）。

### 6.5 示例

```jsonc
// WHERE: status=1 AND (type=2 OR category IS NULL)
{
  "status": 1,
  "type": 2,
  "category": null,
  "@combine": "status & (type | !category)"
}
```

```jsonc
// HAVING: count(id) > 5 OR avg(score) >= 80
{
  "@group": "userId",
  "@having": "countId>0,avgScore>={\"from\":80}",
  "@combine": "countId | avgScore>="
}
```

> 注意示例第二个用逗号列表式 OR；若想嵌套，改写为 `@combine:"(countId | avgScore>=) & ..."`。

---

## 7. JOIN 与子查询

### 7.1 JOIN

- 通过外层数组 key 的 `@join` 或在主对象使用 `"User{}": { "@join": ... }` 声明（见 [JSONRequest](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/JSONRequest.java#L99-L166)）。
- [Join](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/Join.java) 支持：
  - **SQL JOIN**（INNER/LEFT/RIGHT/FULL）与 **APP JOIN**（内存关联）由 [Join.isAppJoin](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/Join.java) 区分。
  - `on`/`outer` 子对象都通过 `newSQLConfig` 独立生成 config（[L6343-L6359](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L6343-L6359)），可在其中继续使用 `@combine`。
  - JOIN 副表强制 `setKeyPrefix(true)`，避免列名冲突。

### 7.2 子查询

- `@from` 值为一个 [Subquery](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/Subquery.java)（内含 `range`/`from`/`join`/自定义对象）。
- EXISTS 用 `Table{}{ ... }` 键后缀 `}{`。
- 关联子查询中的 `@combine` 会走独立的 prepared value 列表，合并回外层时由 [parseCombineExpression](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L3398) 反向重组。

---

## 8. 分组、聚合、排序与分页

| 能力 | Key | 说明 |
|------|-----|------|
| 分组 | `@group` | `"userId,date"` → `GROUP BY userId,date` |
| 聚合条件 | `@having` / `@having&` | 复用 `@combine` 引擎；值可写函数表达式如 `"avg(score)>80"` |
| 排序 | `@order` | `"date-,id+"`；`+` ASC、`-` DESC；方言函数可写 `length(name)-` |
| 返回字段 | `@column` | `id,name;sum(amount)`；`DISTINCT ` 前缀 |
| 分页 | 顶层/数组 `count,page` | 默认 [DEFAULT_QUERY_COUNT=10](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractParser.java#L77)，最大 [MAX_QUERY_COUNT=100](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractParser.java#L78)，页码可由 [IS_START_FROM_1](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractParser.java#L75) 决定起点 |
| 总数 | HEAD 方法或数组 `"query":2` | 自动包成 `SELECT count(*) ...` |

分页限制见 [AbstractParser.java:76-83](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractParser.java#L76-L83)：`MAX_QUERY_PAGE/COUNT/UPDATE_COUNT/SQL_COUNT/OBJECT_COUNT/ARRAY_COUNT/QUERY_DEPTH` 防御大数据与递归炸弹。

---

## 9. 远程函数与脚本引擎

- **远程函数**：键名带括号即调用，如 `"isVerified()":"/User/login"`，由 [FunctionParser](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/FunctionParser.java) / [AbstractFunctionParser](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractFunctionParser.java) 在请求前（`"`0`"`阶段）/响应后（`"+"`阶段）分别调用，见 [onFunctionResponse](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractObjectParser.java#L1077)。
- **脚本**：[script 包](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/script) 提供 [ScriptExecutor](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/script/ScriptExecutor.java) SPI，默认有 JSR223 与 [JavaScriptExecutor](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/script/JavaScriptExecutor.java) 两种实现，供 `@raw`/Function 场景使用。

---

## 10. 安全、鉴权与 Operation

### 10.1 认证与角色
- [AbstractVerifier](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractVerifier.java) 校验 `@role`、登录态（[NotLoggedInException](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/exception/NotLoggedInException.java)）、访问权限（[Access](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/model/Access.java) 模型）。
- 写操作必须带条件（见 §2.1）。

### 10.2 Operation
[Operation](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/Operation.java) 枚举定义在 `Request` 表中可配置的服务端校验规则：

| 值 | 用途 | 示例 |
|----|------|------|
| MUST | 必传字段 | `"id,userId"` |
| REFUSE | 禁传字段 | `"password"` |
| TYPE | 类型校验 | `"id":"NUMBER"`, `"pictureList":"URL[]"` |
| VERIFY | 格式/范围校验 | `"phone~":"PHONE"`, `"status{}":[1,2,3]` |
| EXIST | 联合存在性 | `"name,category"` |
| UNIQUE | 联合唯一性 | `"name,category"` |
| INSERT | 不存在则补加对象 | 子对象 |
| UPDATE | 存在则更新对象 | 子对象 |

### 10.3 注入防御
- 列名、表名、schema 过 [PATTERN_SCHEMA / PATTERN_RANGE / PATTERN_FUNCTION](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L75-L78)，禁止 `/* */`、`--`、`#`、`;`、空格、`/` 与 `*` 同现。
- 全量 PreparedStatement 占位符；`@raw` 必须由服务端在 `rawMap` 中预注册白名单。
- `@explain` 仅 DEBUG 可用，避免暴露执行计划。
- `MAX_QUERY_DEPTH/OBJECT_COUNT/ARRAY_COUNT/SQL_COUNT` 限制递归深度与 SQL 总数。

---

## 11. 多数据源与方言

[SQLConfig](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/SQLConfig.java#L20-L57) 内置常量支持 40+ 种数据库。接入新方言的路径：

1. 继承 `AbstractSQLConfig` 并实现 `isXxx()`、`gainXxxString`（引用符、分页、全文搜索、JSON 操作等）。
2. 在 `Callback.getSQLConfig` 中按 database 返回实例。
3. 若需要不同数据源路由，通过 `@datasource` + 自己的 `DataSource` 路由 SPI。

跨库 JOIN 时，`parseJoin` 强制副表 `database` 与主表一致，否则抛错（[L6325](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L6325)）；跨数据源场景用 APP JOIN。

---

## 12. 扩展点

| SPI | 位置 | 用途 |
|-----|------|------|
| [ParserCreator](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/ParserCreator.java) / [SQLCreator](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/SQLCreator.java) / [VerifierCreator](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/VerifierCreator.java) | 工厂 | 创建 Parser/SQLConfig/Verifier |
| [Callback](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L6575) / [SimpleCallback](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L6604) | `newSQLConfig` 钩子 | id 生成、主键/用户键名、`@combine` 缺键回调 |
| [OnParseCallback](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/OnParseCallback.java) | 通用 | 解析过程回调（权限/审计） |
| [ScriptExecutor](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/script/ScriptExecutor.java) | 脚本 | JSR223/JS 扩展 |
| `AbstractSQLConfig` 的 `protected` 方法 | 子类 | 覆盖 `getMaxCombineDepth` 等阈值、`gainXxxString` 方言方法 |
| `TABLE_KEY_MAP` / `COLUMN_KEY_MAP` | 静态映射 | 隐藏真实表/字段名（[L88-L92](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L88-L92)） |
| `ALLOW_PARTIAL_UPDATE_FAIL_TABLE_MAP` | 静态映射 | 允许批量增删改部分失败的表（[L96](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L96)） |

---

## 13. 配置项与默认值

### 13.1 安全/分页（AbstractParser）

| 常量 | 默认 | 含义 |
|------|------|------|
| `IS_START_FROM_1` | false | 页码从 1 还是 0 开始 |
| `MAX_QUERY_PAGE` | 100 | 最大页码 |
| `DEFAULT_QUERY_COUNT` | 10 | 默认每页条数 |
| `MAX_QUERY_COUNT` | 100 | 单页最大条数 |
| `MAX_UPDATE_COUNT` | 10 | 单次写操作最大记录数 |
| `MAX_SQL_COUNT` | 200 | 单请求 SQL 总数 |
| `MAX_OBJECT_COUNT` | 5 | 单请求嵌套对象数 |
| `MAX_ARRAY_COUNT` | 5 | 单请求嵌套数组数 |
| `MAX_QUERY_DEPTH` | 5 | 嵌套深度 |

### 13.2 Combine 与 Having（AbstractSQLConfig）

| 常量 | 默认 | 含义 |
|------|------|------|
| `MAX_HAVING_COUNT` | 5 | HAVING 条件数 |
| `MAX_WHERE_COUNT` | 10 | WHERE 条件数 |
| `MAX_COMBINE_DEPTH` | 2 | 括号嵌套深度 |
| `MAX_COMBINE_COUNT` | 5 | 表达式 key 数 |
| `MAX_COMBINE_KEY_COUNT` | 2 | 单 key 引用次数 |
| `MAX_COMBINE_RATIO` | 1.0 | 表达式/总条件比值 |
| `ALLOW_MISSING_KEY_4_COMBINE` | true | combine 缺键放行 |
| `IS_HAVING_DEFAULT_AND` | false | 5.0 兼容开关：`@having` 默认 AND/OR |
| `IS_HAVING_ALLOW_NOT_FUNCTION` | false | 5.0 兼容开关：HAVING 允许非函数表达式 |
| `ENABLE_WITH_AS` | false | 开启 WITH AS |
| `IGNORE_EMPTY_STRING_METHOD_LIST` | null | 对哪些方法忽略空串 |
| `IGNORE_BLANK_STRING_METHOD_LIST` | null | 对哪些方法忽略空白串 |

---

## 14. 错误与排错

### 14.1 常见异常

| 场景 | 异常 | 排查 |
|------|------|------|
| 写操作无条件 | `UnsupportedOperationException("写操作请求必须带条件！！！")` | PUT/DELETE/POST(array) 必须带 id/userId/其它条件 |
| `@combine` 空格不合法 | `IllegalArgumentException(... 不允许首尾/连续空格 ...)` | `&`/`\|` 两侧各一空格；`!` 紧贴键名；括号内外无空格 |
| `@combine` 引用 id | `UnsupportedOperationException(... 不允许传 id, id{}, userId, userId{})` | id 强制 AND，不要写进表达式 |
| `@database` 非法 | `UnsupportedDataTypeException` | 取值须在 SQLConfig 常量列表内 |
| `@explain` 非 DEBUG | `UnsupportedOperationException` | 打开 `Log.DEBUG=true` |
| 括号不匹配 | `IllegalArgumentException(... 左括号 ( 比 右括号 ) 多/少 ...)` | 检查 `()` 配对 |
| 超出安全闸 | `IllegalArgumentException(... 已超过最大值 ...)` | 调整 `MAX_COMBINE_*` 或精简条件 |
| 条件值类型错 | `IllegalArgumentException(key:value 中 value 不合法...)` | 比较条件只接受 Boolean/Number/String/Subquery |
| POST 传了非字段 key | `IllegalArgumentException(... POST 请求中不允许传 xxx{})` | POST 只接受字段名（`isName` 校验） |
| NotLoggedIn | [NotLoggedInException](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/exception/NotLoggedInException.java) | 未登录访问需鉴权表 |
| NotExist | [NotExistException](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/exception/NotExistException.java) | id/userId 无效或被过滤空 |
| Conflict | [ConflictException](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/exception/ConflictException.java) | UNIQUE 冲突 |
| OutOfRange | [OutOfRangeException](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/exception/OutOfRangeException.java) | 分页/数量越界 |

### 14.2 调试技巧

1. 打开 `Log.DEBUG=true`，`@explain:true` 返回结构化 SQL、prepared values、JOIN 树。
2. `IS_PRINT_REQUEST_STRING_LOG` / `IS_PRINT_BIG_LOG` 打印完整请求响应。
3. DEBUG 模式下错误响应含 `trace:throw`、`trace:stack`（受 [IS_RETURN_STACK_TRACE](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractParser.java#L69) 控制）。
4. 自定义 [Callback.onMissingKey4Combine](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L6596) 可以让 combine 缺键时记录 warn 而非静默。

---

## 附录 A：一次完整请求的心智模型

```
{
  "[]": {                                # 顶层数组（一对多）
    "count": 10, "page": 0,
    "User": {                            # 主表对象
      "id{}": [1,2,3],                   # Step 4: id 强制 AND 前置
      "status": 1,                       # Step 8: WHERE
      "date>": "2024-01-01",             # §5 操作符
      "@column": "id,name,date",         # Step 10
      "@combine": "id{} & (status | date>)",  # §6 布尔表达式
      "Comment[]": {                     # 一对多子对象，由 ObjectParser 递归
        "Comment": { "momentId@": "User/id" }
      }
    }
  }
}
```

执行流：Parser 解析 `[]` → ObjectParser 解析 `User` → `setSQLConfig()` → `newSQLConfig()`（十步）→ `executeSQL()` → `response()` 递归到 `Comment[]`。

## 附录 B：相关文档

- [APIJSON 规划及路线图.md](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSON%20%E8%A7%84%E5%88%92%E5%8F%8A%E8%B7%AF%E7%BA%BF%E5%9B%BE.md)
- [Commit 规范.md](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/Commit%20%E8%A7%84%E8%8C%83.md)
- [README.md](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/README.md)
