# APIJSON 规划及路线图

> 基于 APIJSON ORM v8.2.0 源码深度分析，从 `newSQLConfig` 请求解析与 `@combine` 条件表达式引擎切入

---

## 一、项目现状分析

### 1.1 架构总览

APIJSON 是腾讯开源的 **零代码 RESTful API 引擎**，核心通过 JSON 驱动 SQL 生成，实现后端接口的自动化。当前版本 **v8.2.0**，核心模块结构：

```
apijson/orm/
├── AbstractParser.java        # 请求解析入口，事务管理、全局配置、递归对象解析
├── AbstractSQLConfig.java     # SQL 配置核心：newSQLConfig()、@combine 引擎、SQL 生成
├── AbstractObjectParser.java  # 对象/数组解析、关联查询、权限校验调度
├── AbstractSQLExecutor.java   # SQL 执行器、连接管理、缓存策略
├── AbstractVerifier.java      # 权限验证、登录校验、角色管理、操作审计
├── AbstractFunctionParser.java# 远程函数解析、脚本引擎支持
├── Logic.java                 # 逻辑运算类型 (|&!)
├── Join.java                  # 连表配置 (12种 JOIN 类型)
├── Operation.java             # 操作枚举 (MUST/REFUSE/TYPE/VERIFY/INSERT/UPDATE...)
├── Subquery.java              # 子查询配置
└── model/                     # 系统表模型 (Access/Request/Table/Column/Document...)

apijson/ (根包)
├── JSON.java                  # JSON 工具类
├── JSONMap.java               # KEY_ 常量定义与链式 API
├── SQL.java                   # SQL 关键字与函数工具
├── RequestMethod.java         # HTTP 方法枚举 (GET/POST/HEAD/PUT/DELETE/CRUD)
└── StringUtil.java            # 字符串工具
```

### 1.2 核心能力矩阵

| 能力域 | 实现类 | 状态 |
|--------|--------|------|
| 请求解析 → SQLConfig | [AbstractSQLConfig.newSQLConfig()](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L5477) | ✅ 成熟 |
| @combine 表达式引擎 | [parseCombineExpression()](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L3364) | ✅ 成熟 |
| 多数据库适配 (30+) | SQLConfig 数据库标识常量 | ✅ 成熟 |
| 12 种 JOIN 类型 | [Join.java](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/Join.java) | ✅ 成熟 |
| 远程函数/脚本引擎 | AbstractFunctionParser + script/ | ✅ 可用 |
| 权限/角色/访问控制 | AbstractVerifier + model/Access | ✅ 可用 |
| 假删除 (软删除) | onFakeDelete() + ACCESS_FAKE_DELETE_MAP | ✅ 可用 |
| PreparedStatement | gainValue() 预编译支持 | ✅ 成熟 |
| SQL 缓存机制 | SQLExecutor 缓存计数 | ✅ 可用 |
| WITH AS 表达式 | isWithAsEnable() | ⚠️ 可配置 |

---

## 二、`newSQLConfig` 请求解析全流程

### 2.1 入口签名

```java
static <T, M extends Map<String, Object>, L extends List<Object>>
SQLConfig<T, M, L> newSQLConfig(
    RequestMethod method,      // GET/POST/PUT/DELETE/HEAD
    String table,              // 表名
    String alias,              // 表别名
    M request,                 // 请求 JSON Map
    List<Join<T,M,L>> joinList,// 连表列表
    boolean isProcedure,       // 是否存储过程
    Callback<T,M,L> callback   // 回调（获取SQLConfig实例、idKey、userIdKey等）
) throws Exception
```

### 2.2 解析阶段（按执行顺序）

```
请求JSON
  │
  ├─[1] 全局参数提取
  │     @explain, @database, @datasource, @namespace, @catalog, @schema
  │
  ├─[2] 创建 SQLConfig 实例
  │     callback.getSQLConfig() → 根据 database 类型创建对应子类
  │
  ├─[3] 解析 JOIN (parseJoin)
  │     处理 </User, &User, @/User 等关联表配置
  │
  ├─[4] id/userId 强制条件处理
  │     过滤无效值(0,负数,空串) → 类型校验 → DELETE/PUT count设置
  │
  ├─[5] 关键词提取
  │     @role, @cache, @from, @column, @null, @cast, @combine,
  │     @group, @having, @having&, @sample, @latest, @partition,
  │     @fill, @order, @key, @raw, @json, @method
  │
  ├─[6] @null / @cast 处理
  │     为指定key设null值 / 设置类型转换映射
  │
  ├─[7] 分流：POST vs 非POST
  │     ├─ POST: 收集所有字段 → columns + values → INSERT
  │     └─ 非POST:
  │           ├─ combine表达式解析 → combineMap (andList/orList/notList)
  │           ├─ WHERE条件 (id/userId/假删除/遍历request剩余key)
  │           ├─ PUT: 非条件字段归入 tableContent (SET 部分)
  │           └─ 假删除: DELETE 转换为 PUT + 设置 deleted 标记
  │
  ├─[8] @column 处理
  │     支持 "key0,key1;fun0(key0);fun1(key0,...)" 格式
  │     含 DISTINCT 前缀处理 / @raw 原始片段 / 函数表达式
  │
  ├─[9] @having 处理
  │     String格式: "fun0(col)?val0;fun1(col)?val1"
  │     Map格式:    { "fun0": "val0", "@combine": "expr" }
  │     默认OR连接(5.0+), @having&强制AND
  │
  ├─[10] @key 映射处理
  │      "year:left(date,4);name_tag:(name,tag)"
  │
  └─[11] 还原 request (finally块)
        所有已remove的key重新put回，保证后续处理可用
```

