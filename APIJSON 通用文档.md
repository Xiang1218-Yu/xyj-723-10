# APIJSON 通用文档

> 基于 APIJSON ORM v8.2.0 源码核对编写
>
> **标注约定**：
> - `[源码]` — 内容可在源码中直接定位验证
> - `[推断]` — 基于源码逻辑的合理推导
> - `[示例]` — 使用示例（非源码直接内容）

---

## 一、快速上手 `[示例]`

### 1.1 请求格式

APIJSON 通过 JSON 描述查询需求，后端自动生成 SQL 并返回 JSON 结果：

```json
// GET /get
{
  "Moment": {
    "id{}": [1, 2, 3],
    "@column": "id,userId,content",
    "@order": "date-"
  }
}
```

生成 SQL `[推断]`：
```sql
SELECT id, userId, content FROM Moment WHERE id IN (1, 2, 3) ORDER BY date DESC
```

---

## 二、请求协议 `[源码]`

### 2.1 请求方法

来源 [RequestMethod.java#L14-L59](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/RequestMethod.java#L14)：

| 方法 | HTTP 语义 | 说明 |
|------|-----------|------|
| GET | 查询 | 常规获取数据 |
| HEAD | 检查 | 非空检查，返回总数 |
| GETS | 安全GET | 通过POST安全获取，不显示请求/返回内容 |
| HEADS | 安全HEAD | 通过POST安全检查 |
| POST | 新增 | 插入数据 |
| PUT | 修改 | 部分更新字段 |
| DELETE | 删除 | 删除数据 |
| CRUD | 批量 | 包含多条增删改查+函数调用 |

### 2.2 请求结构

```
{
  "表名": {
    "字段条件key": "值",
    "@关键字": "关键字值"
  },
  "[]": {             // 数组/分页
    "page": 0,
    "count": 10
  }
}
```

---

## 三、条件操作符 `[源码]`

### 3.1 完整操作符对照表

来源 [gainWhereItem()](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L3897)：

| 后缀 | keyType | SQL | 值类型 | 示例 |
|------|---------|-----|--------|------|
| (无) | 0 | `=` | Boolean/Number/String | `"id": 1` → `id = 1` |
| `!` | 0 | `!=` / `IS NOT NULL` | 任意 | `"sex!": 0` → `sex != 0`; `"name!": null` → `name IS NOT NULL` |
| `$` | 1 | `LIKE` | String | `"name$": "a"` → `name LIKE '%a%'` `[推断]` |
| `~` | 2 | `REGEXP` / `~`(PG) | String/String[] | `"name~": "^[a-z]+"` → 正则匹配 |
| `*~` | -2 | 忽略大小写正则 | String/String[] | `"name*~": "abc"` → PG:`name ~* ?`; MySQL8:`regexp_like(name, ?, 'i')` |
| `%` | 3 | `BETWEEN ... AND ...` | String(逗号分隔) | `"date%": "2024-01-01,2024-12-31"` |
| `{}` | 4 | `IN (...)` | Array/Collection | `"id{}": [1,2,3]` → `id IN (1,2,3)` |
| `}{` | 5 | `EXISTS (子查询)` | Subquery | `"userId}{": "/User/id"` `[推断]` |
| `<>` | 6 | JSON数组包含 | Array | `"tagIdList<>": [1,2]` |
| `>=` | 7 | `>=` | Number/String | `"id>=": 100` |
| `<=` | 8 | `<=` | Number/String | `"id<=": 200` |
| `>` | 9 | `>` | Number/String/Date | `"id>": 50` |
| `<` | 10 | `<` | Number/String/Date | `"id<": 100` |

> **注意**`[源码]`：`~` 正则在不同数据库的实现不同（[gainRegExpString() L4304](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L4304)）：
> - PostgreSQL: `key ~ ?`（`*~`→`key ~* ?`）
> - MySQL 8+/Oracle/Dameng/KingBase: `regexp_like(key, ?, 'c')`（`*~`→`'i'`）
> - ClickHouse: `match(key, ?)`（`*~`→`match(lower(key), lower(?))`）
> - Elasticsearch: `key RLIKE ?`
> - Hive: `key REGEXP ?`
> - Presto/Trino: `regexp_like(key, ?)`

### 3.2 长度函数前缀 `[源码]`

来源 [gainKey() L4014](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L4014)：

| 前缀 | SQL函数 | 语义 | 示例 |
|------|---------|------|------|
| `key[` | `length(key)` / `datalength(key)` (SQL Server) | 字符串长度 | `"name[>": 5` → `length(name) > 5` |
| `key{` | `json_length(key)` | JSON数组长度 | `"tagList{>=": 1` → `json_length(tagList) >= 1` |

### 3.3 NULL 判断 `[源码]`

来源 [gainEqualString() L3995](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L3995)：

| 写法 | SQL |
|------|-----|
| `"name": null` | `name IS NULL` |
| `"name!": null` | `name IS NOT NULL` |
| `"@null": "tag,pictureList"` | 将指定 key 设为 null（SET 或 IS NULL 条件） |

### 3.4 多值逻辑连接（同 key 多值）`[源码]`

来源 [Logic.java](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/Logic.java)：操作符后缀后可追加 `&`/`|` 控制多值间 AND/OR 连接：

```json
{ "name&$": ["a", "b"] }
```

`[推断]` 这会生成类似 `name LIKE '%a%' AND name LIKE '%b%'` 的条件。

---

## 四、`@combine` 条件组合 `[源码]`

### 4.1 两种模式

`@combine` 支持两种格式，由是否含逗号自动判断（[L5803-L5804](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L5803)）：

#### 模式一：简单列表模式

```json
"@combine": "key0,&key1,|key2,!key3"
```

前缀语义：
- `&key` → AND 组
- `|key` → OR 组（无前缀默认 OR 组）
- `!key` → NOT 组

**限制**`[源码]`：PUT 请求禁止使用 `|key` 和 `!key`（[L5901-L5912](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L5901)）。

组合顺序固定：AND组 → OR组 → NOT组，组间 AND 连接。

#### 模式二：布尔表达式模式（5.0+ 推荐）

```json
"@combine": "date> | (contactIdList<> & !(name~ | tag$))"
```

### 4.2 表达式语法严格规则 `[源码]`

来源 [parseCombineExpression()](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L3364)：

| 规则 | 正确 | 错误 |
|------|------|------|
| AND/OR 两侧各一个空格 | `a & b` | `a&b`, `a &b`, `a& b` |
| NOT 紧跟 key/(，无空格 | `!a`, `!(a \| b)` | `! a`, `! (a \| b)` |
| 括号内侧无空格 | `(a & b)` | `( a & b )` |
| 首尾无空格 | `a \| b` | ` a \| b`, `a \| b ` |
| 无连续空格 | `a & b` | `a  &  b` |
| NOT 右侧禁止空格或 ) 或 & 或 \| | `!a`, `!(a \| b)` | `! )`, `! &` |
| 左括号前必须有 &\| | `a & (b \| c)` | `a (b \| c)` |

### 4.3 安全限制 `[源码]`

来源 [AbstractSQLConfig.java#L61-L67](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L61)：

| 限制 | 默认值 | 可配置 |
|------|--------|--------|
| WHERE 条件总数 | ≤ 10（MAX_WHERE_COUNT） | public static |
| HAVING 条件总数 | ≤ 5（MAX_HAVING_COUNT） | public static |
| 括号嵌套深度 | ≤ 2（MAX_COMBINE_DEPTH） | public static |
| 表达式内 key 数量 | ≤ 5（MAX_COMBINE_COUNT） | public static |
| 同 key 引用次数 | ≤ 2（MAX_COMBINE_KEY_COUNT） | public static |
| 表达式key/总条件比 | ≤ 1.0（MAX_COMBINE_RATIO） | public static |

### 4.4 未引用条件的处理 `[源码]`

布尔表达式中**未被引用的 WHERE 条件**会自动 AND 追加到表达式外层（[L3613-L3644](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L3613)）。

PreparedStatement 参数顺序`[源码]`（[L3629-L3650](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L3629)）：
1. AND 条件值（未被表达式引用的条件）
2. 表达式内条件值

### 4.5 @combine 错误排查 `[推断]`

| 错误信息关键词 | 原因 |
|---------------|------|
| "不允许首尾有空格" | 表达式开头或结尾有空格 |
| "左右必须各一个相邻空格" | &\| 前后空格数不对 |
| "左括号...右边不允许有相邻空格" | `( ` 括号后有空格 |
| "右括号...左边不允许有相邻空格" | ` )` 括号前有空格 |
| "左边缺少 & \| 逻辑连接符" | 两个条件间缺少连接符 |
| "括号嵌套层级...超过最大值" | 超过 MAX_COMBINE_DEPTH=2 |
| "重复引用...超过最大值" | 同 key 出现超过 2 次 |
| "key 数量...超过最大值" | 表达式中超过 5 个 key |
| "对应的条件键值对...不存在" | 表达式中的 key 在 WHERE 条件中找不到 |

---

## 五、系统关键词参考 `[源码]`

来源 [JSONMap.java#L166-L242](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/JSONMap.java#L166) 中定义的 KEY_ 常量及 TABLE_KEY_LIST：

### 5.1 数据源/配置类

| 关键词 | 语义 | 值类型 |
|--------|------|--------|
| `@database` | 数据库类型 | String（35种白名单之一） |
| `@datasource` | 数据源标识 | String |
| `@namespace` | 命名空间 | String |
| `@catalog` | 目录 | String |
| `@schema` | 数据库模式 | String |
| `@role` | 当前角色 | String（UNKNOWN/LOGIN/CONTACT/CIRCLE/OWNER/ADMIN） |
| `@explain` | 是否分析SQL | Boolean（仅 DEBUG 模式可用） |
| `@cache` | 缓存策略 | String（RAM/ROM/ALL） |

### 5.2 查询控制类

| 关键词 | 语义 | 值类型 |
|--------|------|--------|
| `@column` | 查询字段/函数 | String: `"col0,col1;fun0(col0);fun1(col1):alias"` |
| `@from` | FROM子查询 | Subquery对象 |
| `@combine` | 条件组合 | String（见第四节） |
| `@group` | 分组字段 | String |
| `@having` | 聚合条件 | String或Map（见第七节） |
| `@having&` | 聚合条件(AND模式) | String |
| `@order` | 排序方式 | String |
| `@sample` | 取样方式 | String |
| `@latest` | 最近方式 | String |
| `@partition` | 分区方式 | String |
| `@fill` | 填充方式 | String |
| `@key` | 字段表达式映射 | String或Map |
| `@raw` | 原始SQL片段标记 | String（逗号分隔key列表） |

### 5.3 值处理类

| 关键词 | 语义 | 值类型 |
|--------|------|--------|
| `@null` | 设为null的字段 | String（逗号分隔） |
| `@cast` | 类型转换 | String: `"key0:type0,key1:type1"` |
| `@json` | 转JSON输出 | String（逗号分隔字段名） |
| `@string` | 转String输入 | String（逗号分隔字段名） |
| `@trim` | 去除首尾空白 | String（逗号分隔字段名） |

### 5.4 方法控制类

| 关键词 | 语义 | 值类型 |
|--------|------|--------|
| `@method` | 对象内操作方法 | String |
| `@get`/`@gets`/`@head`/`@heads`/`@post`/`@put`/`@delete` | 子对象方法 | 对应配置对象 |

### 5.5 Parser 级关键词（不在 TABLE_KEY_LIST 中）`[源码]`

以下关键词在 [JSONMap.java](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/JSONMap.java#L166) 中定义但未列入 TABLE_KEY_LIST（由 Parser 层处理而非 newSQLConfig）：

| 关键词 | 语义 |
|--------|------|
| `@try` | 尝试执行，忽略异常 |
| `@catch` | 捕获异常处理方式 |
| `@drop` | 丢弃不返回 |
| `@default` | 自定义默认值 |

---

## 六、JOIN 连表查询 `[源码]`

### 6.1 JOIN 类型

来源 [Join.java#L21](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/Join.java#L21) 和 [concatJoinWhereString()](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L3753)：

| 符号 | 写法 | 类型 | SQL 语义 |
|------|------|------|----------|
| `@` | `"@/User"` | APP JOIN | 应用层拼接（两次查询） |
| `<` | `"</User"` | LEFT JOIN | A LEFT JOIN B |
| `>` | `">/User"` | RIGHT JOIN | A RIGHT JOIN B |
| `*` | `"*/User"` | CROSS JOIN | A CROSS JOIN B |
| `&` | `"&/User"` | INNER JOIN | A INNER JOIN B |
| `\|`/`""` | `"/User"` 或 `"\|/User"` | FULL JOIN | A FULL JOIN B |
| `!` | `"!/User"` | OUTER JOIN | NOT (A \| B) |
| `^` | `"^/User"` | SIDE JOIN | NOT (A & B) |
| `(` | `"(/User"` | ANTI JOIN | A AND NOT B |
| `)` | `")/User"` | FOREIGN JOIN | NOT A AND B |
| `~` | `"~/User"` | ASOF JOIN | 时序最近匹配 |

### 6.2 ON 关联条件 `[示例]`

```json
{
  "Moment": {},
  "join": {
    "</User/id@": {
      "@column": "id,name"
    }
  }
}
```

`[推断]` 生成：`Moment LEFT JOIN User ON User.id = Moment.userId`（通过 `id@` 引用路径 `/User/id`，Parser 自动将 `userId` 映射到 `User.id`）

---

## 七、子查询 `[源码]`

来源 [Subquery.java](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/Subquery.java)：

| 字段 | 语义 |
|------|------|
| path | 引用路径 |
| from | 是否 FROM 子查询 |
| range | 范围（ALL/ANY） |
| key | 返回字段 |
| config | 子查询 SQLConfig |

`[示例]` 子查询作为条件值：
```json
{ "userId}{": { "from": "User", "where": { "sex": 1 } } }
```
`[推断]` → `EXISTS (SELECT * FROM User WHERE sex = 1)`

---

## 八、远程函数 `[源码]`

来源 [AbstractFunctionParser.java](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractFunctionParser.java) 和 [Operation.java#L118-L138](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/Operation.java#L118)：

- 函数调用格式：`key()` 后缀，例如 `"name()": "getUserName(id)"`
- IF 条件脚本：`"条件表达式": "throw new Error('...')"`
- 需启用 `AbstractFunctionParser.ENABLE_SCRIPT_FUNCTION = true`
- JDK 8-13 自带 Nashorn JS 引擎，其它版本需外部引擎依赖

---

## 九、权限与角色 `[源码]`

### 9.1 六种角色

来源 [AbstractVerifier.java#L76-L96](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractVerifier.java#L76)：

| 角色 | 常量 | 说明 |
|------|------|------|
| UNKNOWN | 未登录 | 不明身份用户 |
| LOGIN | 已登录 | 已登录用户（自动注入 userId>0 条件） |
| CONTACT | 联系人 | userId{} 在 contactIdList 中 |
| CIRCLE | 圈子成员 | CONTACT + OWNER，通过 verifyCircle() 校验 |
| OWNER | 拥有者 | userId = 当前记录.userId |
| ADMIN | 管理员 | 通过 verifyAdmin() 校验，默认不支持需子类重写 |

### 9.2 Operation 访问控制

来源 [Operation.java](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/Operation.java)：

| 操作 | 格式 | 语义 |
|------|------|------|
| MUST | `"key0,key1"` | 必须传入的字段 |
| REFUSE | `"key0,key1"` | 不允许传入的字段 |
| TYPE | `{ "key": "NUMBER" }` | 字段类型校验 |
| VERIFY | `{ "key~": "PHONE" }` | 正则/范围校验 |
| EXIST | `"key0,key1"` | 联合唯一性校验 |
| UNIQUE | `"key0,key1"` | 不存在校验（排除自身） |
| INSERT | `{ ... }` | 不存在时插入 |
| UPDATE | `{ ... }` | 存在则更新不存在则插入 |
| REPLACE | `{ ... }` | 存在时替换 |
| REMOVE | `"key0"` | 存在时移除 |
| IF | `"condition": "code"` | 条件脚本 |
| ALLOW_PARTIAL_UPDATE_FAIL | Boolean | 允许批量部分失败 |
| IS_ID_CONDITION_MUST | Boolean | 强制要求 id/id{} 条件 |

---

## 十、数据库支持 `[源码]`

### 10.1 已注册白名单（35种）

来源 [DATABASE_LIST](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L146)：

| 类别 | 数据库 |
|------|--------|
| 关系型 | MYSQL, POSTGRESQL, SQLSERVER, ORACLE, DB2, MARIADB, TIDB, COCKROACHDB, DAMENG, KINGBASE, DUCKDB, OPENGAUSS, SQLITE(常量定义但未列入白名单) |
| 分析型 | CLICKHOUSE, HIVE, PRESTO, TRINO, DORIS, STARROCKS, SNOWFLAKE, DATABEND, DATABRICKS |
| 搜索 | ELASTICSEARCH, MANTICORE |
| 时序 | INFLUXDB, TDENGINE, TIMESCALEDB, QUESTDB, IOTDB |
| 向量 | MILVUS |
| NoSQL | REDIS, MONGODB, CASSANDRA, SURREALDB |
| 消息 | KAFKA, MQ |

`@database` 值必须为以上 35 种之一（SQLITE 虽有常量定义但不在白名单中，`[源码]`见 [L5490-L5492](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L5490) 校验逻辑）。

---

## 十一、@having 聚合条件 `[源码]`

来源 [L6058-L6148](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L6058)：

### 11.1 String 格式

```json
"@having": "sum(balance)>100;count(id)<10"
```

- 用 `;` 分隔多个条件
- 每个条件必须包含 SQL 函数（`count`, `sum`, `avg`, `max`, `min` 等）
- 5.0+ 默认 OR 连接；使用 `@having&` 强制 AND

### 11.2 Map 格式

```json
"@having": {
  "sumBalance": "sum(balance)>100",
  "countId": "count(id)<10",
  "@combine": "sumBalance & countId"
}
```

- key 为条件别名，value 为含函数的条件字符串
- 内置 `@combine` 支持布尔表达式组合

### 11.3 兼容配置 `[源码]`

- `IS_HAVING_DEFAULT_AND = false`（[L35](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L35)）：设为 true 兼容 5.0 前 AND 默认
- `IS_HAVING_ALLOW_NOT_FUNCTION = false`（[L40](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L40)）：设为 true 允许不含函数的表达式

---

## 十二、@column 字段选择 `[源码]`

来源 [L6010-L6055](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L6010)：

### 12.1 格式

```
@column = "field0,field1;function0(field0):alias0;function1(field0,field1);DISTINCT field2"
```

- `;` 分隔段
- 不含 `(` 的段 → 按空格/逗号分割为字段列表
- 含 `(` 的段 → 直接作为 SQL 函数表达式
- 以 `DISTINCT ` 开头 → SELECT DISTINCT

`[示例]`
```json
"@column": "id,userId;count(id):total;left(name,3):namePrefix"
```
`[推断]` → `SELECT id, userId, count(id) AS total, left(name,3) AS namePrefix`

---

## 十三、@key 字段映射 `[源码]`

来源 [L6150-L6169](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L6150)：

String 格式：
```json
"@key": "year:left(date,4);name_tag:(name,tag)"
```

Map 格式：
```json
"@key": { "year": "left(date,4)", "name_tag": "(name,tag)" }
```

请求中使用 `year>2024` 实际映射到 `left(date,4) > 2024`。

---

## 十四、假删除/软删除 `[源码]`

来源 [L5835-L5881](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L5835) 和 [L5987-L6008](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L5987)：

- 配置 `ACCESS_FAKE_DELETE_MAP` 为表指定 `deletedKey`/`deletedValue`/`notDeletedValue`
- 非 DELETE 查询自动追加 `deletedKey != deletedValue` 或 `deletedKey = notDeletedValue`
- DELETE 请求自动转为 PUT，设置 deletedKey=deletedValue
- 子类重写 `isFakeDelete()` 返回 true 启用

---

## 附录 A：请求示例 `[示例]`

### 复杂条件查询

```json
{
  "Moment": {
    "date>": "2024-01-01",
    "date<": "2024-12-31",
    "userId{}": [10, 20, 30],
    "content$": "APIJSON",
    "praiseList{>=": 1,
    "@column": "id,userId,content,date",
    "@combine": "date> & date< & (userId{} | (content$ & praiseList{>=))",
    "@order": "date-",
    "@group": "userId"
  },
  "[]": {
    "page": 0,
    "count": 20
  }
}
```

### JOIN 查询

```json
{
  "Moment": {
    "@column": "id,content"
  },
  "join": {
    "</User/id@": {
      "@column": "id,name,head",
      "sex": 1
    },
    ">/Comment/momentId@": {
      "@column": "id,content",
      "@order": "date-",
      "@combine": "content$"
    }
  }
}
```
