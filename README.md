# APIJSON ORM · Squirtle

> 腾讯 [APIJSON](https://github.com/Tencent/APIJSON) 协议的 ORM 核心库（本仓库 fork 自 `Tencent/APIJSON` 8.2.0，代号 **Squirtle**）。
> 零 CRUD 后端：前端用一段 JSON 描述"要什么数据 + 什么条件"，后端自动翻译为参数化 SQL，返回结构化 JSON。

![logo](logo.png)

---

## 目录

- [这是什么](#这是什么)
- [架构速览](#架构速览)
- [请求解析主线：以 newSQLConfig 为中心](#请求解析主线以-newsqlconfig-为中心)
- [条件表达式引擎：@combine](#条件表达式引擎combine)
- [快速开始](#快速开始)
- [模块结构](#模块结构)
- [文档](#文档)
- [贡献](#贡献)
- [License](#license)

---

## 这是什么

APIJSON 是一个 **JSON 驱动**的 ORM/协议层：

- **零代码 CRUD**：单表/多表 JOIN、嵌套对象、分页、分组、聚合、子查询、软删除、远程函数都由 JSON 直接声明。
- **多数据源**：[SQLConfig](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/SQLConfig.java) 内置 40+ 种数据库常量（MySQL、PostgreSQL、Oracle、SQL Server、TiDB、ClickHouse、Doris、StarRocks、Dameng、KingBase、MongoDB、Redis、Kafka、Milvus、InfluxDB、TDengine、SQLite、DuckDB…）。
- **安全**：全量 PreparedStatement、列名/表名正则白名单、操作权限与字段级 `Operation` 校验、递归深度与分页阈值、写操作强制带条件。
- **可扩展**：通过 [Callback](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L6575-L6597) / [ParserCreator](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/ParserCreator.java) / [SQLCreator](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/SQLCreator.java) 注入自定义方言、ID 生成、鉴权与脚本引擎。

一个最简查询：

```http
GET /get
```
```json
{
  "User": {
    "id{}": [1, 2, 3],
    "@column": "id,name,date",
    "@combine": "id{} & (status | !deleted)"
  }
}
```

服务端会生成大致如下的参数化 SQL：

```sql
SELECT id, name, date FROM sys.User
WHERE id IN (?,?,?) AND ( (status = ?) OR (NOT(deleted = ?)) )
LIMIT 10 OFFSET 0
```

---

## 架构速览

```
HTTP JSON
   │
   ▼
Parser.parseResponse(M)            ── 顶层：format/version/tag/database/explain/cache
   │
   ▼
ObjectParser                       ── 每个表对象一个实例
   ├─ setSQLConfig()               ── 组装 SQLConfig
   │     └─ AbstractSQLConfig.newSQLConfig(...)   ◀── 请求解析核心（见下）
   │            ├─ parseJoin()     ── 副表/ON/OUTER 递归
   │            ├─ id/userId 强制 AND
   │            ├─ @combine → parseCombineExpression
   │            └─ @column/@having/@group/@order/@limit
   ├─ executeSQL()                 ── AbstractSQLExecutor.execute(config)
   └─ response() / 子对象递归
```

关键类型：

| 类型 | 职责 |
|------|------|
| [AbstractParser](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractParser.java) | 顶层入口、对象树递归、全局属性 |
| [AbstractObjectParser](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractObjectParser.java) | 单表生命周期：parse → setSQLConfig → executeSQL → response |
| [AbstractSQLConfig](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java) | SQL 元数据 + SQL 拼装 + `@combine` 引擎 |
| [AbstractSQLExecutor](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLExecutor.java) | JDBC 执行、缓存、事务、批处理 |
| [AbstractVerifier](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractVerifier.java) / [Operation](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/Operation.java) | 鉴权与 MUST/REFUSE/TYPE/VERIFY/EXIST/UNIQUE 校验 |
| [AbstractFunctionParser](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractFunctionParser.java) / [script 包](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/script) | 远程函数、JSR223 脚本 |

---

## 请求解析主线：以 newSQLConfig 为中心

[AbstractSQLConfig.newSQLConfig](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L5477-L6288) 是**单表 JSON 片段 → SQLConfig** 的静态工厂，按严格顺序执行 10 步：

1. **前置校验** — `@explain` 仅 DEBUG、`@database` 必须合法。
2. **实例化** — [Callback.getSQLConfig](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L6586) 按 database 返回方言实现。
3. **JOIN 处理** — [parseJoin](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L6300) 递归生成副表/ON/OUTER 子配置。
4. **id/userId 强制 AND 条件** — 过滤无效值（≤0、空串）；POST 时通过 [newId](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L6608) 生成。
5. **抽取 @ 关键字** — 从 request 中 `remove` 掉 `@role/@from/@column/@null/@cast/@combine/@group/@having/@order/@key/@raw/@json/@method` 等 20+ 个。
6. **@null/@cast 展开** — `@null:"tag"` → `tag IS NULL`；`@cast:"date:DATE"` 登记类型。
7. **POST 分支** — 剩余键全部是插入列，组装批量 `columns[]/values[]`。
8. **非 POST 分支** — 解析 `@combine`，键按 [isWhere](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L5800) 分流到 `tableWhere`（条件）或 `tableContent`（PUT SET）；条件键后缀由 [gainWhereItem](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L3897) 分派。
9. **软删除改写** — DELETE 命中 `ACCESS_FAKE_DELETE_MAP` 时改写成 PUT SET deleted=1。
10. **列/聚合/分页与还原** — `@column`（含 DISTINCT 与函数片段）、`@having/@having&`（与 `@combine` 同引擎）、最后把 `@` 关键字 `put` 回 request 供下游对象复用。

> 每个条件键后缀（`>`/`<`/`>=`/`<=`/`~`/`*~`/`$`/`%`/`{}`/`}{`/`<>`/`!`）都会被 `gainWhereItem` 分派到专门的 `gainXxxString` 方法生成参数化片段。完整对照表见 [APIJSON 通用文档 §5](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSON%20%E9%80%9A%E7%94%A8%E6%96%87%E6%A1%A3.md#5-条件操作符key-后缀)。

---

## 条件表达式引擎：@combine

`@combine` 有两种形态，最终都进入 [parseCombineExpression](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L3364-L3654) 字符级状态机：

- **逗号列表式**：`"status,&type,|category,!deleted"` —— 前缀 `&/|/!` 决定 AND/OR/NOT，无前缀默认 OR；PUT 禁用 `|/!`。
- **布尔表达式式（5.0+）**：`"id{} & (status | !deleted)"` —— `&`、`|` 两侧必须各一个空格；`!` 紧贴键名；`(` 右侧、`)` 左侧无空格；括号必须闭合。

引擎特性：

- 每个键从 `conditionMap` 取值生成条件片段 `wi`，再由二参重载 [gainCondition(isNot, wi)](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L4791-L4793) 处理：`isNot=true` 时返回 `NOT(wi)`，`false` 时原样返回 `wi`；外层 `( ... )` 由解析器在 [L3480](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L3480) 手动拼接；
- **未出现在表达式里的条件自动 AND 追加到尾部**，保证 id/userId 等强制条件一定生效；
- 同时驱动 `@having`/`@having&`；
- **安全闸**：[MAX_COMBINE_DEPTH=2、MAX_COMBINE_COUNT=5、MAX_COMBINE_KEY_COUNT=2、MAX_COMBINE_RATIO=1.0](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L61-L67)；
- `id`/`id{}`/`userId`/`userId{}` 禁止出现在表达式中，因为它们已经被强制 AND 前置。

更完整的语法规则、安全闸、示例与排错，见 [APIJSON 通用文档 §6](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSON%20%E9%80%9A%E7%94%A8%E6%96%87%E6%A1%A3.md#6-combine-条件表达式引擎)。

---

## 快速开始

### Maven

```xml
<repositories>
  <repository>
    <id>jitpack.io</id>
    <url>https://jitpack.io</url>
  </repository>
</repositories>

<dependency>
  <groupId>com.github.Tencent</groupId>
  <artifactId>APIJSON</artifactId>
  <version>8.2.0</version>
</dependency>
```

### Gradle

```gradle
allprojects { repositories { maven { url 'https://jitpack.io' } } }
dependencies { implementation 'com.github.Tencent:APIJSON:8.2.0' }
```

### 最小可运行骨架（伪代码）

```java
public class DemoSQLConfig extends AbstractSQLConfig<Long, JSONObject, JSONArray> {
    // 继承并实现 isMySQL/gainDataSource 等，或直接使用 Demo 中已有的 MySQL 实现
}

Parser<Long, JSONObject, JSONArray> parser = new DemoParser();
String response = parser.parse(
    "{\"User\":{\"id\":1,\"@column\":\"id,name\"}}"
);
System.out.println(response);
// {"User":{"id":1,"name":"tom"},"code":200,"msg":"success"}
```

> 完整可跑的 Spring Boot/JFinal 工程见官方 Demo（http://apijson.cn）。

---

## 模块结构

```
xyj-723-10_Squirtle/
├── APIJSONORM/                              # ORM 核心库（本仓库主体）
│   ├── src/main/java/apijson/
│   │   ├── JSON.java / JSONMap.java         # JSON 抽象 + @ 关键字常量
│   │   ├── JSONRequest/JSONResponse.java    # 请求/响应构造器
│   │   ├── RequestMethod.java               # GET/HEAD/GETS/HEADS/POST/PUT/DELETE/CRUD
│   │   ├── SQL.java                         # AND/OR/NOT/JOIN/count/sum/avg 等 SQL 片段
│   │   └── orm/
│   │       ├── AbstractParser.java          # 顶层解析器
│   │       ├── AbstractObjectParser.java    # 单表对象编排
│   │       ├── AbstractSQLConfig.java       # newSQLConfig 工厂 + @combine 引擎
│   │       ├── AbstractSQLExecutor.java     # JDBC 执行
│   │       ├── AbstractVerifier.java        # 鉴权
│   │       ├── AbstractFunctionParser.java  # 远程函数
│   │       ├── Join.java / Subquery.java    # JOIN 与子查询
│   │       ├── Operation.java               # MUST/REFUSE/TYPE/VERIFY/EXIST/UNIQUE
│   │       ├── SQLConfig.java               # 40+ 数据库常量
│   │       ├── Parser/ObjectParser/...      # 对外接口
│   │       ├── model/                       # Access/Request/Document/Table/Column…
│   │       ├── script/                      # JSR223 / JavaScript 执行器
│   │       └── exception/                   # CommonException/NotExistException/…
│   ├── pom.xml                              # GAV: com.github.Tencent:APIJSON:8.2.0
│   └── README.md
├── assets/                                  # 文档图片
├── APIJSON 规划及路线图.md                   # 版本规划与里程碑
├── APIJSON 通用文档.md                       # 协议、关键字、@combine、排错
├── Commit 规范.md                           # Conventional Commits 约定
├── LICENSE
└── README.md                                # 本文件
```

---

## 文档

| 文档 | 面向人群 | 内容 |
|------|----------|------|
| [APIJSON 通用文档.md](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSON%20%E9%80%9A%E7%94%A8%E6%96%87%E6%A1%A3.md) | 使用者 / 接入方 | 14 章：架构、十步请求解析、`@` 关键字全表、条件操作符、`@combine`、JOIN、分组聚合、远程函数、安全、方言、扩展点、配置项、排错 |
| [APIJSON 规划及路线图.md](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSON%20%E8%A7%84%E5%88%92%E5%8F%8A%E8%B7%AF%E7%BA%BF%E5%9B%BE.md) | 维护者 / 贡献者 | As-Is 评估、To-Be 目标、v8.2→v9.0 版本线、里程碑、风险与对策 |
| [Commit 规范.md](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/Commit%20%E8%A7%84%E8%8C%83.md) | 贡献者 | Conventional Commits + 本项目 scope 白名单、模板、自动化校验 |
| [APIJSONORM/README.md](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/README.md) | 使用者 | Maven/Gradle 依赖坐标 |

**在线资源**

- 官网：http://apijson.cn
- 在线单元测试：http://apijson.cn/unit
- 上游仓库：https://github.com/Tencent/APIJSON
- DeepWiki：https://deepwiki.com/Tencent/APIJSON

---

## 贡献

1. Fork 并从 `master` 切出分支：`feat/<scope>-<short>` 或 `fix/<scope>-<short>`。
2. 阅读 [Commit 规范.md](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/Commit%20%E8%A7%84%E8%8C%83.md)，保持 commit message 合规。
3. 改动以下核心路径**必须**附带说明或测试：
   - [newSQLConfig](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L5477) 请求解析
   - [parseCombineExpression](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L3364) 条件表达式
   - [parseJoin](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L6300) JOIN
   - 安全闸常量（[MAX_COMBINE_*](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L61-L67)）
4. 新数据库方言必须实现 `AbstractSQLConfig` 子类并补充 Testcontainer 用例。
5. 提 PR 前确保 `mvn -pl APIJSONORM clean package` 通过。

详细路线与贡献流程见 [APIJSON 规划及路线图.md §6](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSON%20%E8%A7%84%E5%88%92%E5%8F%8A%E8%B7%AF%E7%BA%BF%E5%9B%BE.md#6-贡献者指引)。

---

## License

Apache License 2.0，Copyright (C) 2020 Tencent。详见 [LICENSE](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/LICENSE)。
