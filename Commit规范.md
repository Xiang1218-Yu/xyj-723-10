# Commit 规范（Commit Convention）

> 适用于本仓库（APIJSONORM）。遵循 [Conventional Commits](https://www.conventionalcommits.org/) 风格并结合 APIJSON 模块划分。

---

## 一、提交信息格式

```
<type>(<scope>): <subject>

<body>

<footer>
```

- **必填**：`type`、`subject`。
- **可选**：`scope`、`body`、`footer`。
- **首行（header）**：不超过 72 字符，使用祈使句、句尾不加句号。
- 中英文均可，同一仓库内保持一致；正文与首行之间空一行。

---

## 二、type 类型

| type | 说明 | 示例 |
| --- | --- | --- |
| `feat` | 新功能 | `feat(sqlconfig): 支持 @combine 括号分组表达式` |
| `fix` | 缺陷修复 | `fix(parser): 修复数组逐项复用时关键字未还原` |
| `perf` | 性能优化 | `perf(executor): 复用 preparedValueList 降低解析开销` |
| `refactor` | 重构（不改行为） | `refactor(objectparser): 抽取 onParse 分派逻辑` |
| `docs` | 文档 | `docs: 补充 @combine 语法对照表` |
| `test` | 测试 | `test(sqlconfig): 覆盖 key{} 范围表达式` |
| `style` | 格式（不影响逻辑） | `style: 统一缩进与导入顺序` |
| `build` | 构建/依赖 | `build: 升级 maven-compiler-plugin 到 3.12.1` |
| `ci` | CI 配置 | `ci: 增加 JitPack 发布流程` |
| `chore` | 杂项 | `chore: 更新 .gitignore` |
| `revert` | 回滚 | `revert: 回滚 "feat(sqlconfig): ..."` |

---

## 三、scope 范围（结合本项目模块）

按实际改动的核心类/子系统选择：

| scope | 对应代码 |
| --- | --- |
| `parser` | `AbstractParser` / `Parser`（请求编排、事务、校验） |
| `objectparser` | `AbstractObjectParser` / `ObjectParser`（单对象拆解） |
| `sqlconfig` | `AbstractSQLConfig` / `SQLConfig`（SQL 生成、`newSQLConfig`、`@combine`） |
| `executor` | `AbstractSQLExecutor` / `SQLExecutor`（JDBC 执行、缓存） |
| `verifier` | `AbstractVerifier` / `Verifier`（权限、结构校验） |
| `function` | `AbstractFunctionParser` / `FunctionParser`（远程函数） |
| `script` | `script/*`（JSR223、JavaScript 脚本） |
| `model` | `orm/model/*`（Table、Column、Request 等元数据） |
| `json` | `JSON` / `JSONMap` / `JSONList` / `JSONResponse` |
| `util` | `StringUtil` / `Log` / `SQL` 等工具 |

> 跨多个模块时可省略 scope，或用最能代表主改动的 scope。

---

## 四、subject 与 body 要求

- **subject**：一句话说明「做了什么」，动词开头（新增/修复/优化/重构…）。
- **body**：解释「为什么」与「怎么做」，尤其是涉及 `newSQLConfig` 关键字增删、`parseCombineExpression` 扫描逻辑、安全阈值（`MAX_COMBINE_*`、`MAX_QUERY_DEPTH`）等敏感改动时，必须说明影响范围与兼容性。
- 涉及行为变更或破坏性变更，body 中列出前后对比。

---

## 五、footer

- **关联 issue**：`Closes #123` / `Refs #456`。
- **破坏性变更**：以 `BREAKING CHANGE:` 开头，说明迁移方式。

```
feat(sqlconfig): @combine 默认连接符调整为 OR

5.0+ 中 @having 缺省逻辑连接符由 AND 调整为 OR，与文档语义对齐。

BREAKING CHANGE: 旧请求若依赖 @having 默认 AND，需显式使用 @having& 或在 @having 内声明 @combine。
Closes #789
```

---

## 六、提交前检查清单

- [ ] 单行 header ≤ 72 字符，type/scope 正确。
- [ ] 敏感逻辑（请求关键字增删、条件解析、安全阈值）已在 body 说明影响。
- [ ] 不提交密钥、凭证、构建产物（遵循 `.gitignore`）。
- [ ] 优先使用**新提交**而非 `--amend`；一次提交聚焦一件事。
- [ ] 破坏性变更已在 footer 标注 `BREAKING CHANGE`。
