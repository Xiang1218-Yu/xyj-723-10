# APIJSONORM

<p>
腾讯 <a href="https://github.com/Tencent/APIJSON">APIJSON</a> ORM 库 —— 一份 JSON 描述请求，服务端自动完成增删改查与关联查询。<br/>
Tencent APIJSON ORM library. Describe your request with one JSON, the server auto-generates and executes the SQL.
</p>

- **版本 / Version**：`8.2.0`
- **语言 / Language**：Java 1.8
- **构建 / Build**：Maven（`APIJSONORM/pom.xml`）
- **分支 / Branch**：`Bulbasaur`

---

## 这是什么 / What is it

APIJSON 让客户端用**结构化 JSON** 表达查询意图：请求什么结构，就返回什么结构，无需为每个查询单独编写后端接口。本仓库为其 **ORM 核心引擎**，负责把 JSON 请求解析成 SQL 并执行。

核心链路：

```
HTTP → AbstractParser → AbstractObjectParser → AbstractSQLConfig → SQLExecutor → JSON 响应
       (编排/校验/事务)    (单对象拆解)         (newSQLConfig 生成SQL)  (JDBC)
```

---

## 快速开始 / Getting Started

### Maven
1. 添加 JitPack 仓库：
```xml
<repositories>
    <repository>
        <id>jitpack.io</id>
        <url>https://jitpack.io</url>
    </repository>
</repositories>
```
2. 添加依赖：
```xml
<dependency>
    <groupId>com.github.Tencent</groupId>
    <artifactId>APIJSON</artifactId>
    <version>8.2.0</version>
</dependency>
```

### Gradle
```gradle
allprojects {
    repositories {
        maven { url 'https://jitpack.io' }
    }
}
dependencies {
    implementation 'com.github.Tencent:APIJSON:8.2.0'
}
```

---

## 核心概念一览

| 概念 | 说明 |
| --- | --- |
| **请求解析入口** | `AbstractParser.parseResponse(M)` 剥离全局关键字、校验、开事务，递归调用 `onObjectParse` |
| **单对象拆解** | `AbstractObjectParser.parse` 将键分派到 `sqlRequest` / `customMap` / `functionMap` / `childMap` |
| **SQL 生成核心** | `AbstractSQLConfig.newSQLConfig`（静态）把请求翻译为 `SQLConfig` |
| **条件表达式引擎** | `@combine` 支持 `&`(AND) `|`(OR) `!`(NOT) 与括号，由 `parseCombineExpression` 解析 |
| **条件操作符** | `key$`(LIKE) `key~`(REGEXP) `key%`(BETWEEN) `key{}`(IN/范围) `key<>`(JSON包含) `key>=/<=/>/<`(比较) 等 |
| **扩展点** | `Callback`/`SimpleCallback`，以及应用侧实现的 `SQLConfig`/`ObjectParser`/`Parser` 子类 |

### `@combine` 示例
请求中的组合条件：
```
"@combine": "date> | (contactIdList<> & (name*~ | tag&$))"
```
生成的 WHERE（约）：
```sql
( date > ? ) OR ( ( json_contains(contactIdList, ?) ) AND ( ( name REGEXP ? ) OR ( tag LIKE ? ) ) )
```

安全阈值（防滥用）：`MAX_COMBINE_DEPTH=2`、`MAX_COMBINE_COUNT=5`、`MAX_COMBINE_KEY_COUNT=2`。

---

## 项目结构

```
xyj-723-10_Bulbasaur/
├── APIJSONORM/                 # ORM 核心库（Maven 模块）
│   └── src/main/java/apijson/
│       ├── orm/
│       │   ├── AbstractParser.java        # 请求编排
│       │   ├── AbstractObjectParser.java  # 单对象拆解
│       │   ├── AbstractSQLConfig.java     # newSQLConfig / parseCombineExpression / gainSQL
│       │   ├── AbstractSQLExecutor.java   # JDBC 执行
│       │   ├── Logic.java                 # 逻辑运算模型（& | !）
│       │   ├── AbstractVerifier.java      # 权限/结构校验
│       │   ├── AbstractFunctionParser.java# 远程函数
│       │   ├── script/                    # JSR223 / JavaScript 脚本执行
│       │   └── model/                     # Table/Column/Request 等元数据
│       ├── JSON.java / JSONMap.java / JSONList.java / JSONResponse.java
│       ├── SQL.java / StringUtil.java / Log.java
├── assets/                     # 文档配图
├── APIJSON规划及路线图.md
├── Commit规范.md
└── APIJSON通用文档.md          # 开发者深入文档（推荐从这里入手）
```

---

## 文档 / Docs

| 文档 | 用途 |
| --- | --- |
| [APIJSON通用文档.md](./APIJSON通用文档.md) | 从 `newSQLConfig` 切入的请求解析全貌 + `@combine` 表达式引擎详解 |
| [APIJSON规划及路线图.md](./APIJSON规划及路线图.md) | 产品定位、架构现状与演进路线 |
| [Commit规范.md](./Commit规范.md) | 提交信息格式与模块 scope 约定 |
| 单元测试 / Unit Test | http://apijson.cn/unit |
| FASTJSON2 分支 | https://github.com/Tencent/APIJSON/tree/fastjson2 |

---

## 参与贡献

- 提交前请阅读 [Commit规范.md](./Commit规范.md)。
- 修改 `newSQLConfig` / `parseCombineExpression` 等核心逻辑时，务必在提交说明中标注影响范围与兼容性，并补充测试。

## License

见仓库根目录 [LICENSE](./LICENSE)（Apache License 2.0）。
