# APIJSON 规划及路线图

> 基于对 [AbstractSQLConfig.newSQLConfig](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L5477-L6288) 请求解析主链路与 [parseCombineExpression](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L3364-L3654) 条件表达式引擎的源码级分析制定。

---

## 1. 现状评估（As-Is）

### 1.1 模块结构

| 层 | 关键类型 | 职责 |
|----|----------|------|
| 入口 | [AbstractParser.parseResponse](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractParser.java#L505) | 接收 JSON 字符串/Map，分发到对象解析器 |
| 编排 | [AbstractObjectParser](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractObjectParser.java) | setSQLConfig → executeSQL → response，含子对象递归与函数处理 |
| 配置 | [AbstractSQLConfig](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java) | `newSQLConfig` 静态工厂 + SQL 拼装 + `@combine` 引擎 |
| 执行 | [AbstractSQLExecutor](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLExecutor.java) | JDBC 执行、缓存、事务、批处理 |
| 鉴权 | [AbstractVerifier](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractVerifier.java) | 角色、权限、[Operation](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/Operation.java) 校验 |
| 基础 | [apijson.JSON](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/JSON.java) / [JSONMap](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/JSONMap.java) | JSON 抽象 + 关键字常量（`@combine`、`@having` 等） |

### 1.2 newSQLConfig 主链路（请求解析全貌）

`newSQLConfig` 在 [AbstractSQLConfig.java:5477](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L5477) 是单表 JSON 片段 → SQLConfig 的工厂方法，顺序如下：

1. **前置校验**：`@explain` 仅 DEBUG 允许；`@database` 必须命中 [DATABASE_LIST](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L98)。
2. **实例化**：通过 [Callback.getSQLConfig](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L6586) 按数据库类型返回具体 SQLConfig（MySQL/PG/Oracle/ClickHouse/Doris/MongoDB 等 40+ 种）。
3. **JOIN 处理**：[parseJoin](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L6300) 递归 `newSQLConfig` 生成副表与 ON/OUTER 子配置。
4. **主键/用户 ID 强制 AND 条件**：`id`、`id{}`、`userId`、`userId{}` 先过滤无效值（≤0、空串），POST 时通过 [newId](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L6608) 生成时间戳递增 ID（默认实现为 `System.currentTimeMillis()`，同毫秒内自增避免冲突；非雪花算法）。
5. **关键字抽取**：从 request 中 `remove` 掉 `@role/@cache/@from/@column/@null/@cast/@combine/@group/@having/@order/@key/@raw/@json/@method` 等 20+ 个 `@` 开头关键字，分别存入 config 字段。
6. **`@null`/`@cast` 展开**：`@null:"tag"` → `request.put("tag", null)`（IS NULL）；`@cast:"date:DATE"` 登记类型转换。
7. **POST 分支**：剩余键全部视为插入列，组装 `columns[]`/`values[]` 批量 INSERT。
8. **非 POST 分支**：
   - 解析 `@combine`（见第 1.3 节），构建 `andList/orList/notList`；
   - 遍历 request 键值，按 [isWhere](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L5800) 分入 `tableWhere`（条件）或 `tableContent`（PUT SET 内容）；
   - 条件键后缀（`$`/`~`/`%`/`{}`/`}{`/`<>`/`>=`/`<=`/`>`/`<`/`!`）由 [gainWhereItem](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L3897) 分派到 `gainSearchString/gainRegExpString/gainBetweenString/gainRangeString/gainExistsString/gainContainString/gainCompareString/gainEqualString`。
9. **软删除**：DELETE 命中 ACCESS 假删除配置时，自动改写成 PUT SET deleted=1，并叠加 `deletedTime`。
10. **@column/@having/@group/@order/@limit**：`@column` 支持 `DISTINCT` 前缀与 `fun(key)` 函数片段；`@having` 默认 OR、`@having&` AND；最后把所有 `@` 关键字 put 回 request 供下游对象复用。
    > 注："写操作必须带条件"的硬校验**不在** `newSQLConfig` 内，而是在 [gainSQL](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L4932) 拼装 SQL 时调用 [getWhereString](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L3337-L3346)（[L3344-L3346](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L3344-L3346)）对非查询方法（PUT/DELETE/CRUD 子语句）触发 `UnsupportedOperationException("写操作请求必须带条件！！！")`；POST 单条 INSERT 因在 Step 7 提前 `return` 且 INSERT 无 WHERE 子句，不经过此校验。

### 1.3 @combine 条件表达式引擎

`@combine` 有两种形态，均在 `newSQLConfig` 中预处理，最终由 [parseCombineExpression](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L3364) 消费：

- **逗号列表式**（兼容旧版/PUT）：`"a,&b,|c,!d"` → `&/|/!` 前缀决定 `andList/orList/notList`，无前缀默认 OR。PUT 禁止 `|`/`!`。
- **布尔表达式式**（5.0+）：`"a & (b | !c)"` —— 字符级状态机解析，语法严格：
  - `&`、`|` 两侧必须各一个空格；`!` 紧贴键名、不接空格；`(` 右侧与 `)` 左侧不允许空格；不允许首尾/连续空格。
  - 每个键名从 `conditionMap` 取值后用 `gainWhereItem` 生成片段 `wi`，再调用二参重载 [gainCondition(isNot, wi)](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L4791-L4793)（该重载内部委托三参版且 `addOuterBracket=false`）；`isNot=true` 时返回 `NOT(wi)`，否则原样返回 `wi`。外层的 `( ... )` 由 [parseCombineExpression L3480](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L3480) 通过字符串拼接 `"( " + ... + " )"` 手动加上，最终 AND/OR/NOT 串接。
  - 未出现在表达式里的条件会以 AND 追加到尾部（保证 prepared value 顺序正确）。
  - **安全闸**：[MAX_COMBINE_DEPTH=2 / MAX_COMBINE_COUNT=5 / MAX_COMBINE_KEY_COUNT=2 / MAX_COMBINE_RATIO=1.0](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L63-L67)，同引擎复用于 `@having`。

### 1.4 关键约束与短板

| 类别 | 现状 | 风险 |
|------|------|------|
| 工程化 | Java 8、无外部依赖、纯 ORM 包 | 缺官方 starter、缺可观测性 |
| 安全 | `ALLOW_MISSING_KEY_4_COMBINE=true` 默认放行 | 误用可能漏条件 |
| 性能 | prepared value 列表在 combine 解析时被反向重建 | JOIN/子查询深链有顺序坑 |
| 扩展 | 新增数据库需继承 `AbstractSQLConfig` 重写数十个 `gainXxxString` | 接入成本高 |
| 可测试性 | 依赖外部 Demo 工程，ORM 包内无单元测试 | 回归防护弱 |
| 文档 | 仅 ORM README 有依赖坐标 | 架构/关键字/安全/扩展文档缺失 |

---

## 2. 目标（To-Be）

围绕 **"零 CRUD 后端、安全可观测、多数据源一体化"** 三个方向：

1. **协议稳定**：`@combine`/`@having` 语法语义与安全闸 SLA 化，破坏性改动走 major version。
2. **能力完备**：补齐窗口函数、CTE（开关 [ENABLE_WITH_AS](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L45) 已存在，`isWithAsEnable()` 仅对 MySQL 做了版本号 ≥8 校验，对 PG/Oracle 等无版本门槛需各方言自行验证）、JSON 字段函数；时序/向量库（Milvus、InfluxDB、TDengine、IoTDB、QuestDB）在 [DATABASE_LIST](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L163-L168) 已注册并在部分分支（如 LIMIT、标识符引用，见 [L1259-L1303](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L1259-L1303)、[L3051-L3058](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L3051-L3058)）有处理，需补齐完整方言实现与 TCK。
3. **工程易用**：提供 Spring Boot Starter、可观测性（Micrometer/OpenTelemetry）、GraalVM native-image。
4. **安全加固**：默认拒绝无键 `@combine`、写操作必带条件的审计、SQL 注入正则升级。
5. **可观测/可测试**：`@explain` 输出执行计划，内置 JUnit5 测试套件与 TCK（Technology Compatibility Kit）。

---

## 3. 版本路线图

> 版本号遵循 SemVer。当前 [pom.xml](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/pom.xml#L8) 为 `8.2.0`（基线日 2026-07-24）。下列日期为目标季度/月份，按"季度发布、补丁滚动"节奏推进。

### 3.1 v8.2.x（当前维护分支，持续滚动）

定位：`8.2.0` 之后的补丁/小改线，不做破坏性变更，按 `8.2.1`/`8.2.2`…滚动发布，无固定截止日期。首个补丁计划在基线日后 2–4 周内。

- [ ] 将 `newSQLConfig` 中 `id/userId` 处理抽为 `IdentityPolicy` 策略接口，默认保留 [SimpleCallback.newId](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L6608)，允许注入雪花/UUID/数据库自增。
- [ ] `@combine` 表达式解析增加单测与错误码枚举（替代散落的 `IllegalArgumentException` 文本）。
- [ ] 修复 prepared value 在 `JOIN ON + @combine` 场景的顺序耦合点（见 [setPreparedValueList(new ArrayList<>())](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L3398)）。
- [ ] 静态常量（MAX_COMBINE_*、IGNORE_*）改为可通过 `apijson.properties` 外部化配置。

### 3.2 v8.3（2026 Q4）— 可观测性 & 测试

- [ ] 引入 `apijson-micrometer`：`parser.parse`/`sql.execute`/`combine.parse` 埋点 Timer + Counter。
- [ ] 在 ORM 包内新增 `src/test`：针对 `newSQLConfig` 与 `parseCombineExpression` 的参数化测试（JUnit5），覆盖全部条件后缀与安全闸。
- [ ] `@explain` 返回结构化 JSON（SQL、prepared values、join 树、combine AST），DEBUG 与非 DEBUG 可控字段。
- [ ] 发布官方 Spring Boot 3 Starter（`apijson-spring-boot-starter`），自动装配 `Parser/Verifier/SQLExecutor`。

### 3.3 v8.4（2027 Q1）— SQL 能力增强

- [ ] 默认开启 `ENABLE_WITH_AS`：当前 [isWithAsEnable()](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L5054-L5057) 对非 MySQL 数据库无版本门槛（开启即返回 true），需先为 PG/Oracle 等补 CTE 兼容性校验，再切换默认值。
- [ ] 新增 `@window` 关键字：`"@window":"row_number() over(partition by userId order by date desc)"`，`@column` 可引用别名。
- [ ] `@combine` 支持集合谓词：`"tags{} &| any(...)"` 桥接 JSON 字段；为 JSON 类型字段统一 `json_contains` 方言。
- [ ] 完成 Milvus/InfluxDB/TDengine/IoTDB/QuestDB 等时序与向量库的 `AbstractSQLConfig` 实现并加 TCK。

### 3.4 v9.0（2027 Q2–Q3）— 架构升级

- [ ] **模块化**（JPMS）拆分：把当前自研的 [JSONMap](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/JSONMap.java)/[JSONList](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/JSONList.java)（无外部 JSON 依赖，[pom.xml](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/pom.xml#L23-L24) `<dependencies/>` 为空）抽到 `apijson-core`，并预留 `JSONProvider` SPI 以便后续可替换为 fastjson2/jackson 等实现，解耦 [JSONCreator](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/JSONCreator.java)。
- [ ] **编译期安全**：提供 OpenAPI/JSON Schema 导出，前端可按角色拉取可用结构。
- [ ] **Reactive 执行器**：新增 `AbstractSQLExecutor` 的 R2DBC 实现，Parser 链路保持同步兼容。
- [ ] **多租户/行级安全**：在 `newSQLConfig` 注入租户策略 SPI，自动追加 `tenantId` AND 条件，绕过 [ALLOW_MISSING_KEY_4_COMBINE](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L67) 带来的风险。
- [ ] 默认值加固：`ALLOW_MISSING_KEY_4_COMBINE` 改默认 `false`；写操作必须显式带条件，否则抛错而非可配置。

### 3.5 v9.x+（2027 Q4 及以后展望）

- GraalVM native-image 支持（AOT 反射配置生成）。
- 自然语言 → APIJSON 的 AI 适配层（复用 `@combine` AST）。
- 分布式 SQL 联邦查询（跨数据源 JOIN 的执行计划器）。
- WebAssembly 构建，让同一份协议在浏览器/边缘侧校验请求。

---

## 4. 里程碑与验收

| 里程碑 | 目标日期 | 验收标准 |
|--------|----------|----------|
| M1 v8.2.x 首个补丁 | 2026-08（滚动维护至 v8.3 发布） | IdentityPolicy 合入；`@combine` 单测覆盖率 ≥ 85%；prepared value 顺序修复合入 |
| M2 v8.3 可观测 | 2026-12 | Starter 可直接 `mvn spring-boot:run`；Micrometer 指标 ≥ 15 项 |
| M3 v8.4 SQL 增强 | 2027-03 | TCK 在 MySQL/PG/Oracle/ClickHouse/Doris/StarRocks 全绿 |
| M4 v9.0 模块化 | 2027-09 | Java 17 baseline、JPMS 模块图通过 `jdeps` 检查、迁移指南发布 |
| M5 v9.x 联邦/AOT | 2027-12 | native-image Helloworld 镜像 < 80MB；跨数据源 JOIN Demo |

---

## 5. 风险与对策

| 风险 | 影响 | 对策 |
|------|------|------|
| `@combine` 语法严格（空格要求）导致用户体验差 | 接入成本高 | v8.3 提供预检 lint API 与 IDE 插件；保留逗号列表式作为兜底 |
| 40+ 数据库方言维护成本 | 版本碎片化 | TCK 作为合入门禁，社区 PR 必须带对应数据库 Testcontainer 用例 |
| 默认放行 `ALLOW_MISSING_KEY_4_COMBINE` | 安全漏洞 | v9.0 改默认值；提供 `@audit` 模式日志告警 |
| Java 8 baseline 限制 API 演进 | 无法用 Records/sealed | v9.0 升 Java 17，保留 8.x 维护分支至 2027 年底 |
| 自研 JSON 容器无 SPI | [JSONMap](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/JSONMap.java)/[JSONList](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/src/main/java/apijson/JSONList.java) 基于 LinkedHashMap/ArrayList 手写（见类注释 "replace com.alibaba.fastjson"），[pom.xml](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/APIJSONORM/pom.xml#L23-L24) 无外部 JSON 依赖，无法插拔第三方 JSON 库，序列化特性需要自己维护 | v9.0 抽 `JSONProvider` SPI，保留当前实现为默认，允许接入 fastjson2/jackson 等 |

---

## 6. 贡献者指引

- 新功能分支从 `master` 切出，命名 `feat/<scope>-<short>`；修复 `fix/<scope>-<short>`。
- 改动 `newSQLConfig` 或 `parseCombineExpression` **必须**附带单元测试并更新 `APIJSON 通用文档.md` 中关键字章节。
- 新增数据库方言需在 `pom.xml` profile 中声明 Testcontainer 镜像并跑通 TCK。
- Commit 遵循 [Commit 规范.md](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Squirtle/Commit%20规范.md)。