### 2.3 WHERE vs CONTENT 分流规则

```
非POST方法时，对每个request中剩余key:
  ├─ isWhere=true (GET/DELETE/HEAD) → tableWhere (WHERE条件)
  ├─ isWhere=false (PUT) 且 key含功能符后缀(+/-/$/~/%/{}等) → tableWhere
  ├─ PUT且在@combine表达式中 → tableWhere
  └─ 其余 → tableContent (SET更新内容)
```

---

## 三、`@combine` 条件表达式引擎详解

### 3.1 两种模式

#### 模式一：简单列表模式 (旧版兼容)

```json
{
  "id{}": [1,2,3],
  "sex": 0,
  "name$": "a",
  "@combine": "id{},&sex,!name$"
}
```

前缀语义：`&key` → AND组, `|key` → OR组, `!key` → NOT组, 无前缀 → OR组
生成 `combineMap: { "&":["id{}","sex"], "|":["name$"], "!":[] }`
最终SQL：`(id IN(1,2,3)) AND (sex=0) AND NOT (name LIKE '%a%')`

#### 模式二：布尔表达式模式 (5.0+推荐)

```json
{
  "date>": "2024-01-01",
  "contactIdList<>": [1,2],
  "name*~": "a",
  "tag&$": "%a%",
  "@combine": "date> | (contactIdList<> & (name*~ | tag&$))"
}
```

### 3.2 表达式语法规范

| 语法元素 | 规则 | 示例 |
|----------|------|------|
| 条件key | 与where中key完全一致 | `date>`, `name*~`, `id{}` |
| AND连接 | ` & ` (两侧各一个空格) | `a & b` |
| OR连接 | ` \| ` (两侧各一个空格) | `a \| b` |
| NOT前缀 | `!` (紧跟key或`(`，无空格) | `!a`, `!(a \| b)` |
| 括号分组 | `(...)` (内侧无空格) | `(a & b) \| c` |
| 嵌套层级 | 最大 `MAX_COMBINE_DEPTH=2` | `(a & (b \| c))` |
| key引用次数 | 最大 `MAX_COMBINE_KEY_COUNT=2` | 同一key最多出现2次 |
| 总key数量 | 最大 `MAX_COMBINE_COUNT=5` | 表达式中最多5个key |
| key/条件比 | 最大 `MAX_COMBINE_RATIO=1.0` | 引用key数/总条件数 ≤ 1.0 |

### 3.3 解析器状态机 (parseCombineExpression)

核心是一个**逐字符扫描**的状态机，关键状态变量：

```
i          → 当前字符位置
depth      → 括号嵌套深度
lastLogic  → 上一个逻辑运算符 (&|)
last       → 上一个字符
first      → 是否第一个条件
isNot      → 是否处于 NOT 取反状态
key        → 当前正在累积的 key 字符串
```

处理流程：

```
逐字符 c in combine表达式:
  c == ' '  → 终结当前key → 生成条件片段 wi → 拼入 result
  c == '&'  → 前一个字符是空格 → 追加 SQL.AND → lastLogic='&'
              否则作为key的一部分 (功能符后缀)
  c == '|'  → 前一个字符是空格 → 追加 SQL.OR → lastLogic='|'
              否则作为key的一部分
  c == '!'  → 前一个字符是空格或'(':
                后一个字符是'(' → 追加 SQL.NOT
                否则 → isNot=true (对单个key取反)
              否则作为key的一部分
  c == '('  → 校验前必须有&|连接 → depth++ → 追加'('
  c == ')'  → depth-- → 追加')'
  其它       → 累加入key
结束后:
  → 未被表达式引用的where条件 → AND 连接到 result 后
  → preparedValueList 先 AND 条件值，后表达式条件值 (保证占位符顺序)
```

### 3.4 条件操作符映射 (gainWhereItem)

| key后缀 | keyType | SQL生成方法 | 语义 |
|---------|---------|-------------|------|
| (无) | 0 | gainEqualString | `=`, `!=` (key!后缀), `IS NULL` |
| `$` | 1 | gainSearchString | `LIKE` (支持%_占位符) |
| `~` | 2 | gainRegExpString | `REGEXP` / `~`(PG) / `regexp_like` |
| `*~` | -2 | gainRegExpString(ignoreCase) | 忽略大小写正则 |
| `%` | 3 | gainBetweenString | `BETWEEN ... AND ...` |
| `{}` | 4 | gainRangeString | `IN (...)` / 比较条件链 |
| `}{` | 5 | gainExistsString | `EXISTS (子查询)` |
| `<>` | 6 | gainContainString | JSON数组包含 |
| `>=` | 7 | gainCompareString | `>=` |
| `<=` | 8 | gainCompareString | `<=` |
| `>` | 9 | gainCompareString | `>` |
| `<` | 10 | gainCompareString | `<` |

