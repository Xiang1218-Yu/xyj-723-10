# APIJSON ORM

> 腾讯开源的零代码 RESTful API 引擎 ORM 内核 — 用 JSON 描述查询，自动生成 SQL，零后端代码

![Version](https://img.shields.io/badge/version-8.2.0-blue)
![Java](https://img.shields.io/badge/Java-1.8%2B-orange)
![License](https://img.shields.io/badge/License-Apache%202.0-green)
![Database](https://img.shields.io/badge/Databases-35-brightgreen)

---

## 目录

- [简介](#简介)
- [架构概览](#架构概览)
- [核心特性](#核心特性)
- [快速上手](#快速上手)
- [请求解析原理 (newSQLConfig)](#请求解析原理-newsqlconfig)
- [条件表达式引擎 (@combine)](#条件表达式引擎-combine)
- [项目结构](#项目结构)
- [支持的数据库](#支持的数据库)
- [文档](#文档)
- [许可证](#许可证)

---

## 简介

APIJSON 是一个基于 JSON 协议的 **零代码 ORM/API 引擎**。前端发送 JSON 描述"查什么、怎么查"，后端自动生成 SQL 执行并返回 JSON 结果，无需编写 Controller/Service/DAO。

**核心理念：** [推断] 前端（客户端）决定数据需求，后端提供通用引擎，消除接口开发与维护成本。

```
前端 JSON 请求  →  Parser 解析  →  newSQLConfig 生成 SQLConfig  →  SQL 执行  →  JSON 响应
```

> `[源码]` 表示源码直接可验证的事实（附行号）；`[推断]` 表示基于源码逻辑推导的结论；`[示例]` 表示用法示例。

---

## 架构概览

```
                          ┌─────────────────────────────┐
                          │       HTTP Request           │
                          │   (JSON: { "User":{...} })   │
                          └──────────────┬──────────────┘
                                         │
                                         ▼
                          ┌─────────────────────────────┐
                          │     AbstractParser           │
                          │  ┌───────────────────────┐   │
                          │  │ parseResponse()       │   │
                          │  │ - 全局配置提取         │   │
                          │  │ - 登录/角色校验        │   │
                          │  │ - 事务管理            │   │
                          │  └──────────┬────────────┘   │
                          │             │ onObjectParse() │
                          │             ▼                │
                          │  ┌───────────────────────┐   │
                          │  │ AbstractObjectParser   │   │
                          │  │ - 递归对象/数组解析    │   │
                          │  │ - JOIN 解析           │   │
                          │  └──────────┬────────────┘   │
                          └─────────────┼───────────────┘
                                        │
                                        ▼
                          ┌─────────────────────────────┐
                          │   AbstractSQLConfig          │
                          │  ┌───────────────────────┐   │
                          │  │ newSQLConfig()        │   │
                          │  │ - 关键词提取(@xxx)     │   │
                          │  │ - WHERE/CONTENT分流   │   │
                          │  │ - @combine表达式解析  │   │
                          │  │ - SQL语句生成         │   │
                          │  └──────────┬────────────┘   │
                          └─────────────┼───────────────┘
                                        │
                                        ▼
                          ┌─────────────────────────────┐
                          │   AbstractSQLExecutor        │
                          │  ┌───────────────────────┐   │
                          │  │ - PreparedStatement   │   │
                          │  │ - 缓存/结果映射       │   │
                          │  │ - 多数据源路由        │   │
                          │  └──────────┬────────────┘   │
                          └─────────────┼───────────────┘
                                        │
                                        ▼
                          ┌─────────────────────────────┐
                          │        Database              │
                          │   (MySQL/PG/CK/... 35种)    │
                          └─────────────────────────────┘
```

### 核心类职责

| 类 | 职责 |
|----|------|
| [AbstractParser.java](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractParser.java) | `[源码]` 请求入口、事务、全局配置、递归调度 |
| [AbstractSQLConfig.java](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java) | `[源码]` SQL 配置生成、@combine 引擎、方言适配（6600+ 行） |
| [AbstractObjectParser.java](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractObjectParser.java) | `[源码]` 对象/数组解析、关联查询、权限校验 |
| [AbstractSQLExecutor.java](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLExecutor.java) | `[源码]` SQL 执行、连接管理、缓存 |
| [AbstractVerifier.java](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractVerifier.java) | `[源码]` 权限验证、6种角色（UNKNOWN/LOGIN/CONTACT/CIRCLE/OWNER/ADMIN，L76-L96） |
| [Join.java](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/Join.java#L21) | `[源码]` 12 种 JOIN 类型定义 |
| [Logic.java](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/Logic.java#L15-L22) | `[源码]` 逻辑运算符 `&`(AND)/`|`(OR)/`!`(NOT) |
| [Operation.java](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/Operation.java) | `[源码]` 13种操作：MUST/REFUSE/TYPE/VERIFY/EXIST/UNIQUE/INSERT/UPDATE/REPLACE/REMOVE/IF/ALLOW_PARTIAL_UPDATE_FAIL/IS_ID_CONDITION_MUST |

---

## 核心特性

- **零代码 API** `[推断]`：零后端代码，JSON 即接口，自动 CRUD
- **@combine 表达式引擎** `[源码]`：布尔表达式精确控制 WHERE/HAVING 条件组合，[parseCombineExpression()](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L3364)
- **35 种数据库白名单** `[源码]`：DATABASE_LIST 包含 MySQL/PostgreSQL/Oracle/SQL Server/ClickHouse/Elasticsearch/MongoDB/Redis/TiDB 等共 35 种（[AbstractSQLConfig.java#L146-L181](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L146-L181)）
- **12 种 JOIN** `[源码]`：APP/LEFT/RIGHT/CROSS/INNER/FULL/OUTER/SIDE/ANTI/FOREIGN/ASOF + 空字符串默认FULL（[Join.java#L21](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/Join.java#L21)，[AbstractSQLConfig.java#L3782](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L3782)）
- **PreparedStatement** `[推断]`：预编译防注入，ordered preparedValueList 自动类型转换
- **假删除** `[源码]`：软删除自动转换 DELETE→PUT + 查询过滤（ACCESS_FAKE_DELETE_MAP，[AbstractSQLConfig.java#L5835-L5881](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L5835-L5881)）
- **权限体系** `[源码]`：6种角色 + Access表控制（[AbstractVerifier.java#L76-L96](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractVerifier.java#L76-L96)）
- **8 种请求方法** `[源码]`：GET/HEAD/GETS/HEADS/POST/PUT/DELETE/CRUD（[RequestMethod.java#L14-L59](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/RequestMethod.java#L14-L59)）
- **安全限制** `[源码]`：MAX_HAVING_COUNT=5, MAX_WHERE_COUNT=10, MAX_COMBINE_DEPTH=2, MAX_COMBINE_COUNT=5（[AbstractSQLConfig.java#L61-L67](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L61-L67)）
- **远程函数** `[推断]`：@function 调用服务端方法，支持 JSR223 脚本引擎
- **多数据源** `[推断]`：动态路由不同数据库/实例

---

## 快速上手

### 查询示例

**请求：** `[示例]`
```json
{
  "Moment": {
    "@column": "id,userId,content,date",
    "@order": "date-",
    "userId": 1,
    "date>": "2024-01-01",
    "content$": "hello"
  }
}
```

**自动生成 SQL：** `[示例]`
```sql
SELECT id, userId, content, date
FROM Moment
WHERE userId = 1 AND date > '2024-01-01' AND content LIKE '%hello%'
ORDER BY date DESC
```

> `[源码]` `content$` → LIKE（keyType=1，[AbstractSQLConfig.java#L3914-L3915](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L3914-L3915)）；`date>` → `>` 比较（keyType=9，L3938-L3940）；`@order: "date-"` 中 `-` 后缀表示 DESC。`@count`/`@page` 是 Parser 层分页参数（由 AbstractParser 处理，不在 newSQLConfig 关键词列表中）。

**响应：** `[示例]`
```json
{
  "Moment": [{ "id": 1, "userId": 1, "content": "...", "date": "2024-06-01" }],
  "ok": true,
  "total": 42
}
```

### @combine 复杂条件

```json
{
  "User": {
    "age>": 18,
    "sex": 0,
    "name$": "a",
    "@combine": "age> & (sex | name$)"
  }
}
```

`[示例]` 生成：
```sql
WHERE age > 18 AND (sex = 0 OR name LIKE '%a%')
```

> `[源码]` `&` = AND（两侧有空格），`|` = OR（两侧有空格）；`sex` 无后缀 → `=`（keyType=0，L3943-L3944）。

### 连表查询 `[示例]`

```json
{
  "Moment": {
    "id": 1,
    "User@": { "id@": "/Moment/userId" }
  }
}
```

> `[源码]` `@` 后缀 = APP JOIN（应用层关联）；`/Moment/userId` 是引用路径，指向父表的 userId 字段。

---

## 请求解析原理 (`newSQLConfig`)

核心入口：[AbstractSQLConfig.newSQLConfig()](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L5477)（L5477-L6288，共811行）

这是将请求 JSON 转化为 SQLConfig（进而生成 SQL）的核心方法，`[源码]` 19 个处理阶段：

```
[1]  @explain 校验          L5484  非DEBUG模式禁止 @explain
[2]  @database 提取与校验    L5489  DATABASE_LIST 白名单校验（35种）
[3]  连接参数提取            L5495  @datasource/@namespace/@catalog/@schema
[4]  创建 SQLConfig 实例     L5500  callback.getSQLConfig 按数据库类型子类化
[5]  存储过程短路            L5509  isProcedure 直接返回
[6]  parseJoin 关联表解析    L5513  递归 newSQLConfig 处理 JOIN 副表
[7]  空请求短路              L5515  request.isEmpty() 直接返回
[8]  id/id{} 校验            L5521  无效值过滤、类型校验、DELETE/PUT 强制 count=1
[9]  userId/userId{} 校验    L5588  同上逻辑处理用户标识字段
[10] 关键词批量提取          L5639  @role/@cache/@from/@column/@null/@cast/@combine/
                              @group/@having/@having&/@sample/@latest/@partition/
                              @fill/@order/@key/@raw/@json/@method 共20个
[11] try: remove 条件/关键词 L5661  从request中remove已提取的key
[12] @null 处理              L5693  NULL条件key注入request
[13] @cast 处理              L5710  CAST类型转换map构建
[14] @raw 设置               L5737  原始SQL关键词列表
[15] POST 分支：字段收集      L5750  纯字段名校验 → columns + values (INSERT)
[16] 非POST分支：条件分流     L5799  combine解析（简单模式/布尔表达式模式）
                              WHERE vs SET分流（PUT时）：
                              L5963: isWhere || !isName(key) || key in combineExpr
                              → WHERE; 其余 → SET
                              假删除条件注入(L5835-L5881)
                              combineMap固定顺序 &→|→! (L5977-L5979)
                              PUT禁止 |key/!key (L5901-L5912)
[17] DELETE假删除转换         L5987  转为PUT + 补充deletedKey字段
[18] @column 解析            L6010  DISTINCT/函数/字段列表解析
[19] @having/@having& 解析   L6058  聚合函数条件 + 内部@combine
    + @key映射               L6150  字段表达式别名
    + config属性批量设置      L6172
    finally: 还原request     L6200  put回所有已remove的key
```

**WHERE vs SET 分流规则** `[源码]`（PUT 时，[L5963-L5973](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L5963-L5973)）：
- `isWhere || !isName(key.replaceFirst("[+-]$",""))` → WHERE 条件
- `key` 在 @combine 表达式中被引用 → WHERE 条件
- `key` 在 combineMap 的 whereList 中 → WHERE 条件
- 其余 → SET 更新内容

---

## 条件表达式引擎 (`@combine`)

核心方法：[parseCombineExpression()](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L3364)（L3364-L3654，逐字符状态机）

### 两种模式

**简单列表模式**（旧版兼容，无逗号时）`[源码]`：
```json
"@combine": "id{},&sex,!name$"
// → (id IN(..)) AND (sex=0) AND NOT (name LIKE '%a%')
```
> `[源码]` 以逗号分割，`&`前缀→AND组、`|`前缀→OR组、`!`前缀→NOT组、无前缀→OR组（[L5892-L5943](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L5892-L5943)）。PUT请求禁止`|key`/`!key`。

**布尔表达式模式**（5.0+推荐，含逗号时为表达式）`[源码]`：
```json
"@combine": "date> | (contactIdList<> & (name*~ | tag$))"
```
> `[源码]` 模式判断：`String[] ws = StringUtil.split(combine);` 当 `ws.length == 1` 且无逗号时为表达式模式（[L5803-L5804](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L5803-L5804)）。

### 语法规范

| 元素 | 规则 `[源码]` | 示例 |
|------|------|------|
| AND | ` & `（两侧各一个空格） | `a & b` |
| OR | ` \| `（两侧各一个空格） | `a \| b` |
| NOT | `!` 紧跟 key/`(` | `!a`, `!(a \| b)` |
| 括号 | `(...)` 内侧无空格 | `(a & b) \| c` |
| 嵌套深度 | 最大 2 层（MAX_COMBINE_DEPTH=2） | `(a & (b \| c))` |
| 键数量上限 | MAX_COMBINE_KEY_COUNT=2（简单模式） | — |
| 总条件上限 | MAX_COMBINE_COUNT=5 | — |

### 条件操作符 `[源码]`

keyType 检测顺序（[gainWhereItem()](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L3913-L3945) L3913-L3945）：

| key后缀 | keyType | SQL | 语义 |
|---------|---------|-----|------|
| (无) | 0 | `=` / `IS NULL` | 等于/空值 |
| `!` | 0(特殊) | `!=` / `IS NOT NULL` | 不等于/非空 |
| `$` | 1 | `LIKE '%x%'` | 模糊匹配 |
| `~` | 2 | `REGEXP` | 正则匹配 |
| `*~` | -2 | 忽略大小写正则 | 数据库相关 |
| `%` | 3 | `BETWEEN` | 区间 |
| `{}` | 4 | `IN`/条件链 | 集合/多条件 |
| `}{` | 5 | `EXISTS` | 子查询存在 |
| `<>` | 6 | `JSON_CONTAINS` | JSON数组包含 |
| `>=` | 7 | `>=` | 大于等于 |
| `<=` | 8 | `<=` | 小于等于 |
| `>` | 9 | `>` | 大于 |
| `<` | 10 | `<` | 小于 |
| `key[` | — | `length(key)`/`datalength(key)` | 字符串长度（SQL Server用datalength） |
| `key{` | — | `json_length(key)` | JSON长度 |

> `[源码]` `key!` 不等于由 [gainEqualString()](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L3985) L3985 处理：`column.endsWith("!")` → `!=` / `IS NOT`。长度前缀 `key[`/`key{` 由 [gainKey()](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L4014-L4023) L4014-L4023 处理。

### 表达式解析状态机 `[源码]`

逐字符扫描（[parseCombineExpression()](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L3364)），关键状态：
- 遇空格 → 终结当前 key → 调用 `gainWhereItem()` 生成条件 SQL
- 遇 `&`/`|`（空格后）→ 拼接 AND/OR
- 遇 `!`（空格或`(`后）→ NOT 取反标记
- 遇 `(`/`)` → 深度计数，校验嵌套深度 ≤ 2
- 未在表达式中引用的 WHERE 条件 → AND 追加到表达式之后

---

## 项目结构

```
APIJSONORM/
├── src/main/java/apijson/
│   ├── orm/
│   │   ├── AbstractParser.java          # [源码] 请求解析入口
│   │   ├── AbstractSQLConfig.java       # [源码] SQL配置/生成/@combine引擎（~6600行）
│   │   ├── AbstractObjectParser.java    # [源码] 对象/数组解析、JOIN
│   │   ├── AbstractSQLExecutor.java     # [源码] SQL执行
│   │   ├── AbstractVerifier.java        # [源码] 权限校验、6种角色
│   │   ├── AbstractFunctionParser.java  # [源码] 远程函数
│   │   ├── Logic.java                   # [源码] &/|/! 逻辑类型（L15-L22）
│   │   ├── Join.java                    # [源码] 12种JOIN类型（L21）
│   │   ├── Operation.java               # [源码] 13种操作枚举
│   │   ├── Subquery.java                # [源码] 子查询
│   │   ├── SQLConfig.java               # [源码] SQL配置接口、36个DATABASE_常量（L20-L57）
│   │   ├── SQLExecutor.java             # [源码] 执行器接口
│   │   ├── Parser.java                  # [源码] 解析器接口
│   │   ├── model/                       # [源码] 系统表模型
│   │   │   ├── Access.java, Request.java, Table.java,
│   │   │   ├── Column.java, Document.java, Function.java...
│   │   ├── script/                      # [源码] 脚本引擎
│   │   │   ├── ScriptExecutor.java
│   │   │   ├── JavaScriptExecutor.java
│   │   │   └── JSR223ScriptExecutor.java
│   │   └── exception/                   # [推断] 异常体系
│   ├── JSON.java, JSONMap.java, JSONRequest.java
│   ├── SQL.java                         # [源码] SQL关键字/工具函数
│   ├── RequestMethod.java               # [源码] 8种方法: GET/HEAD/GETS/HEADS/POST/PUT/DELETE/CRUD（L14-L59）
│   ├── StringUtil.java
│   └── Log.java
├── pom.xml                              # [源码] Maven 配置 (v8.2.0, JDK 1.8)
└── README.md
```

> `[源码]` [JSONMap.java#L207-L242](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/JSONMap.java#L207-L242) TABLE_KEY_LIST 定义了 34 个进入 newSQLConfig 的关键词（@try/@catch/@drop/@default 由 Parser 层处理，不在此列表）。

---

## 支持的数据库

`[源码]` DATABASE_LIST 白名单共 35 种（[AbstractSQLConfig.java#L146-L181](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L146-L181)）：

| 类型 | 数据库 |
|------|--------|
| **关系型** | MySQL, MariaDB, PostgreSQL, SQL Server, Oracle, DB2, TiDB, CockroachDB, Dameng(达梦), KingBase(人大金仓), OpenGauss, DuckDB |
| **分析型** | ClickHouse, Hive, Presto, Trino, Doris, StarRocks, Snowflake, Databricks, Databend |
| **NoSQL** | Elasticsearch, Manticore, MongoDB, Cassandra, Redis, Kafka, MQ |
| **时序** | InfluxDB, TDengine, TimescaleDB, QuestDB, IoTDB |
| **向量/其他** | Milvus, SurrealDB |

> `[源码]` 注意：[SQLConfig.java#L51](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/SQLConfig.java#L51) 定义了 `DATABASE_SQLITE` 常量，但 SQLite **未加入** DATABASE_LIST 白名单（不可通过 `@database` 直接指定）。

---

## 文档

| 文档 | 说明 |
|------|------|
| [APIJSON 通用文档.md](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSON%20通用文档.md) | 完整使用文档：条件操作符、@combine语法、JOIN、子查询、权限、示例 |
| [APIJSON 规划及路线图.md](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSON%20规划及路线图.md) | 架构分析、newSQLConfig全流程、@combine引擎原理、技术债务、v8.x/v9.0 路线图 |
| [Commit 规范.md](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/Commit%20规范.md) | 基于 Conventional Commits 的提交规范 |

---

## 构建

```bash
cd APIJSONORM
mvn clean package -DskipTests
```

`[源码]` Java 1.8+，零外部依赖（纯 JDK 编写，JDBC 驱动由使用方引入）。

```xml
<dependency>
    <groupId>com.github.Tencent</groupId>
    <artifactId>APIJSON</artifactId>
    <version>8.2.0</version>
</dependency>
```

---

## 许可证

[Apache License 2.0](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/LICENSE)

Copyright (C) 2020 Tencent.
