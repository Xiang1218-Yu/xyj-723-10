# APIJSON 通用文档（Developer Guide）

> 基于源码 `APIJSONORM 8.2.0` 深入梳理。核心目标：以 `newSQLConfig` 为切入点看清**请求解析全貌**，并通过 `@combine` 理解**条件表达式引擎**。
> 所有代码位置均可点击跳转。

---

## 目录
1. [整体架构与请求生命周期](#1-整体架构与请求生命周期)
2. [从 newSQLConfig 切入：请求解析全貌](#2-从-newsqlconfig-切入请求解析全貌)
3. [JSON 请求分解：关键字与条件操作符](#3-json-请求分解关键字与条件操作符)
4. [@combine 条件表达式引擎](#4-combine-条件表达式引擎)
5. [SQL 生成与执行](#5-sql-生成与执行)
6. [扩展点与关键数据结构](#6-扩展点与关键数据结构)

---

## 1. 整体架构与请求生命周期

APIJSON ORM 的核心是三层递归下降：

```
AbstractParser         请求编排（全局关键字、校验、事务、递归入口）
    └─ AbstractObjectParser   单个 Table:{} 对象拆解
            └─ AbstractSQLConfig   请求 → SQLConfig → SQL 字符串
                    └─ SQLExecutor      JDBC 执行
```

所有核心类都是泛型 `<T, M extends Map<String,Object>, L extends List<Object>>`，其中 `M` 表示一个 JSON 对象，`L` 表示一个 JSON 数组。

### 请求生命周期（`parseResponse(M)`）
入口 [AbstractParser.java:505](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Bulbasaur/APIJSONORM/src/main/java/apijson/orm/AbstractParser.java#L505)：

1. **剥离全局关键字**：`@format`、`@version`/`tag`、`@database`、`@datasource`、`@namespace`、`@catalog`、`@schema`、`@explain`、`@cache`，存为 `globalXxx` 字段。
2. **权限前置校验**：`onVerifyLogin()`、`onVerifyContent()`（内部调用 `parseCorrectRequest()` 按 `Request` 表配置校验/改写请求结构）。
3. **`@role` 提取**（需要角色校验时）。
4. **初始化** `queryResultMap`（路径→结果，用于 `key@` 引用解析）。
5. **`onBegin()`** 对非查询方法开启事务。
6. **`onObjectParse(request, null, null, null, false, null)`** —— 递归下降起点，见 [AbstractParser.java:588](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Bulbasaur/APIJSONORM/src/main/java/apijson/orm/AbstractParser.java#L588)。
7. **`onCommit()` / `onRollback()`**。
8. **组装响应**：`extendSuccessResult` / `extendErrorResult`，附带 `code/msg/time` 与调试元数据（`sql`、`depth`、`time`、`trace`）。
9. **`onClose()`** 返回。

---

## 2. 从 newSQLConfig 切入：请求解析全貌

`newSQLConfig` 是**请求 → SQLConfig** 的翻译核心，分两层。

### 2.1 实例方法（ObjectParser 侧）
[AbstractObjectParser.java:934](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Bulbasaur/APIJSONORM/src/main/java/apijson/orm/AbstractObjectParser.java#L934) 的 `newSQLConfig(boolean isProcedure)`：校验 `@raw` 后，用对象自身的 `method, table, alias, sqlRequest, joinList` 委托给 6 参数形式（声明在 [ObjectParser.java:135](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Bulbasaur/APIJSONORM/src/main/java/apijson/orm/ObjectParser.java#L135)，由应用侧实现），后者最终转调静态核心构建器并传入 `Callback`。

### 2.2 静态核心构建器
[AbstractSQLConfig.java:5477](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Bulbasaur/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L5477)：

```java
public static <T, M extends Map<String,Object>, L extends List<Object>> SQLConfig<T,M,L> newSQLConfig(
    RequestMethod method, String table, String alias,
    M request, List<Join<T,M,L>> joinList, boolean isProcedure,
    Callback<T,M,L> callback) throws Exception
```

执行顺序：

1. **校验**：request 非空；`@explain` 仅 DEBUG 可用。
2. **提取库坐标**：`@database/@datasource/@namespace/@catalog/@schema`。
3. **`callback.getSQLConfig(...)`** 获取具体 `SQLConfig` 实例（依赖注入点）。
4. `isProcedure` 为真则提前返回（存储过程）。
5. **`parseJoin(...)`** 挂接 JOIN 子配置（见 `parseJoin`）。
6. **主键解析**：`idKey`、`id{}`→`idInKey`、`userIdKey`、`userId{}`→`userIdInKey`，校验值类型；POST 无 id 时 `callback.newId(...)`。
7. **提取并移除全部表级关键字**（见 [AbstractSQLConfig.java:5639](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Bulbasaur/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L5639) 起）：`@role/@cache/@from/@column/@null/@cast/@combine/@group/@having/@having&/@sample/@latest/@partition/@fill/@order/@key/@raw/@json/@method`。
8. 处理 `@null`、`@cast`、`@raw`。
9. **POST 分支**：校验剩余键均为合法列名，构建 `column` 与 `VALUES(...)`。
10. **非 POST 分支**：构建 `tableWhere`（有序）与 `combineMap`（`&`/`|`/`!` → `andList/orList/notList`）；id/userId 强制进 `andList`；注入假删除条件；**解析 `@combine` 表达式**；遍历剩余键落入 `tableWhere`（条件）或 `tableContent`（PUT 的 SET 值）。
11. 假删除 → DELETE 改写为 PUT。
12. **`@column` 分解**：按 `;` 切成函数表达式 `fun(a,b)` 与普通列，处理 `DISTINCT`。
13. **`@having`/`@having&` 分解** → `havingMap` + `havingCombine`。
14. **`@key`（keyMap）** 列别名/表达式解析。
15. **setter 回填**：`setColumn/setFrom/setWhere/setGroup/setHaving/setHavingCombine/setOrder/...`。
16. **`finally`** 把移除的关键字**还原回 request**（因为同一 request map 会被数组逐项复用）。
17. 返回填充完毕的 `config`。

> ⚠️ 关键陷阱：第 16 步的还原逻辑是数组多项查询能正确复用请求的前提，修改本方法时务必保持。

---

## 3. JSON 请求分解：关键字与条件操作符

### 3.1 全局关键字（顶层，`parseResponse` 剥离）
`@format`、`@version`、`tag`、`@database`、`@datasource`、`@namespace`、`@catalog`、`@schema`、`@explain`、`@cache`、`@role`。

### 3.2 表级关键字（`newSQLConfig` 剥离，常量见 [JSONMap.java](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Bulbasaur/APIJSONORM/src/main/java/apijson/JSONMap.java)）
`@column`(SELECT列/函数)、`@combine`(条件组合)、`@group`(GROUP BY)、`@having`/`@having&`(HAVING)、`@order`(ORDER BY)、`@raw`(原始SQL)、`@role`、`@null`、`@cast`、`@from`、`@sample`、`@latest`、`@partition`、`@fill`、`@key`、`@json`、`@method`。

### 3.3 条件操作符后缀（`gainWhereItem` 分派）
由 [AbstractSQLConfig.java:3897](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Bulbasaur/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L3897) 起根据键后缀决定 `keyType` 并分派：

| 后缀 | keyType | 处理方法 | 含义 | SQL |
| --- | --- | --- | --- | --- |
| `key$` | 1 | `gainSearchString` | 模糊搜索 | `key LIKE 'value'` |
| `key~` / `key*~` | 2 / -2 | `gainRegExpString` | 正则（`*`忽略大小写） | `key REGEXP ...` / `~*` / `regexp_like(...)` |
| `key%` | 3 | `gainBetweenString` | 区间 | `key BETWEEN a AND b` |
| `key{}` | 4 | `gainRangeString` | 集合/范围 | `key IN (...)` 或 `>0`、`<=1,3` 等 |
| `key}{` | 5 | `gainExistsString` | 存在子查询 | `EXISTS (subquery)` |
| `key<>` | 6 | `gainContainString` | JSON 包含 | `json_contains(...)` / `@>` |
| `key>=` `key<=` `key>` `key<` | 7-10 | `gainCompareString` | 比较 | `key >= value` 等 |
| （无） | 0 | `gainEqualString` | 等值 | `key = value` / `IS [NOT] NULL` |

- `key!`（列名带 `!`）在 `gainEqualString` 中转 `!=` / `IS NOT`。
- `key[`→`length(key)`、`key{`→`json_length(key)`（在 `gainKey` 中）。
- 每个操作符方法还会用 `new Logic(column)` 解析**列名自带的 `&`/`|`/`!` 后缀**，把数组多值用 AND/OR/NOT 连接——这是与顶层 `@combine` 正交的第二层逻辑。

### 3.4 键分派（ObjectParser.onParse）
[AbstractObjectParser.java:415](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Bulbasaur/APIJSONORM/src/main/java/apijson/orm/AbstractObjectParser.java#L415) 将键分为三类：
- **`key@`**（引用/子查询）：Map → SQL 子查询（`Subquery`）；String → 依赖路径，经 `onReferenceParse` → `getValueByPath` 解析。
- **`key()`**（远程函数）：按 `-`/`+`/`0` 时机执行，非表的 `-` 函数立即执行，其余入 `functionMap`。
- **其他真实列/条件** → `sqlRequest.put(key, value)`，最终喂给 `newSQLConfig`。

---

## 4. @combine 条件表达式引擎

### 4.1 两种形态
`@combine` 常量定义于 [JSONMap.java:184](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Bulbasaur/APIJSONORM/src/main/java/apijson/JSONMap.java#L184)。在 `gainWhereString` [AbstractSQLConfig.java:3324](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Bulbasaur/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L3324) 选择路径：

```java
public String gainWhereString(boolean hasPrefix) throws Exception {
    String combineExpr = getCombine();
    if (StringUtil.isEmpty(combineExpr, false)) {
        return getWhereString(hasPrefix, getMethod(), getWhere(), getCombineMap(), getJoinList(), !isTest());
    }
    return getWhereString(hasPrefix, getMethod(), getWhere(), combineExpr, getJoinList(), !isTest());
}
```

- **新版表达式**：`"a | (b & c & !(d | !e))"`，完整布尔表达式，交给 `parseCombineExpression`。
- **旧版列表**：`"key0,&key1,|key2,!key3"`，逗号分隔、每项前缀 `&`/`|`/`!`，聚成 `combineMap`（`{"&":[...],"|":[...],"!":[...]}`）。保留是因为它能保证 JOIN 多 ON 子句的顺序且更快。

判定：在 `newSQLConfig` 中先 `String[] ws = StringUtil.split(combine)`，再取 `combineExpr = (ws == null || ws.length != 1) ? null : ws[0]`。随后（[AbstractSQLConfig.java:5883](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Bulbasaur/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L5883)）：
- **`combineExpr` 非空**（`StringUtil.isNotEmpty(combineExpr, true)`）→ 走**新版表达式**路径（校验 id/userId 等禁用键后交给 `parseCombineExpression`）。
- **`combineExpr` 为空且 `ws != null`** → 走**旧版逗号列表**路径，逐项去除 `&`/`|`/`!` 前缀聚成 `combineMap`。

未在表达式中出现的 `where` 键会被隐式 AND。

### 4.2 Logic 模型
[Logic.java](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Bulbasaur/APIJSONORM/src/main/java/apijson/orm/Logic.java)：`TYPE_OR=0` `TYPE_AND=1` `TYPE_NOT=2`，字符 `| & !`。`Logic(String key)` 检查键末字符决定类型，默认 OR。

### 4.3 核心解析器 parseCombineExpression
[AbstractSQLConfig.java:3364](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Bulbasaur/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L3364) —— **单遍手写扫描器**（非递归下降/AST），直接把括号与 `AND/OR/NOT` 写入输出串，把运算符优先级交给数据库。WHERE 与 HAVING 共用（`isHaving` 区分）。

**安全阈值**（[AbstractSQLConfig.java:63](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Bulbasaur/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L63)）：
```java
public static int MAX_COMBINE_DEPTH = 2;      // 括号最大嵌套
public static int MAX_COMBINE_COUNT = 5;      // 条件键最大数量
public static int MAX_COMBINE_KEY_COUNT = 2;  // 同一键最大重复
public static float MAX_COMBINE_RATIO = 1.0f; // 键数 / conditionMap 大小
```

**扫描状态**：`depth`（括号深度）、`lastLogic`（上一个连接符）、`isNot`（下一个 key 是否取反）、`key`（累积中的条件键）。逐字符处理：
- **空格 / `)` / 结尾**：flush 当前 `key` → 取 `conditionMap.get(column)` → `gainWhereItem`/`gainHavingItem` 生成片段 → `( gainCondition(isNot, wi) )` 追加。
- **` & `**：必须前后各一个空格，才当 AND；否则 `&` 是键的一部分（如 `tag&$`）。
- **` | `**：同理，当 OR。
- **`!`**：`!(` → 组取反（`NOT`）；紧贴键 → 词项取反（`isNot=true`）；否则是键的一部分。禁止 `!` 后跟空格/`)`/`&`/`!`。
- **`(`**：校验前有连接符，`depth++` 且 ≤ `MAX_COMBINE_DEPTH`。
- **`)`**：`depth--`，为负则报括号不匹配。

**`preparedValueList` 重置**：当 `isHaving == false` 时，方法在开始扫描前调用 `setPreparedValueList(new ArrayList<>())` 重置预编译值列表（[AbstractSQLConfig.java:3397-3399](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Bulbasaur/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L3397)），只收集本次表达式条件值——否则 JOIN ON 内部 `@combine` 拼接后占位符顺序会错乱。HAVING（`isHaving == true`）不重置，沿用已有列表以接在 WHERE 值之后。

**`key:placeholder` 引用形式**：flush 时会用 `column.indexOf(":")` 拆分 `key`（[AbstractSQLConfig.java:3442-3454](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Bulbasaur/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L3442)）。当 `column` 不在 `conditionMap` 中时（兼容 `@null`），以冒号后的 `placeholder` 作为条件片段（`wi = key.substring(keyIndex + 1)`），且强制 `isNot = false`、`size++` 计入数量；若占位为空则报错。

**预编译值顺序**：由于 `?` 占位符须按 SQL 顺序填充，方法分别收集表达式内条件值与「未在表达式中出现」的 AND 条件值，WHERE 时 `andCond AND ( result )`，HAVING 时相反，保证 `preparedValueList` 顺序正确。

### 4.4 语法（新版表达式）
```
expr        := term ( SP logicOp SP term )*
term        := "!"? conditionKey            // 词项取反：key 前紧贴 !
             | "!"? "(" expr ")"            // 分组，组前可加 !
logicOp     := "&" | "|"                     // AND | OR，两侧各恰好一个空格
conditionKey:= 带操作符后缀的 where/having 键（> < >= {} $ ~ % <> }{ ...）
             | key ":" placeholder           // 引用形式（值不在 map 中时）
```
严格词法（否则报中文错误）：无首尾/连续空格；`&`/`|` 两侧各一空格；`!` 词项取反须紧贴键；括号须配对且嵌套 ≤ 2；非首词项前必须有连接符。

**示例**：输入 `"date> | (contactIdList<> & (name*~ | tag&$))"` 生成约：
```
( date > ? ) OR ( ( json_contains(contactIdList, ?) ) AND ( ( name REGEXP ? ) OR ( tag LIKE ? ) ) )
```

### 4.5 HAVING 复用
`@having` / `@having&` 复用同一引擎，`gainHavingString` [AbstractSQLConfig.java:1655](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Bulbasaur/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L1655) 构建 `havingMap`+`havingCombine` 后调 `parseCombineExpression(isHaving=true)`，单项 SQL 由 `gainHavingItem` 生成。5.0+ 缺省连接符为 OR，`@having&` 或内部 `@combine` 可强制 AND。

---

## 5. SQL 生成与执行

1. `SQLConfig.gainSQL(prepared)` → 静态 `gainSQL` [AbstractSQLConfig.java:4947](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Bulbasaur/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L4947)，按方法生成：
   - POST → `INSERT INTO ... VALUES`
   - PUT → `UPDATE ... SET ... WHERE`
   - DELETE → `DELETE FROM ... WHERE`
   - GET → `SELECT column FROM conditionString LIMIT`
2. `gainConditionString` 组装 JOIN + `gainWhereString`（内部跑 `parseCombineExpression`）+ GROUP/HAVING/ORDER。
3. `SQLExecutor.execute(config, false)`（[AbstractSQLExecutor.java:169](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Bulbasaur/APIJSONORM/src/main/java/apijson/orm/AbstractSQLExecutor.java#L169)）执行 JDBC，返回结果。
4. 结果经 `response()`（[AbstractObjectParser.java:1048](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Bulbasaur/APIJSONORM/src/main/java/apijson/orm/AbstractObjectParser.java#L1048)）逐层回填、解析函数与延迟子对象。

---

## 6. 扩展点与关键数据结构

### 扩展点（应用侧实现）
- **`Callback` / `SimpleCallback`**（[AbstractSQLConfig.java:6575](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Bulbasaur/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java#L6575)）：`getSQLConfig`、`onMissingKey4Combine`、`newId`、`getIdKey`、`getUserIdKey`。
- 具体 `SQLConfig` / `ObjectParser` / `Parser` 子类，以及 `createObjectParser` / `createSQLConfig` / 6 参数 `newSQLConfig` 由使用方实现——ORM 只定义抽象/接口。
- `AbstractFunctionParser`（远程函数）、`ScriptExecutor`（脚本）。

### 关键数据结构
| 结构 | 说明 |
| --- | --- |
| `requestObject (M)` | 可变请求；关键字被移除后又在 `finally` 还原，支持数组逐项复用 |
| `queryResultMap` | 路径→结果，支撑 `key@` 引用 |
| `sqlRequest` | 列+条件，喂给 `newSQLConfig` |
| `customMap` | `@` 关键字回显 |
| `functionMap` | `Map<"-"/"0"/"+", Map<funcKey,funcExpr>>` 定时远程函数 |
| `childMap` | 延迟解析的嵌套表 |
| `where` | 有序条件（LinkedHashMap） |
| `combineMap` | `{"&"/"|"/"!": List<key>}` 旧版组合 |
| `combine` | 新版组合表达式串 |
| `content` / `values` | PUT 的 SET 值 / POST 的行数据 |
| `preparedValueList` | 预编译占位符值，须与 SQL 顺序一致 |
| `Subquery` / `Join` | 子查询 / 关联 |

---

## 附：核心文件速查
- [AbstractParser.java](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Bulbasaur/APIJSONORM/src/main/java/apijson/orm/AbstractParser.java) — 请求编排
- [AbstractObjectParser.java](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Bulbasaur/APIJSONORM/src/main/java/apijson/orm/AbstractObjectParser.java) — 单对象拆解
- [AbstractSQLConfig.java](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Bulbasaur/APIJSONORM/src/main/java/apijson/orm/AbstractSQLConfig.java) — `newSQLConfig` / `parseCombineExpression` / `gainSQL`
- [Logic.java](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Bulbasaur/APIJSONORM/src/main/java/apijson/orm/Logic.java) — 逻辑运算模型
- [JSONMap.java](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-10/xyj-723-10_Bulbasaur/APIJSONORM/src/main/java/apijson/JSONMap.java) — 关键字常量
