# APIJSON ORM

> 腾讯开源的零代码 RESTful API 引擎 ORM 内核 — 用 JSON 描述查询，自动生成 SQL，零后端代码

![Version](https://img.shields.io/badge/version-8.2.0-blue)
![Java](https://img.shields.io/badge/Java-1.8%2B-orange)
![License](https://img.shields.io/badge/License-Apache%202.0-green)
![Database](https://img.shields.io/badge/Databases-30%2B-brightgreen)

---

## 目录

- [简介](#简介)
- [架构概览](#架构概览)
- [核心特性](#核心特性)
- [快速上手](#快速上手)
- [请求解析原理 (`newSQLConfig`)](#请求解析原理-newsqlconfig)
- [条件表达式引擎 (`@combine`)](#条件表达式引擎-combine)
- [项目结构](#项目结构)
- [支持的数据库](#支持的数据库)
- [文档](#文档)
- [许可证](#许可证)

---

## 简介

APIJSON 是一个基于 JSON 协议的 **零代码 ORM/API 引擎**。前端发送 JSON 描述"查什么、怎么查"，后端自动生成 SQL 执行并返回 JSON 结果，无需编写 Controller/Service/DAO。

**核心理念：** 前端（客户端）决定数据需求，后端提供通用引擎，消除接口开发与维护成本。

```
前端 JSON 请求  →  Parser 解析  →  newSQLConfig 生成 SQLConfig  →  SQL 执行  →  JSON 响应
```

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
                          │   (MySQL/PG/CK/... 30+)     │
                          └─────────────────────────────┘
```

### 核心类职责

| 类 | 职责 |
|----|------|
| [AbstractParser.java](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractParser.java) | 请求入口、事务、全局配置、递归调度 |
| [AbstractSQLConfig.java](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java) | SQL 配置生成、@combine 引擎、方言适配 |
| [AbstractObjectParser.java](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractObjectParser.java) | 对象/数组解析、关联查询、权限校验 |
| [AbstractSQLExecutor.java](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLExecutor.java) | SQL 执行、连接管理、缓存 |
| [AbstractVerifier.java](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractVerifier.java) | 权限验证、登录校验、角色、审计 |
| [Join.java](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/Join.java) | 12 种 JOIN 类型定义 |
| [Logic.java](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/Logic.java) | 逻辑运算符 (\|&!) |
| [Operation.java](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/Operation.java) | MUST/REFUSE/TYPE/VERIFY/INSERT/UPDATE... |

---

## 核心特性

- **零代码 API**：零后端代码，JSON 即接口，自动 CRUD
- **@combine 表达式引擎**：布尔表达式精确控制 WHERE/HAVING 条件组合
- **30+ 数据库**：MySQL, PostgreSQL, Oracle, SQL Server, ClickHouse, Elasticsearch, MongoDB, Redis, TiDB...
- **12 种 JOIN**：APP/LEFT/RIGHT/INNER/FULL/OUTER/ANTI/FOREIGN/ASOF/SIDE/CROSS
- **智能分页**：自动 count 总数、page/count 分页、防全表扫描
- **PreparedStatement**：预编译防注入，自动类型转换
- **假删除**：软删除自动转换 DELETE→UPDATE + 查询过滤
- **权限体系**：角色(UNKNOWN/LOGIN/CONTACT/CIRCLE/OWNER/ADMIN) + Access表控制
- **远程函数**：@function 调用服务端方法，支持 JSR223 脚本引擎
- **子查询**：EXISTS/IN/FROM 子查询
- **多数据源**：动态路由不同数据库/实例
- **SQL 缓存**：生成/执行统计、缓存命中计数

---

## 快速上手

### 查询示例

**请求：**
```json
{
  "Moment": {
    "@column": "id,userId,content,date",
    "@order": "date-",
    "@count": 10,
    "userId": 1,
    "date>": "2024-01-01",
    "content$": "hello"
  }
}
```

**自动生成 SQL：**
```sql
SELECT id, userId, content, date
FROM Moment
WHERE userId = 1 AND date > '2024-01-01' AND content LIKE '%hello%'
ORDER BY date DESC
LIMIT 10
```

**响应：**
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
    "tag<>": ["tech","music"],
    "@combine": "age> & (sex | name$) & tag<>"
  }
}
```

生成：
```sql
WHERE age > 18 AND (sex = 0 OR name LIKE '%a%') AND tag LIKE '%tech%'
```

### 连表查询

```json
{
  "Moment": {
    "id": 1,
    "User@": { "id@": "/Moment/userId" }
  }
}
```

---

## 请求解析原理 (`newSQLConfig`)

核心入口：[AbstractSQLConfig.newSQLConfig()](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L5477)

这是将请求 JSON 转化为 SQLConfig（进而生成 SQL）的核心方法，11 个阶段：

```
[1] 全局参数提取  → @explain/@database/@datasource/@schema...
[2] 创建 SQLConfig → 按 database 类型实例化对应子类
[3] 解析 JOIN      → parseJoin 处理关联表
[4] id/userId 处理 → 过滤无效值、类型校验、强制 AND 条件
[5] 关键词提取     → @column/@combine/@group/@having/@order/@key/@raw...
[6] @null/@cast   → NULL 条件 / CAST 类型转换
[7] POST/非POST分流
    ├─ POST: 字段收集 → columns + values (INSERT)
    └─ 非POST: combine解析 → WHERE条件 vs PUT的SET内容
[8] @column 处理   → 字段/函数/DISTINCT/@raw片段
[9] @having 处理   → 聚合函数条件 + 内部@combine
[10] @key 映射     → 字段表达式别名
[11] 还原request   → finally块put回已remove的key
```

**WHERE vs SET 分流规则** (PUT时)：
- 含功能符后缀的 key（`$`, `~`, `{}`, `>`, `<`...）→ WHERE 条件
- 在 @combine 表达式中引用的 key → WHERE 条件
- 其余 → SET 更新内容

---

## 条件表达式引擎 (`@combine`)

核心方法：[parseCombineExpression()](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L3364)

### 两种模式

**简单列表模式**（旧版兼容）：
```json
"@combine": "id{},&sex,!name$"
// → (id IN(..)) AND (sex=0) AND NOT (name LIKE '%a%')
```

**布尔表达式模式**（5.0+推荐）：
```json
"@combine": "date> | (contactIdList<> & (name*~ | tag&$))"
```

### 语法规范

| 元素 | 规则 | 示例 |
|------|------|------|
| AND | ` & `（两侧各一个空格） | `a & b` |
| OR | ` \| `（两侧各一个空格） | `a \| b` |
| NOT | `!` 紧跟 key/`(` | `!a`, `!(a \| b)` |
| 括号 | `(...)` 内侧无空格 | `(a & b) \| c` |
| 嵌套深度 | 最大 2 层 | `(a & (b \| c))` |

### 条件操作符

| key后缀 | SQL | 语义 |
|---------|-----|------|
| (无) | `=` | 等于 |
| `!` | `!=` | 不等于 |
| `$` | `LIKE` | 模糊匹配 |
| `~`/`*~` | `REGEXP` | 正则/忽略大小写 |
| `%` | `BETWEEN` | 区间 |
| `{}` | `IN`/条件链 | 集合/多条件 |
| `}{` | `EXISTS` | 子查询存在 |
| `<>` | JSON包含 | 数组包含 |
| `>`,`<`,`>=`,`<=` | 比较运算符 | 大小比较 |
| `key[` | `length(key)` | 字符串长度 |
| `key{` | `json_length(key)` | JSON长度 |

### 表达式解析状态机

逐字符扫描，关键状态：
- 遇 ` `（空格）→ 终结当前 key → 调用 `gainWhereItem()` 生成条件 SQL
- 遇 `&`/`|`（空格后）→ 拼接 AND/OR
- 遇 `!`（空格或`(`后）→ NOT 取反标记
- 遇 `(`/`)` → 深度计数，校验嵌套
- 结束后未引用的 WHERE 条件 → AND 追加到表达式之后

---

## 项目结构

```
APIJSONORM/
├── src/main/java/apijson/
│   ├── orm/
│   │   ├── AbstractParser.java          # 请求解析入口 (~6000行)
│   │   ├── AbstractSQLConfig.java       # SQL配置/生成/表达式引擎 (~6600行)
│   │   ├── AbstractObjectParser.java    # 对象/数组解析
│   │   ├── AbstractSQLExecutor.java     # SQL执行
│   │   ├── AbstractVerifier.java        # 权限校验
│   │   ├── AbstractFunctionParser.java  # 远程函数
│   │   ├── Logic.java                   # &|! 逻辑类型
│   │   ├── Join.java                    # 12种JOIN
│   │   ├── Operation.java               # MUST/REFUSE/TYPE/VERIFY...
│   │   ├── Subquery.java                # 子查询
│   │   ├── SQLConfig.java               # SQL配置接口
│   │   ├── SQLExecutor.java             # 执行器接口
│   │   ├── Parser.java                  # 解析器接口
│   │   ├── model/                       # 系统表模型
│   │   │   ├── Access.java, Request.java, Table.java,
│   │   │   ├── Column.java, Document.java, Function.java...
│   │   ├── script/                      # 脚本引擎
│   │   │   ├── ScriptExecutor.java
│   │   │   ├── JavaScriptExecutor.java
│   │   │   └── JSR223ScriptExecutor.java
│   │   └── exception/                   # 异常体系
│   ├── JSON.java, JSONMap.java, JSONRequest.java
│   ├── SQL.java                         # SQL关键字/工具函数
│   ├── RequestMethod.java               # GET/POST/PUT/DELETE/HEAD/CRUD
│   ├── StringUtil.java
│   └── Log.java
├── pom.xml                              # Maven 配置 (v8.2.0, JDK 1.8)
└── README.md
```

---

## 支持的数据库

| 类型 | 数据库 |
|------|--------|
| **关系型** | MySQL, MariaDB, PostgreSQL, SQL Server, Oracle, DB2, SQLite, DuckDB, TiDB, CockroachDB, Dameng(达梦), KingBase(人大金仓), OpenGauss |
| **分析型** | ClickHouse, Hive, Presto, Trino, Doris, StarRocks, Snowflake, Databricks, Databend |
| **NoSQL** | Elasticsearch, Manticore, MongoDB, Cassandra, Redis, Kafka, MQ |
| **时序** | InfluxDB, TDengine, TimescaleDB, QuestDB, IoTDB |
| **其他** | SurrealDB, Milvus(向量) |

---

## 文档

| 文档 | 说明 |
|------|------|
| [APIJSON 通用文档.md](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSON%20通用文档.md) | 完整使用文档：条件操作符、@combine语法、JOIN、子查询、权限、示例 |
| [APIJSON 规划及路线图.md](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSON%20规划及路线图.md) | 架构分析、newSQLConfig全流程、@combine引擎原理、技术债务、v8.x/v9.0 路线图 |
| [Commit 规范.md](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/Commit%20规范.md) | 基于 Conventional Commits 的提交规范，含 type/scope/body 完整示例 |

---

## 构建

```bash
cd APIJSONORM
mvn clean package -DskipTests
```

Java 1.8+，零外部依赖（纯 JDK 编写，JDBC 驱动由使用方引入）。

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