key尾部逻辑符（在功能符之后）：
- `&` → AND 组合同key多值
- `|` → OR 组合同key多值
- `!` → NOT 取反

key长度/JSON函数前缀：
- `key[` → `length(key)` 字符串长度比较
- `key{` → `json_length(key)` JSON长度比较

---

## 四、JOIN 连表体系

### 4.1 JOIN 类型全集 (Join.java)

| 符号 | 类型 | SQL语义 |
|------|------|---------|
| `@` | APP JOIN | 应用层Join（分两次查询，程序拼接） |
| `<` | LEFT JOIN | A LEFT JOIN B ON ... |
| `>` | RIGHT JOIN | A RIGHT JOIN B ON ... |
| `*` | CROSS JOIN | A CROSS JOIN B |
| `&` | INNER JOIN | A INNER JOIN B ON ... (A & B) |
| `\|` / `""` | FULL JOIN | A FULL JOIN B (A \| B) |
| `!` | OUTER JOIN | NOT (A \| B) |
| `^` | SIDE JOIN | NOT (A & B) |
| `(` | ANTI JOIN | A & NOT B |
| `)` | FOREIGN JOIN | B & NOT A |
| `~` | ASOF JOIN | B ~= A (时序数据最近匹配) |

### 4.2 ON 条件表达式 (Join.On)

引用格式：`</User/id@` → User 表 LEFT JOIN，ON 条件为 `User.id = 当前表.userId`

支持的关联后缀：
- `{}` → 一对多关联
- `<>` → 多对一关联
- `$`, `~`, `>=`, `<=`, `>`, `<` → 比较关联

---

## 五、版本路线图

### v8.2.0 (当前版本)

- ✅ 30+ 数据库支持
- ✅ @combine 布尔表达式引擎
- ✅ 12种 JOIN 类型
- ✅ PreparedStatement 预编译
- ✅ 假删除/软删除
- ✅ 远程函数/脚本引擎 (JSR223)
- ✅ 多租户/多数据源
- ✅ SQL缓存机制

### v8.x 短期规划 (优先级从高到低)

| 阶段 | 目标 | 关键任务 |
|------|------|----------|
| P0 | 表达式引擎增强 | @combine 支持 IN/BETWEEN 字面量；允许嵌套深度可配置化 |
| P0 | 安全加固 | 脚本引擎沙箱强制隔离；SQL注入检测增强；@raw权限精细化 |
| P1 | 性能优化 | WITH AS 全数据库覆盖；SQL执行计划缓存；批量操作优化 |
| P1 | 新数据库适配 | 完善 Snowflake/Databricks/DuckDB 适配器 |
| P2 | 开发者体验 | 错误信息中文化+结构化；@explain 返回建议索引；Schema自动生成 |

### v9.0 长期愿景

| 方向 | 愿景 |
|------|------|
| 智能查询优化 | 基于查询历史自动推荐 JOIN 策略、索引提示 |
| GraphQL 融合层 | 在现有JSON协议上叠加GraphQL风格查询能力 |
| 实时推送 | 基于 CDC (Change Data Capture) 的 @subscribe 实时订阅 |
| 多语言 SDK | Go/Rust/Python 版 ORM 内核，共享同一份JSON协议 |
| 可视化管理台 | 内置Admin UI，可视化配置权限/函数/文档 |
| AI 辅助查询 | 自然语言 → APIJSON 请求的智能转换 |

---

## 六、技术债务与优化建议

### 6.1 代码质量

1. **超大类问题**：[AbstractSQLConfig.java](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java) 超过 6000 行，建议拆分：
   - `SQLConditionBuilder` — @combine 表达式引擎
   - `SQLConditionOperators` — gainWhereItem 各操作符
   - `SQLJoinBuilder` — JOIN 处理逻辑
   - `SQLHavingBuilder` — @having 处理

2. **静态状态**：`MAX_COMBINE_DEPTH` 等为 public static，全局可修改，线程安全风险。建议改为实例级可配置。

3. **泛型参数链路过长**：`<T, M extends Map<String,Object>, L extends List<Object>>` 贯穿所有类，可读性差，可考虑引入类型别名或简化。

### 6.2 性能相关

1. `newSQLConfig` 中对 request 的 `remove` + `finally put` 操作在并发场景下需确保传入的是副本。
2. `isKeyInCombineExpr()` 使用 indexOf 循环子串匹配，最坏 O(n²)，可预编译为 token 流。
3. RAW_MAP 查询为 HashMap 线性 containsValue 遍历，大数据量下可优化。

### 6.3 安全性

1. @raw 原始 SQL 片段需强权限管控，建议默认关闭，通过白名单机制启用。
2. 脚本引擎 (IF/Function) 必须配置 ClassFilter 沙箱，文档中须强调。
3. 建议增加请求频率限制和慢查询告警的内建支持。
