# APIJSON 通用文档

> 版本：v8.2.0 | 基于 ORM 内核源码深度整理

---

## 目录

1. [快速开始](#一快速开始)
2. [请求协议](#二请求协议)
3. [条件操作符](#三条件操作符)
4. [@combine 条件表达式引擎](#四combine-条件表达式引擎)
5. [关键词 (Keywords)](#五关键词-keywords)
6. [连表查询 (JOIN)](#六连表查询-join)
7. [子查询 (Subquery)](#七子查询-subquery)
8. [远程函数与脚本](#八远程函数与脚本)
9. [权限与安全](#九权限与安全)
10. [多数据库支持](#十多数据库支持)

---

## 一、快速开始

### 1.1 什么是 APIJSON

APIJSON 是一个 **零代码 RESTful API 引擎**。你只需定义数据库表结构，前端通过 JSON 描述查询需求，后端自动生成 SQL 并返回 JSON 结果，无需编写任何 Controller/Service/DAO 代码。

### 1.2 Hello World

**请求：** 查询 id=1 的 User

```json
{
  "User": {
    "id": 1
  }
}
```

**自动生成 SQL：**
```sql
SELECT * FROM `User` WHERE `id` = 1 LIMIT 10
```

**响应：**
```json
{
  "User": {
    "id": 1,
    "name": "Tommy",
    "sex": 0,
    "picture": "https://..."
  },
  "ok": true
}
```

---

## 二、请求协议

### 2.1 HTTP 方法

| 方法 | 对应 SQL | 语义 |
|------|----------|------|
| GET | SELECT | 查询单条/多条记录 |
| HEAD | SELECT COUNT | 仅返回总数，不返回数据 |
| POST | INSERT | 新增记录 |
| PUT | UPDATE | 更新记录 |
| DELETE | DELETE (或假删除) | 删除记录 |
| CRUD | 动态指定 | JSON 中通过 `"@method":"POST"` 等指定 |

### 2.2 请求结构

```json
{
  "@role": "ADMIN",
  "@database": "MYSQL",
  "@schema": "mydb",

  "Moment": {
    "@column": "id,userId,content",
    "@order": "date-",
    "@count": 10,
    "@page": 0,

    "userId": 1,
    "date>": "2024-01-01",
    "content$": "%hello%"
  }
}
```

### 2.3 响应结构

```json
{
  "ok": true,
  "code": 200,
  "msg": "success",
  "Moment": [{ ... }],
  "total": 100,
  "count": 10,
  "time": 1700000000000
}
```

---

## 三、条件操作符

### 3.1 比较运算

| 后缀 | SQL 等价 | 示例 | 说明 |
|------|----------|------|------|
| (无) | `=` | `"id": 1` | 等于 |
| `!` | `!=` | `"sex!": 0` | 不等于 |
| `>` | `>` | `"age>": 18` | 大于 |
| `<` | `<` | `"age<": 65` | 小于 |
| `>=` | `>=` | `"age>=": 18` | 大于等于 |
| `<=` | `<=` | `"age<=": 65` | 小于等于 |

### 3.2 模糊匹配

| 后缀 | SQL 等价 | 示例 | 说明 |
|------|----------|------|------|
| `$` | `LIKE` | `"name$": "a"` | LIKE '%a%' |
| `%$` | `LIKE 'x%'` | `"name%$": "a"` | 前缀匹配 LIKE 'a%' |
| `_$` | `LIKE '_x'` | `"name_$": "a"` | 单字符后缀匹配 |
| `~` | `REGEXP` | `"name~": "^[A-Z]"` | 正则匹配 |
| `*~` | `REGEXP (ignore case)` | `"name*~": "^[a-z]"` | 忽略大小写正则 |

`$` 后缀的占位符规则：
- `key$:"a"` → `LIKE '%a%'`（包含）
- `key%$:"a"` → `LIKE 'a%'`（以...开头）
- `key_$:"a"` → `LIKE '_a'`（倒数第二字符匹配）
- `key%_$:"a"` → `LIKE 'a%_'`（复合占位）

### 3.3 范围与集合

| 后缀 | SQL 等价 | 示例 | 说明 |
|------|----------|------|------|
| `{}` + 数组 | `IN (...)` | `"id{}": [1,2,3]` | 在集合中 |
| `{}` + 字符串 | 条件链 | `"id{}": ">0;<=100"` | 多条件 AND 连接 |
| `\|{}` + 数组 | `IN (...)` (OR) | `"id\|{}": [1,2,3]` | OR 语义的 IN |
| `!{}` + 数组 | `NOT IN (...)` | `"id!{}": [4,5]` | 不在集合中 |
| `%` | `BETWEEN` | `"age%": "18,65"` | 在闭区间内 |
| `}{` | `EXISTS` | `"id}{": {"Comment":{...}}` | EXISTS 子查询 |

`key{}` 的字符串格式支持：
- 分号分隔多条件：`"id{}": ">0;<=1000;!=500"`
- `=null` → `IS NULL`
- `!=null` → `IS NOT NULL`
- 函数条件：`"length(name)<=10"`
- 算术表达式：`"+3*2<=10"`

### 3.4 JSON 操作

| 后缀 | SQL 等价 | 示例 | 说明 |
|------|----------|------|------|
| `<>` + 数组/值 | JSON包含 | `"tagList<>": "tech"` | JSON数组包含元素 |
| `key[` | `length(key)` | `"name[": ">0"` | 字符串长度比较 |
| `key{` | `json_length(key)` | `"images{": ">0"` | JSON数组长度比较 |

### 3.5 逻辑后缀

所有操作符 key 末尾可附加逻辑符：

| 后缀 | 语义 | 示例 |
|------|------|------|
| `&` | 同 key 多值 AND | `"name&$": ["a","b"]` |
| `\|` | 同 key 多值 OR | `"name\|$": ["a","b"]` |
| `!` | NOT 取反 | `"name!$": "a"` |

---

## 四、@combine 条件表达式引擎

### 4.1 概述

`@combine` 是 APIJSON 的条件组合引擎，允许用布尔表达式精确控制 WHERE 子句中各条件间的逻辑关系。

两种模式：
1. **简单列表模式**（兼容旧版）：逗号分隔，前缀标记逻辑
2. **布尔表达式模式**（5.0+推荐）：完整的 &\|!()+括号表达式

### 4.2 简单列表模式

```json
{
  "Moment": {
    "id{}": [1,2,3],
    "sex": 0,
    "name$": "a",
    "@combine": "id{},&sex,!name$"
  }
}
```

规则：
- `&key` → key 在 AND 组
- `|key` → key 在 OR 组
- `!key` → key 在 NOT 组
- 无前缀 → 归入 OR 组
- PUT 请求不允许 `|key` 或 `!key`

生成的 SQL：
```sql
WHERE (id IN (1,2,3)) AND (sex = 0) AND NOT (name LIKE '%a%')
```

### 4.3 布尔表达式模式

```json
{
  "Moment": {
    "date>": "2024-01-01",
    "contactIdList<>": [1,2],
    "name*~": "a",
    "tag&$": "%a%",
    "@combine": "date> | (contactIdList<> & (name*~ | tag&$))"
  }
}
```

生成的 SQL：
```sql
WHERE (date > '2024-01-01')
   OR (contactIdList LIKE '%1%' AND (name REGEXP 'a' OR tag LIKE '%a%'))
```

### 4.4 语法规则

| 规则 | 说明 |
|------|------|
| `&` 两侧必须各一个空格 | `a & b` ✅ / `a&b` ❌ / `a &b` ❌ |
| `\|` 两侧必须各一个空格 | `a \| b` ✅ |
| `!` 紧跟 key 或 `(` 无空格 | `!a` ✅ / `!(a \| b)` ✅ / `! a` ❌ |
| `(` 右侧不允许空格 | `(a` ✅ / `( a` ❌ |
| `)` 左侧不允许空格 | `a)` ✅ / `a )` ❌ |
| 不允许首尾空格 | 整个表达式不能以空格开头/结尾 |

### 4.5 安全限制

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `MAX_COMBINE_DEPTH` | 2 | 最大括号嵌套层级 |
| `MAX_COMBINE_COUNT` | 5 | 表达式中最多引用的 key 数量 |
| `MAX_COMBINE_KEY_COUNT` | 2 | 单个 key 最多被引用次数 |
| `MAX_COMBINE_RATIO` | 1.0 | 引用key数 / 总条件数 上限 |
| `ALLOW_MISSING_KEY_4_COMBINE` | true | 允许表达式中引用不存在的key |

这些限制通过 `AbstractSQLConfig` 的 public static 变量配置，也可通过子类重写对应的 `getMaxXxx()` 方法在实例级别调整。

### 4.6 额外规则

- `id`, `id{}`, `userId`, `userId{}` **不允许**出现在 @combine 表达式中（它们始终强制 AND）
- 表达式中未引用的条件 key，会以 AND 方式连接到表达式结果之后
- PreparedStatement 模式下，值的顺序为：先 AND 条件值，后表达式内条件值
- @having 同样支持 @combine：`"@having": { "avg(id)>": "100", "count(0)>": "5", "@combine": "avg(id)> & count(0)>" }`

### 4.7 错误场景与排查

| 错误信息 | 原因 | 修复 |
|----------|------|------|
| 不允许首尾有空格 | 表达式有多余空格 | 去掉首尾空格，规范 &\| 两侧空格 |
| 空格左边缺少条件key | &\| 前缺少条件 | 检查表达式完整性 |
| 左括号比右括号多/少 | 括号不匹配 | 检查括号配对 |
| key数量已超过最大值 | 超过 MAX_COMBINE_COUNT | 减少条件或增大限制 |
| 重复引用次数超过最大值 | 同key引用超2次 | 去重或增大 MAX_COMBINE_KEY_COUNT |
| 条件键值对不存在 | 表达式引用了where中没有的key | 添加对应key:value 或设置 ALLOW_MISSING_KEY_4_COMBINE |

---

## 五、关键词 (Keywords)

所有 `@` 开头的 key 为系统关键词，在 [JSONMap.java](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Charmander/APIJSONORM/src/main/java/apijson/JSONMap.java#L166) 中定义：

### 5.1 数据源与配置

| 关键词 | 类型 | 说明 |
|--------|------|------|
| `@database` | String | 数据库类型：MYSQL/POSTGRESQL/ORACLE/... |
| `@datasource` | String | 数据源名称（多数据源场景） |
| `@namespace` | String | 命名空间 |
| `@catalog` | String | Catalog |
| `@schema` | String | Schema |
| `@role` | String | 角色：UNKNOWN/LOGIN/CONTACT/CIRCLE/OWNER/ADMIN |
| `@explain` | Boolean | true 时返回 SQL 执行计划（仅 DEBUG 模式） |
| `@cache` | String/Int | 缓存策略：RAM/ROM/ALL |

### 5.2 查询控制

| 关键词 | 类型 | 说明 |
|--------|------|------|
| `@column` | String | 返回字段/函数：`"id,name;count(id):total"` |
| `@from` | Subquery | 子查询作为表来源 |
| `@combine` | String | 条件组合表达式（见第四章） |
| `@group` | String | GROUP BY：`"type,sex"` |
| `@having` | String/Map | HAVING 条件 |
| `@having&` | String | HAVING 强制 AND 连接 |
| `@order` | String | 排序：`"date-,id+"` (+升序/-降序) |
| `@count` | Int | 每页条数（默认10，最大100） |
| `@page` | Int | 页码（从0开始，可配置从1开始） |
| `@query` | Int | 查询类型：0-总数和数据，1-总数，2-数据 |
| `@sample` | String | 取样（ClickHouse等） |
| `@latest` | String | 最近N条 |
| `@partition` | String | 分区（Hive等） |
| `@fill` | String | 填充（时序数据库） |
| `@distinct` | Boolean | DISTINCT 去重 |

### 5.3 数据操作

| 关键词 | 类型 | 说明 |
|--------|------|------|
| `@null` | String | 设为NULL的字段：`"tag,pictureList"` |
| `@cast` | String | 类型转换：`"date:DATE,price:DECIMAL"` |
| `@key` | String/Map | 字段映射/表达式：`"year:left(date,4)"` |
| `@raw` | String | 原始SQL片段白名单 |
| `@json` | String | 将字段转为 JSON 输出 |
| `@method` | String | JSON对象内部指定HTTP方法 |
| `@get`/`@gets` | - | 子对象强制GET方法 |
| `@post`/`@put`/`@delete` | - | 子对象指定方法 |

### 5.4 @column 格式

```
"@column": "key0,key1,key2;fun0(key0,key1):alias0;fun1(key2):alias1"
```

- `,` 分隔同组字段
- `;` 分隔不同组（字段组 / 函数组）
- `:` 后跟别名
- 支持函数：`count`, `sum`, `max`, `min`, `avg`, `length`, `left`, `substring`, `concat`, ...
- `DISTINCT` 前缀去重：`"@column": "DISTINCT name"`

### 5.5 @order 格式

```
"@order": "key0+,key1-,key2"
```

- `+` 升序 ASC（默认）
- `-` 降序 DESC
- 逗号分隔多字段排序优先级

---

## 六、连表查询 (JOIN)

### 6.1 JOIN 类型总表

| 符号 | 类型 | 语义 | SQL |
|------|------|------|-----|
| `@/` | APP JOIN | 应用层关联（非SQL JOIN） | 两次查询程序拼接 |
| `</` | LEFT JOIN | 左外连接 | A LEFT JOIN B |
| `>/` | RIGHT JOIN | 右外连接 | A RIGHT JOIN B |
| `*/` | CROSS JOIN | 交叉连接 | A CROSS JOIN B |
| `&/` | INNER JOIN | 内连接 | A INNER JOIN B |
| `\|/` | FULL JOIN | 全外连接 | A FULL JOIN B |
| `!/` | OUTER JOIN | 外连接（两边独有） | NOT(A \| B) |
| `^/` | SIDE JOIN | 边缘连接 | NOT(A & B) |
| `(/` | ANTI JOIN | 反连接（A有B没有） | A & NOT B |
| `)/` | FOREIGN JOIN | 外键反连接（B有A没有） | B & NOT A |
| `~/` | ASOF JOIN | 时序最近连接 | B ~= A |

### 6.2 基本用法

```json
{
  "Moment": {
    "id": 1,
    "User@": {
      "id@": "/Moment/userId"
    }
  }
}
```

或使用 JOIN 语法：

```json
{
  "Moment": {
    "@column": "id,content",
    "join": {
      "</User": {
        "@column": "id,name",
        "name~": "a",
        "@combine": "name~",
        "@order": "id-"
      }
    },
    "userId{}": ">0"
  }
}
```

生成 SQL：
```sql
SELECT Moment.id, Moment.content, User.id AS User_id, User.name AS User_name
FROM Moment
LEFT JOIN User ON User.id = Moment.userId AND User.name REGEXP 'a'
WHERE Moment.userId > 0
ORDER BY User.id DESC
LIMIT 10
```

### 6.3 ON 关联条件

在 join 的 key 中使用 `/Table/refKey@` 格式指定关联：

```json
{
  "Comment": {
    "join": {
      "</User": {
        "id@": "/Comment/userId"
      }
    }
  }
}
```

ON 条件支持的后缀：

| 后缀 | 关联类型 |
|------|----------|
| `@` (无后缀) | 一对一 `=` |
| `{}@` | 一对多 `IN` |
| `<>@` | 多对一 |
| `$@` | LIKE 关联 |
| `~@` | REGEXP 关联 |
| `>=@`, `<=@`, `>@`, `<@` | 比较关联 |

### 6.4 APP JOIN vs SQL JOIN

**APP JOIN (`@/`)**：
- 先查主表，再用主表结果的字段值查询副表
- 不依赖数据库 JOIN 能力
- 支持所有数据库
- 副表条件无法在 ON 中过滤
- 适合一对多、多对一场景

**SQL JOIN (`</`, `&/` 等)**：
- 单条 SQL 完成，性能更好
- 副表条件可在 ON 中过滤
- 数据库需支持对应 JOIN 语法
- 复杂嵌套可能产生性能问题

---

## 七、子查询 (Subquery)

### 7.1 基本子查询

```json
{
  "Moment": {
    "userId}{": {
      "User": {
        "@column": "id",
        "sex": 1
      }
    }
  }
}
```

生成 SQL：
```sql
SELECT * FROM Moment WHERE userId EXISTS (SELECT id FROM User WHERE sex = 1)
```

### 7.2 IN 子查询

```json
{
  "Moment": {
    "userId{}": {
      "from": "User",
      "User": {
        "@column": "id",
        "sex": 1
      }
    }
  }
}
```

生成：
```sql
WHERE userId IN (SELECT id FROM User WHERE sex = 1)
```

### 7.3 FROM 子查询

```json
{
  "Moment": {
    "@from": {
      "from": "Moment",
      "Moment": {
        "@column": "id,userId",
        "date>": "2024-01-01"
      }
    },
    "id{}": ">0"
  }
}
```

---

## 八、远程函数与脚本

### 8.1 远程函数

通过 `AbstractFunctionParser` 支持在请求中调用服务端函数：

```json
{
  "Moment": {
    "id": "verify(http://example.com/verify)"
  }
}
```

### 8.2 IF 条件脚本

通过 `@if` 或 `Operation.IF` 支持条件脚本：

```json
{
  "@role": "ADMIN",
  "User": {
    "id": 1,
    "name": "newName",
    "@if": {
      "sex != 0 && sex != 1": "throw new Error('sex must be 0 or 1')",
      "ELSE": ""
    }
  }
}
```

支持的脚本引擎：
- JavaScript (Nashorn, JDK 8-13)
- JSR223 兼容引擎（Groovy, Python, Lua 等）

**安全警告：**
- 必须启用 `AbstractFunctionParser.ENABLE_SCRIPT_FUNCTION = true`
- 必须配置 `ClassFilter` 防止脚本注入
- JDK 14+ 需外部脚本引擎依赖
- 强烈建议在沙箱环境中运行

---

## 九、权限与安全

### 9.1 角色体系 (RequestRole)

| 角色 | 常量 | 说明 |
|------|------|------|
| UNKNOWN | 0 | 未登录 |
| LOGIN | 1 | 已登录 |
| CONTACT | 2 | 联系人 |
| CIRCLE | 3 | 圈子成员 |
| OWNER | 4 | 所有者（自己的数据） |
| ADMIN | 5 | 管理员 |

### 9.2 系统表

| 表名 | 作用 |
|------|------|
| `Access` | API访问权限控制（角色、方法、频率限制） |
| `Request` | 请求结构校验（MUST/REFUSE/TYPE/VERIFY规则） |
| `Table` | 表别名映射、假删除配置 |
| `Column` | 字段别名、类型、校验规则 |
| `Function` | 远程函数注册 |
| `Document` | API文档自动生成 |

### 9.3 Operation 校验规则

在 Request 表中配置：

| Operation | 格式 | 说明 |
|-----------|------|------|
| MUST | `"key0,key1"` | 必须传的字段 |
| REFUSE | `"key0,key1"` | 禁止传的字段 |
| TYPE | `{ "key": "NUMBER" }` | 类型校验 |
| VERIFY | `{ "key~": "PHONE" }` | 正则/条件校验 |
| EXIST | `"key0,key1"` | 联合校验存在性 |
| UNIQUE | `"key0,key1"` | 联合唯一性校验 |
| INSERT | `{ "key": defaultValue }` | 不存在时插入默认值 |
| UPDATE | `{ "key": value }` | 强制设置值 |
| REPLACE | `{ "key": value }` | 替换值 |
| REMOVE | `"key0,key1"` | 移除字段 |

支持的类型：BOOLEAN, NUMBER, DECIMAL, STRING, URL, DATE, TIME, DATETIME, OBJECT, ARRAY，及它们的数组形式如 NUMBER[]

支持的 VERIFY 正则：PHONE, EMAIL, PASSWORD, ID_CARD, BANK_CARD 等，在 `AbstractVerifier.COMPILE_MAP` 中定义。

### 9.4 假删除（软删除）

配置 Access 表中对应表的 `deletedKey`、`deletedValue`、`notDeletedValue`：

```json
{
  "Moment": {
    "deletedKey": "isDeleted",
    "deletedValue": 1,
    "notDeletedValue": 0
  }
}
```

此时 DELETE 请求会自动转为 UPDATE SET isDeleted=1，GET 查询自动追加 `isDeleted = 0` 条件。

### 9.5 安全最佳实践

1. **禁用 @raw**：非必要不启用 `@raw`，启用后严格控制白名单
2. **脚本沙箱**：开启 IF/Function 脚本必须配置 ClassFilter
3. **角色最小权限**：Access 表中按需分配，不使用 ADMIN 作为默认角色
4. **参数校验**：所有写操作在 Request 表中配置 MUST/VERIFY/TYPE
5. **PreparedStatement**：保持预编译开启（默认），防 SQL 注入
6. **频率限制**：Access 表中配置 `max:1`/`time:60` 等频率参数
7. **DEBUG模式**：生产环境关闭 `Log.DEBUG`，防止泄露表结构

---

## 十、多数据库支持

### 10.1 支持的数据库 (30+)

| 类别 | 数据库 |
|------|--------|
| 关系型 | MySQL, PostgreSQL, SQL Server, Oracle, DB2, MariaDB, TiDB, CockroachDB, SQLite, DuckDB, Dameng(达梦), KingBase(人大金仓), OpenGauss |
| 分析型 | ClickHouse, Hive, Presto, Trino, Doris, StarRocks, Snowflake, Databricks, Databend |
| NoSQL | Elasticsearch, Manticore, MongoDB, Cassandra, Redis, Kafka, MQ |
| 时序 | InfluxDB, TDengine, TimescaleDB, QuestDB, IoTDB |
| 图/向量 | SurrealDB, Milvus |

### 10.2 指定数据库

在请求中通过 `@database` 指定：
```json
{
  "@database": "CLICKHOUSE",
  "Log": {
    "@sample": "0.1",
    "event": "login"
  }
}
```

或在全局配置中设置默认数据库。

### 10.3 数据库方言差异处理

| 功能 | MySQL | PostgreSQL | Oracle | ClickHouse |
|------|-------|------------|--------|------------|
| 引号 | `` ` `` | `"` | `""` | `` ` `` |
| REGEXP | `REGEXP BINARY` | `~`/`~*` | `regexp_like()` | `match()` |
| 字符串长度 | `length()` | `length()` | `length()` | `length()` |
| JSON长度 | `json_length()` | `json_array_length()` | - | `length()` |
| 分页 | `LIMIT offset,count` | `LIMIT count OFFSET offset` | `OFFSET offset ROWS FETCH NEXT count ROWS` | `LIMIT offset,count` |
| 自增ID | AUTO_INCREMENT | SERIAL | SEQUENCE | - |

---

## 附录：常用查询示例

### A.1 分页查询
```json
{
  "Moment": {
    "@column": "id,content,date",
    "@order": "date-,id+",
    "@count": 20,
    "@page": 1,
    "userId": 1
  }
}
```

### A.2 分组聚合
```json
{
  "Moment[]": {
    "@column": "sex;count(id):total,avg(age):avgAge",
    "@group": "sex",
    "@having": "count(id)>10"
  }
}
```

### A.3 复杂条件
```json
{
  "User": {
    "age{}": ">=18;<=65",
    "name$": "a",
    "status{}": [0,1,2],
    "registerDate%": "2024-01-01,2024-12-31",
    "@combine": "age{} & name$ & (status{} | registerDate%)"
  }
}
```

### A.4 关联查询
```json
{
  "Moment": {
    "id": 1,
    "User@": {
      "id@": "/Moment/userId"
    },
    "Comment[]@": {
      "momentId@": "/Moment/id"
    }
  }
}
```

### A.5 批量插入
```json
{
  "Moment[]": {
    "Moment": {
      "content": "test",
      "userId": 1
    }
  }
}
```

### A.6 条件更新
```json
{
  "Moment": {
    "id": 1,
    "content": "updated content",
    "@combine": "id"
  }
}
```
