# APIJSON 规划及路线图

> 基于 APIJSON ORM v8.2.0 源码逐行核对分析，从 `newSQLConfig` 请求解析与 `@combine` 条件表达式引擎切入
>
> **标注约定**：
> - `[源码]` — 内容可在源码中直接定位验证，附有文件行号链接
> - `[推断]` — 基于源码逻辑的合理推导，非代码中显式声明
> - `[规划]` — 路线图/建议内容，当前版本未实现

---

## 一、项目现状分析

### 1.1 架构总览 `[源码]`

APIJSON 是腾讯开源的 **零代码 RESTful API 引擎**，核心通过 JSON 驱动 SQL 生成，实现后端接口的自动化。当前版本 **v8.2.0`[源码]`**（[pom.xml#L8](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/pom.xml#L8)），核心模块结构：

```
apijson/orm/
├── AbstractParser.java         # 请求解析入口，事务管理、全局配置、递归对象解析
├── AbstractObjectParser.java   # 对象/数组解析、关联查询、权限校验调度
├── AbstractSQLConfig.java      # SQL 配置核心：newSQLConfig()、@combine 引擎、SQL 生成 (6600+ 行)
├── AbstractSQLExecutor.java    # SQL 执行器、连接管理
├── AbstractVerifier.java       # 权限验证、登录校验、角色管理、6种角色常量
├── AbstractFunctionParser.java # 远程函数解析、脚本引擎支持 (JSR223)
├── Logic.java                  # 逻辑运算类型 (|&!) TYPE_OR=0, TYPE_AND=1, TYPE_NOT=2
├── Join.java                   # 连表配置，12种 JOIN 类型符号
├── Operation.java              # 操作枚举 (13种: MUST/REFUSE/TYPE/VERIFY/EXIST/UNIQUE/INSERT/UPDATE/REPLACE/REMOVE/IF/ALLOW_PARTIAL_UPDATE_FAIL/IS_ID_CONDITION_MUST)
├── Subquery.java               # 子查询配置 (path/from/range/key/config)
├── SQLConfig.java              # 接口：36种数据库常量、getter/setter声明
├── script/                     # 脚本执行器 (JSR223/JavaScript)
└── model/                      # 系统表模型 (Access/Request/Table/Column/Document/Function/Script)

apijson/ (根包)
├── JSONMap.java                # KEY_ 常量定义（~40个），TABLE_KEY_LIST 共 34 个 @ 关键词白名单
├── JSON.java                   # JSON 工具类
├── SQL.java                    # SQL 关键字常量与函数工具 (count/sum/max/min/avg/concat/replace/...)
├── RequestMethod.java          # 8种方法枚举 (GET/HEAD/GETS/HEADS/POST/PUT/DELETE/CRUD)
└── StringUtil.java             # 字符串工具
```

> `[源码]` 模块结构来源于 [src/main/java/apijson/orm/](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm) 和 [src/main/java/apijson/](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson) 目录实际文件列表。

> **关于关键词数量的区分**`[源码]`：
> - **TABLE_KEY_LIST（[JSONMap.java#L207-L242](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/JSONMap.java#L207)）共 34 个** — 这是所有被识别为"表级配置关键词"的 `@` 前缀 key 白名单，由 `AbstractObjectParser` 传入 `newSQLConfig` 前过滤使用。34 个中：25 个在 `newSQLConfig` 内被提取处理，9 个由 Parser/ObjectParser 层消费（`@string`/`@trim`/`@get`/`@gets`/`@head`/`@heads`/`@post`/`@put`/`@delete`）。
> - **newSQLConfig 内提取的关键词共 25 个** — 分为两批：阶段[1]-[2]提前提取 6 个（`@explain`/`@database`/`@datasource`/`@namespace`/`@catalog`/`@schema`，[L5484-L5498](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L5484)）；阶段[8] try 块内批量提取 19 个（`@role`/`@cache`/`@from`/`@column`/`@null`/`@cast`/`@combine`/`@group`/`@having`/`@having&`/`@sample`/`@latest`/`@partition`/`@fill`/`@order`/`@key`/`@raw`/`@json`/`@method`，[L5639-L5657](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L5639)）。
> - **不在 TABLE_KEY_LIST 中的 4 个关键词**：`@try`/`@catch`/`@drop`/`@default`（[JSONMap.java#L166-L170](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/JSONMap.java#L166)），完全由 Parser 层处理，不进入 `newSQLConfig`。

### 1.2 核心能力矩阵

| 能力域 | 实现位置 | 状态 | 来源 |
|--------|----------|------|------|
| 请求解析 → SQLConfig | [AbstractSQLConfig.newSQLConfig()](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L5477) | ✅ 成熟 | `[源码]` |
| @combine 布尔表达式引擎 | [parseCombineExpression()](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L3364) | ✅ 成熟 | `[源码]` |
| @combine 简单列表模式 | [getWhereString()](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L3666) | ✅ 兼容 | `[源码]` |
| 35种数据库适配 | [DATABASE_LIST](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L146) 静态初始化 | ✅ 成熟 | `[源码]` |
| 12种 JOIN 类型 | [Join.java#L21](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/Join.java#L21) + [concatJoinWhereString()](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L3734) switch | ✅ 成熟 | `[源码]` |
| 远程函数/脚本引擎 | [AbstractFunctionParser.java](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractFunctionParser.java) + [script/](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/script) | ✅ 可用 | `[源码]` |
| 6种权限角色 | [AbstractVerifier.java#L76-L96](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractVerifier.java#L76) | ✅ 可用 | `[源码]` |
| 假删除(软删除) | [isFakeDelete()](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L4918) + ACCESS_FAKE_DELETE_MAP | ✅ 可用 | `[源码]` |
| PreparedStatement 预编译 | [gainValue()](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L4062) + preparedValueList | ✅ 成熟 | `[源码]` |
| WITH AS 表达式 | [ENABLE_WITH_AS](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L45) = false 默认关闭 | ⚠️ 可配置 | `[源码]` |

> **关于数据库数量**`[源码]`：SQLConfig 接口定义了 36 个 DATABASE_ 常量（[SQLConfig.java#L20-L57](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/SQLConfig.java#L20)，含 SQLITE），但 [DATABASE_LIST](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L146)（`@database` 合法值白名单）实际注册了 35 种，SQLITE 未列入白名单。

---

## 二、`newSQLConfig` 请求解析全流程 `[源码]`

### 2.1 入口签名 `[源码]`

[AbstractSQLConfig.java#L5477-L5479](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L5477)：

```java
public static <T, M extends Map<String, Object>, L extends List<Object>> SQLConfig<T, M, L> newSQLConfig(
    RequestMethod method,       // GET/HEAD/GETS/HEADS/POST/PUT/DELETE/CRUD
    String table,               // 表名
    String alias,               // 表别名
    M request,                  // 请求 JSON Map（会被修改后还原）
    List<Join<T,M,L>> joinList, // 连表列表
    boolean isProcedure,        // 是否存储过程
    Callback<T,M,L> callback    // 回调：getSQLConfig()/getIdKey()/getUserIdKey()/newId() 等
) throws Exception
```

### 2.2 解析阶段（按源码执行顺序，对应行号） `[源码]`

```
请求JSON (request Map)
  │
  ├─[1] @explain 校验 (L5484-L5487)
  │     非 DEBUG 模式禁止 @explain:true
  │
  ├─[2] 全局参数提取 (L5489-L5498)
  │     @database, @datasource, @namespace, @catalog, @schema
  │     @database 值必须在 DATABASE_LIST (35种) 白名单内
  │
  ├─[3] 创建 SQLConfig 实例 (L5500)
  │     callback.getSQLConfig() → 根据 database 创建对应子类
  │     设置 alias/database/datasource/namespace/catalog/schema
  │
  ├─[4] 存储过程短路 (L5509-L5511)
  │     isProcedure=true → 直接返回 config
  │
  ├─[5] 解析 JOIN (L5513) → parseJoin()
  │     递归调用 newSQLConfig 处理每个 join 的 request/on/outer
  │     设置 keyPrefix（副表加别名前缀）
  │
  ├─[6] 空请求短路 (L5515-L5517)
  │     request.isEmpty() → 直接返回
  │
  ├─[7] id/userId 强制条件处理 (L5519-L5636)
  │     idKey="id", idInKey="id{}", userIdKey="userId", userIdInKey="userId{}"
  │     过滤无效值(≤0的数字、空字符串) → 类型校验(仅Number/String/Subquery)
  │     id{}∈Collection 时去重 → DELETE/PUT 时 setCount(size)
  │     id 非 null 时 DELETE/PUT setCount(1)
  │     POST 时 id==null 调用 callback.newId() 生成
  │
  ├─[8] 关键词变量提取 (L5639-L5657)
  │     @role, @cache, @from, @column, @null, @cast, @combine,
  │     @group, @having, @having&, @sample, @latest, @partition,
  │     @fill, @order, @key, @raw, @json, @method
  │     共19个关键词（此为 try 块内批量提取的数量；加上阶段[1]-[2]提前提取的 6 个，newSQLConfig 共处理 25 个关键词）
  │
  ├─[9] try 块内：remove 所有 id/userId + 25个关键词 (L5661-L5690)
  │
  ├─[10] @null 处理 (L5693-L5708)
  │      逗号分隔 "key0,key1..." → request.put(nk, null)
  │      校验：nk 不可为空、nk 不可已有非 null 值
  │
  ├─[11] @cast 处理 (L5710-L5734)
  │      格式 "key0:type0,key1:type1..." → castMap
  │      校验：key 非空、type 符合命名格式、不可重复
  │      gainValue() 时使用 cast(? AS type)
  │
  ├─[12] @raw 数组处理 (L5737-L5738)
  │      config.setRaw(rawArr) — 标记哪些 key 允许原始 SQL 片段
  │
  ├─[13] 分流：POST vs 非POST (L5750)
  │      │
  │      ├─ POST 路径 (L5750-L5798):
  │      │   禁止 id{}/userId{} 非命名字段
  │      │   校验所有 key 必须是合法字段名 (StringUtil.isName)
  │      │   columns = 剩余 key 集 + id/userId
  │      │   values = 对应值 + id/userId值
  │      │   config.setValues([[id?, userId?, val0, val1, ...]])
  │      │
  │      └─ 非POST 路径 (L5799-L5985):
  │         isWhere = (method != PUT)  // GET/HEAD/DELETE 全是条件
  │         
  │         combine 解析 (L5803-L5944):
  │           ws = split(combine, ",")
  │           若 ws.length==1 → combineExpr (布尔表达式模式)
  │           否则 → 简单列表模式：
  │             &key → andList, |key → orList, !key → notList, 无前缀 → orList
  │             PUT 禁止 |key 和 !key
  │             禁止引用 id/userId/id{}/userId{}
  │             缺失key → callback.onMissingKey4Combine()
  │         
  │         假删除处理 (L5835-L5881):
  │           非DELETE时追加 deletedKey!=deletedValue / deletedKey=notDeletedValue
  │         
  │         遍历剩余 key 分流 (L5950-L5974):
  │           忽略空/空白字符串（IGNORE_EMPTY_STRING_METHOD_LIST 控制）
  │           禁止非 <> 后缀的 key 对应 Map 类型值
  │           if (isWhere || key非字段名 || key在combineExpr中) → tableWhere
  │           else if (key在whereList中) → tableWhere
  │           else → tableContent (PUT SET 部分)
  │         
  │         combineMap = {"&": andList, "|": orList, "!": notList}
  │         config.setCombine(combineExpr) / setCombineMap(combineMap)
  │         config.setContent(tableContent)
  │
  ├─[14] 假删除 DELETE→PUT 转换 (L5987-L6008)
  │      method==DELETE 且 enableFakeDelete → setMethod(PUT) + setContent(fakeDeleteMap)
  │
  ├─[15] @column 解析 (L6010-L6055)
  │      @raw 包含 @column 时 → gainRawSQL 原始片段
  │      DISTINCT 前缀检测 PREFIX_DISTINCT="DISTINCT "
  │      按 ; 分割：含 ( 的为函数表达式，否则按空格分割为字段列表
  │
  ├─[16] @having 处理 (L6058-L6148)
  │      @having& 强制 AND；@having 默认 OR (IS_HAVING_DEFAULT_AND=false, L35)
  │      String格式: 按 ; 分割 → 每段按 ( ) 检测函数 → 生成 having0,having1...
  │      Map格式: { "fun0": "val0", "@combine": "expr" }
  │      默认 OR 连接多个 having 条件 (L6102-L6103)
  │
  ├─[17] @key 映射处理 (L6150-L6169)
  │      Map 直接 setKeyMap；String 按 ; 分割 Pair.parseEntry
  │
  ├─[18] config 属性批量设置 (L6172-L6197)
  │      explain/cache/distinct/column/from/role/id/idIn/userId/userIdIn
  │      null/cast/where/group/having/havingCombine/sample/latest/partition/fill/order/json
  │
  └─[19] finally 块：还原 request (L6200-L6285)
        所有已 remove 的 key 按是否非 null 决定是否 put 回
        （id/userId/条件/25个关键词，注意 namespace/catalog 未在 finally 中还原）
```

### 2.3 WHERE vs CONTENT 分流规则 `[源码]`

对应 [L5950-L5974](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L5950)：

```java
// isWhere = (method != PUT)
if ((isWhere || (StringUtil.isName(key.replaceFirst("[+-]$", "")) == false))
    || (isWhere == false && StringUtil.isNotEmpty(combineExpr, true) && isKeyInCombineExpr(combineExpr, key))) {
    tableWhere.put(key, value);  // → WHERE 条件
} else if (whereList.contains(key)) {
    tableWhere.put(key, value);  // → WHERE 条件 (简单combine引用)
} else {
    tableContent.put(key, value); // → PUT SET 子句
}
```

| 场景 | 去向 |
|------|------|
| GET/HEAD/DELETE (`isWhere=true`) | 全部 → tableWhere |
| PUT + key含操作符后缀(`$~%{}{}`等)或`+`/`-`后缀 | → tableWhere |
| PUT + key在@combine布尔表达式中 | → tableWhere |
| PUT + key在简单combine的whereList中 | → tableWhere |
| PUT + 纯字段名且不在combine中 | → tableContent (SET) |

> `[推断]` 注意：`+`/`-` 后缀在 PUT SET 中分别对应 `gainAddString()`（字段值+=）和 `gainRemoveString()`（字段值-=或REPLACE），见 [gainSetString()](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L4842)。但它们因 `StringUtil.isName(key.replaceFirst("[+-]$", "")) == false` 而被归入 WHERE——这在 PUT 场景下 `[推断]` 意味着带 `+`/`-` 后缀的 key 在非 combine 时被当条件而非 SET 内容。实际 SET 中 `+`/`-` 的检测发生在 [gainSetString()](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L4859) 遍历 content 时（L4859-L4864），即 `+`/`-` 后缀的 key 必须同时出现在 tableContent 中才会被 SET 使用——这说明 PUT 时 `key+`/`key-` 如果不在 @combine 表达式中，会被归入 tableWhere（条件），而 SET 中的加减逻辑依赖于 key 已被放入 tableContent。`[推断]` 实际使用时，PUT 操作的 `key+` 通常配合 @combine 使用。

---

## 三、`@combine` 条件表达式引擎详解 `[源码]`

### 3.1 两种模式 `[源码]`

#### 模式一：简单列表模式（`getWhereString()` L3666）

```json
{
  "id{}": [1,2,3],
  "sex": 0,
  "name$": "a",
  "@combine": "id{},&sex,!name$"
}
```

前缀语义`[源码]`（[L5892-L5918](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L5892)）：
- `&key` → andList（AND 组内连接）
- `|key` → orList（OR 组内连接）
- `!key` → notList（NOT 包裹 OR 组）
- 无前缀 → orList（默认 OR 组）

combineMap 固定按 `"&"`, `"|"`, `"!"` 顺序存入 LinkedHashMap（[L5977-L5979](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L5977)），组间 AND 连接（[L3718](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L3718)）。

对于上例：andList=["sex"], orList=["id{}"], notList=["name$"]
生成SQL：`(sex = 0) AND (id IN(1,2,3)) AND NOT (name LIKE '%a%')`

#### 模式二：布尔表达式模式（`parseCombineExpression()` L3364）

```json
{
  "date>": "2024-01-01",
  "contactIdList<>": [1,2],
  "name*~": "a",
  "tag&$": "%a%",
  "@combine": "date> | (contactIdList<> & (name*~ | tag&$))"
}
```

### 3.2 表达式语法规范 `[源码]`

所有规则来自 [parseCombineExpression()](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L3364) 中的校验逻辑：

| 语法元素 | 规则 | 源码校验位置 |
|----------|------|-------------|
| 条件key | 与 where 中 key 完全一致 | L3445-L3457 |
| AND连接 | ` & `（两侧各一个空格） | L3496-L3505 |
| OR连接 | ` \| `（两侧各一个空格） | L3511-L3526 |
| NOT前缀 | `!` 紧跟 key 或 `(`，右侧无空格；`!(...)` 合法 | L3528-L3557 |
| NOT右侧禁止 | 空格、`)`、`&`、`\|` | L3533-L3542 |
| 括号分组 | `(...)` 内侧无空格 | L3372, L3562 |
| 首尾禁止空格 | 不允许首尾有空格/连续空格 | L3369-L3373 |
| 左括号(左侧 | 必须有 &\| 连接符或位于开头 | L3563 |
| 嵌套层级上限 | MAX_COMBINE_DEPTH=2 | L63, L3570 |
| 表达式key总数上限 | MAX_COMBINE_COUNT=5 | L64, L3437 |
| 同key引用上限 | MAX_COMBINE_KEY_COUNT=2 | L65, L3473 |
| key/总条件比上限 | MAX_COMBINE_RATIO=1.0 | L66, L3460 |
| WHERE条件总数上限 | MAX_WHERE_COUNT=10 | L62, L3381 |
| HAVING条件总数上限 | MAX_HAVING_COUNT=5 | L61, L3381 |

### 3.3 解析器状态机 `[源码]`

核心是 [L3409-L3595](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L3409) 的逐字符 `while(i <= n)` 扫描，状态变量：

```
i          → 当前字符位置
depth      → 括号嵌套深度
lastLogic  → 上一个逻辑运算符 char (&| 或 0)
last       → 上一个字符
first      → 是否第一个条件（第一个条件不需要前置 &|）
isNot      → 是否处于 NOT 取反状态
key        → 当前正在累积的 key 字符串
```

处理逻辑（按字符类型分支）：

| 字符 | 处理 |
|------|------|
| ` ` (空格) 或 `)` 或结束 | 终结当前 key → gainWhereItem/gainHavingItem → `result += "( " + gainCondition(isNot, wi) + " )"` |
| `&` | 前一字符是空格 → 追加 SQL.AND（` AND `），否则作为 key 一部分 |
| `\|` | 前一字符是空格 → 追加 SQL.OR（` OR `），否则作为 key 一部分 |
| `!` | 前一字符是空格或`(`：后一字符是`(`→追加 SQL.NOT；否则 isNot=true。否则作为 key 一部分 |
| `(` | 校验左侧必须有 &\| → depth++ → 追加`(` → first=true |
| `)` | depth-- → 追加`)` |
| 其它 | 累加入 key |

表达式处理完后`[源码]`（[L3608-L3651](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L3608)）：

1. **未引用条件**：conditionMap 中不在 usedKeyCountMap 里的 key → AND 拼接为 `andCond`
2. **WHERE模式**（非HAVING）：
   - 表达式内条件值单独收集到 exprPreparedValues
   - AND 条件值先放入 preparedValueList，再追加表达式条件值（保证 `?` 占位符顺序）
   - 最终组合：`andCond AND ( result )`（WHERE 时 andCond 在前；HAVING 时 result 在前）
3. **HAVING模式**：`( result ) AND andCond`

### 3.4 条件操作符映射 `[源码]`

[gainWhereItem()](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L3897) 通过 key 后缀判断 keyType，switch 分发：

| key后缀 | keyType | 分发方法 | SQL 语义 |
|---------|---------|----------|----------|
| (无后缀) | 0 | gainEqualString | `key = ?` / `key != ?`(key!后缀) / `key IS NULL`(value=null) |
| `$` | 1 | gainSearchString | `key LIKE ?`（自动添加%前缀/后缀`[推断]`） |
| `~` | 2 | gainRegExpString | 正则匹配：PG→`key ~ ?`，MySQL8+/Oracle→`regexp_like(key, ?, 'c')`，CK→`match(key, ?)` |
| `*~` | -2 | gainRegExpString(ignoreCase=true) | 忽略大小写正则：PG→`key ~* ?`，MySQL8+→`regexp_like(key, ?, 'i')`，CK→`match(lower(key), lower(?))` |
| `%` | 3 | gainBetweenString | `key BETWEEN ? AND ?` |
| `{}` | 4 | gainRangeString | `key IN (?,?,...)` 或条件链 |
| `}{` | 5 | gainExistsString | `EXISTS (子查询)` |
| `<>` | 6 | gainContainString | JSON 数组包含 |
| `>=` | 7 | gainCompareString | `key >= ?` |
| `<=` | 8 | gainCompareString | `key <= ?` |
| `>` | 9 | gainCompareString | `key > ?` |
| `<` | 10 | gainCompareString | `key < ?` |

> **后缀匹配顺序**`[源码]`：if-else if 链从长后缀到短后缀依次检测：`$`→`~`→`%`→`{}`→`}{`→`<>`→`>=`→`<=`→`>`→`<`→默认0。注意 `>=` 必须在 `>` 之前检测（否则 `>=` 会先匹配到 `>`）。

**key 逻辑符后缀**`[源码]`（gainEqualString L3985-L3988）：
- `!`（紧跟字段名后，如 `name!`）→ NOT 等价：`name != value` / `name IS NOT NULL`
- `&` / `|` 在 RegExp/Range 等多值条件中控制 AND/OR 连接（通过 [Logic.java](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/Logic.java) 构造器解析 key 最后一个字符）

**key 长度函数前缀**`[源码]`（[gainKey()](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L4014) L4016-L4022）：
- `key[` → 字符串长度：SQL Server→`datalength(key)`，其他→`length(key)`
- `key{` → JSON 长度：`json_length(key)`
- 这些是 key 前缀（在操作符后缀之前），例如 `content[>}` `[推断]` 应为 `content[>` 表示 `length(content) > ?`

---

## 四、JOIN 连表体系 `[源码]`

### 4.1 JOIN 类型全集

来源 [Join.java#L21](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/Join.java#L21) 注释 + [concatJoinWhereString()](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L3753) switch 分支：

| 符号 | 类型 | SQL 语义 | 源码位置 |
|------|------|----------|---------|
| `@` | APP JOIN | 应用层 Join（分两次查询，程序内存拼接） | L3755 |
| `<` | LEFT JOIN | `A LEFT JOIN B ON ...` | L3756 |
| `>` | RIGHT JOIN | `A RIGHT JOIN B ON ...` | L3757 |
| `*` | CROSS JOIN | `A CROSS JOIN B` | L3754 |
| `&` | INNER JOIN | `A INNER JOIN B ON ...`（A & B） | L3781 |
| `\|` / `""` | FULL JOIN | `A FULL JOIN B`（A \| B），空串`""`为默认（路径`/Table`无符号前缀时） | L3782-L3783 |
| `!` | OUTER JOIN | `NOT (A \| B)`（A 或 B 为空的部分） | L3784, L3802 |
| `^` | SIDE JOIN | `NOT (A & B)`（A、B 不相交的部分） | L3785, L3846 |
| `(` | ANTI JOIN | `A AND NOT B` | L3786, L3840 |
| `)` | FOREIGN JOIN | `NOT A AND B` | L3787, L3843 |
| `~` | ASOF JOIN | `B ~= A`（时序数据最近匹配） | L3788 |

> **joinType 提取**`[源码]`（[AbstractParser.java#L1619](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractParser.java#L1619)）：joinType = 路径中第一个 `/` 之前的子串。例如 `</User` → `<`，`/User` → `""`（默认FULL），`&/User` → `&`。

### 4.2 ON 条件 `[源码]`

join 路径支持 `/key@` 后缀指定 ON 关联键：
- `</User/id@` → LEFT JOIN User ON User.id = 当前表.userId
- 支持 `{}`, `<>`, `$`, `~`, `>=`, `<=`, `>`, `<` 后缀作为 ON 比较关系
- join 对象内部可独立传 `@column`/`@group`/`@order`/`@combine`/`@where条件` 等

---

## 五、安全限制常量 `[源码]`

全部来自 [AbstractSQLConfig.java#L61-L67](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L61)：

| 常量 | 默认值 | 语义 |
|------|--------|------|
| MAX_HAVING_COUNT | 5 | HAVING 条件最大数量 |
| MAX_WHERE_COUNT | 10 | WHERE 条件最大数量 |
| MAX_COMBINE_DEPTH | 2 | @combine 括号最大嵌套深度 |
| MAX_COMBINE_COUNT | 5 | @combine 表达式中 key 最大数量 |
| MAX_COMBINE_KEY_COUNT | 2 | 同一 key 在 @combine 中最大引用次数 |
| MAX_COMBINE_RATIO | 1.0f | 表达式key数/总条件数最大比值 |
| ALLOW_MISSING_KEY_4_COMBINE | true | @combine 引用不存在的 key 时是否允许（warn 而非 error） |

---

## 六、版本路线图 `[规划]`

### v8.2.0（当前版本）`[源码]`

- ✅ 35种数据库白名单支持（DATABASE_LIST）
- ✅ @combine 布尔表达式引擎（逐字符状态机）
- ✅ 12种 JOIN 类型（含 APP/ANTI/FOREIGN/ASOF）
- ✅ PreparedStatement 预编译 + 有序值绑定
- ✅ 假删除/软删除（ACCESS_FAKE_DELETE_MAP）
- ✅ 远程函数/脚本引擎（JSR223，ENALBE_SCRIPT_FUNCTION 控制）
- ✅ 多租户/多数据源（@datasource/@namespace/@schema/@catalog）
- ✅ 6种权限角色（UNKNOWN/LOGIN/CONTACT/CIRCLE/OWNER/ADMIN）

### v8.x 短期规划 `[规划]`

| 优先级 | 目标 | 关键任务 |
|--------|------|----------|
| P0 | 表达式引擎增强 | @combine 支持 IN/BETWEEN 字面量；嵌套深度可配置化 |
| P0 | 安全加固 | 脚本引擎沙箱强制隔离；@raw 白名单机制；SQL注入检测增强 |
| P1 | 性能优化 | WITH AS 全数据库覆盖（当前默认关闭）；执行计划缓存 |
| P1 | 新数据库适配 | 完善 Snowflake/Databricks/DuckDB 适配器；SQLITE 加入白名单 |
| P2 | 开发者体验 | 错误信息结构化+精确位置标注；@explain 返回索引建议 |

### v9.0 长期愿景 `[规划]`

| 方向 | 愿景 |
|------|------|
| 智能查询优化 | 基于查询历史自动推荐 JOIN 策略、索引提示 |
| GraphQL 融合层 | 在现有 JSON 协议上叠加 GraphQL 风格查询能力 |
| 实时推送 | 基于 CDC 的 @subscribe 实时订阅 |
| 多语言 SDK | Go/Rust/Python 版 ORM 内核，共享 JSON 协议 |
| 可视化管理台 | 内置 Admin UI，可视化配置权限/函数/文档 |
| AI 辅助查询 | 自然语言 → APIJSON 请求的智能转换 |

---

## 七、技术债务与优化建议 `[推断]`

### 7.1 代码质量 `[推断]`

1. **超大类**：[AbstractSQLConfig.java](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java) 6600+ 行，承担 SQL 生成、条件解析、JOIN 处理、表达式引擎等多重职责，建议拆分为：
   - `CombineExpressionParser` — @combine 状态机（当前 L3364-L3654）
   - `ConditionOperatorRegistry` — gainWhereItem 各操作符（当前 L3897-L4300+）
   - `JoinClauseBuilder` — JOIN ON 拼接（当前 L3734-L3886）
   - `HavingClauseBuilder` — @having 处理（当前 gainHavingString）
2. **public static 可变配置**：MAX_* 系列限制、ENABLE_WITH_AS、IS_HAVING_DEFAULT_AND 等为 public static，全局可修改，`[推断]`存在线程安全和多租户隔离风险。建议改为实例级或 Builder 模式配置。
3. **泛型参数链过长**：`<T, M extends Map<String,Object>, L extends List<Object>>` 贯穿几乎所有类，`[推断]`可引入中间类型别名或简化。

### 7.2 性能相关 `[推断]`

1. `newSQLConfig` 对 request 的 `remove` + `finally put` 操作（[L5661-L6285](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L5661)）修改传入 Map，`[推断]`调用方需确保传入副本以避免并发问题。
2. `isKeyInCombineExpr()` 使用子串匹配，`[推断]`最坏 O(n×m)，可预编译为 token 集合。
3. RAW_MAP 为 LinkedHashMap（[L184](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L184)），包含 80+ 条目，`[推断]`高频查询场景可优化。

### 7.3 安全性 `[推断]`

1. @raw 原始 SQL 片段需强权限管控。[gainRawSQL()](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java) 依赖 RAW_MAP 白名单校验，但 RAW_MAP 是 public static 可运行时修改。
2. 脚本引擎（IF/Function）需配置 ClassFilter 沙箱，见 [Operation.java#L135](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/Operation.java#L135)。
3. PATTERN_FUNCTION 正则（[L110](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L110)）禁止了 `'`/`"`/`;`/`--`/`#`/`/**/` 等注入字符，`[推断]`建议增加更多边界 case 测试。
