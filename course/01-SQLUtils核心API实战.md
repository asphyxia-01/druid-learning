---
lesson: 1
title: SQLUtils 核心 API 实战
duration: 45 分钟
objectives:
  - 掌握 SQLUtils 的主要 API 分类
  - 理解 format/parse/toSQLString 的调用链路
  - 学会通过 SQLUtils 与 AST 交互
prerequisites: 第 0 课
---

# 第 1 课：SQLUtils 核心 API 实战

## 学习目标

SQLUtils 是 Druid SQL 模块的**统一入口**。你不需要直接操作 Lexer、Parser 或 Visitor —— 一切都可以通过 SQLUtils 来完成。本节课将带你全面掌握 SQLUtils 的所有核心 API。

## SQLUtils 位于哪里

**文件位置**: `druid/core/src/main/java/com/alibaba/druid/sql/SQLUtils.java`

这是一个约 1200 行的工具类，包含了 SQL 处理的各种静态方法。

## API 分类速览

| 分类 | 代表性方法 | 用途 |
|------|-----------|------|
| **格式化** | `format()`, `formatMySql()`, `formatOracle()` | SQL 美化 |
| **解析** | `parseStatements()`, `parseSingleStatement()`, `toSQLExpr()` | SQL → AST |
| **输出** | `toSQLString()`, `toMySqlString()`, `toOracleString()` | AST → SQL |
| **条件操作** | `addCondition()`, `removeCondition()` | 动态修改 WHERE |
| **函数操作** | `acceptFunction()` | 查找/处理函数调用 |
| **工具方法** | `normalize()`, `toLowerCase()`, `toUpperCase()` | 字符串处理 |

## 1. SQL 格式化 (format)

`format()` 是最常用的方法之一。它内部执行了 **parse → visitor output** 的完整流程。

```java
// 源码位置: SQLUtils.java:385
public static String format(String sql, DbType dbType, List<Object> parameters,
                            FormatOption option, SQLParserFeature[] features) {
    SQLStatementParser parser = SQLParserUtils.createSQLStatementParser(sql, dbType, features);
    List<SQLStatement> statementList = parser.parseStatementList();
    return toSQLString(statementList, dbType, parameters, option);
}
```

### 使用示例

```java
// 基础格式化
String sql = "select * from t where id = ?";
String result = SQLUtils.format(sql, DbType.mysql);
// 输出:
// SELECT *
// FROM t
// WHERE id = ?

// 带参数填充
String result2 = SQLUtils.format(sql, DbType.mysql, Arrays.asList("abc"));
// 输出:
// SELECT *
// FROM t
// WHERE id = 'abc'

// 快捷方法
SQLUtils.formatMySql(sql);
SQLUtils.formatOracle(sql);
SQLUtils.formatOdps(sql);
SQLUtils.formatPresto(sql);
```

### FormatOption 详解

```java
public class FormatOption {
    boolean uppCase = true;        // 是否大写关键字
    boolean prettyFormat = true;   // 是否美化(换行缩进)
    boolean parameterized = false; // 是否参数化(? 替换字面量)
    int features;                  // VisitorFeature 位标记
}
```

### 💡 设计思想

`format()` 的流程是 Druid SQL 处理的**标准模板**：
1. `SQLParserUtils.createSQLStatementParser()` — 创建对应方言的解析器
2. `parser.parseStatementList()` — 解析为 List<SQLStatement> (AST)
3. `SQLUtils.toSQLString()` — 通过 Visitor 输出 SQL

这其实是"解析 → AST → 访问者输出"的经典编译流程。

## 2. SQL 解析 (parseStatements)

将 SQL 字符串解析为 AST 节点树。

```java
// 源码位置: SQLUtils.java:680
public static List<SQLStatement> parseStatements(String sql, DbType dbType,
                                                  SQLParserFeature... features) {
    SQLStatementParser parser = SQLParserUtils.createSQLStatementParser(sql, dbType, features);
    List<SQLStatement> stmtList = new ArrayList<SQLStatement>();
    parser.parseStatementList(stmtList, -1, null);
    if (parser.getLexer().token() != Token.EOF) {
        throw new ParserException("syntax error : " + parser.getLexer().info());
    }
    return stmtList;
}
```

### 使用示例

```java
String sql = "SELECT id, name FROM users WHERE age > ?";

// 解析为 AST
List<SQLStatement> stmts = SQLUtils.parseStatements(sql, DbType.mysql);

// 遍历 AST
for (SQLStatement stmt : stmts) {
    System.out.println("语句类型: " + stmt.getClass().getSimpleName());

    // 输出 AST 结构
    System.out.println("AST 输出:\n" + SQLUtils.toSQLString(stmt, DbType.mysql));
}

// 解析单条语句
SQLStatement single = SQLUtils.parseSingleStatement(sql, DbType.mysql, false);

// 解析表达式
SQLExpr expr = SQLUtils.toSQLExpr("age > 18", DbType.mysql);
System.out.println(expr.getClass().getSimpleName()); // SQLBinaryOpExpr
```

### ⚠️ 注意

- `parseStatements()` 支持多语句（以分号分隔），返回 List
- `parseSingleStatement()` 只接受单条语句，多条会抛异常
- 解析完成后会检查是否到达 EOF，否则抛 `ParserException`

## 3. AST 输出 (toSQLString)

将 AST 节点重新转换为 SQL 字符串。

```java
// 源码位置: SQLUtils.java:119
public static String toSQLString(SQLObject sqlObject, DbType dbType,
                                  FormatOption option, VisitorFeature... features) {
    StringBuilder out = new StringBuilder();
    SQLASTOutputVisitor visitor = createOutputVisitor(out, dbType);
    // ... 配置 option ...
    sqlObject.accept(visitor);  // ★ 核心: 调用 AST 节点的 accept 方法
    return out.toString();
}
```

### 使用示例

```java
List<SQLStatement> stmts = SQLUtils.parseStatements("SELECT * FROM t", DbType.mysql);

// 默认格式(大写、美化)
SQLUtils.toSQLString(stmts, DbType.mysql);

// 小写格式
FormatOption lowerOption = new FormatOption(false, true);
SQLUtils.toSQLString(stmts, DbType.mysql, lowerOption);

// 参数化格式(字面量变?)
FormatOption paramOption = FormatOption.of(VisitorFeature.OutputParameterized);
SQLUtils.toSQLString(stmts, DbType.mysql, paramOption);

// 方言快捷方法
SQLUtils.toMySqlString(stmt);
SQLUtils.toOracleString(stmt);
SQLUtils.toPGString(stmt);
```

## 4. 条件操作 (addCondition / removeCondition)

动态修改 SQL 的 WHERE 条件。

```java
String sql = "SELECT * FROM t WHERE id = 1";

// 添加条件 (AND)
String result = SQLUtils.addCondition(sql, "name = 'abc'", DbType.mysql);
// SELECT * FROM t WHERE id = 1 AND name = 'abc'

// 移除条件
String result2 = SQLUtils.removeCondition(sql, "id", DbType.mysql);
// SELECT * FROM t
```

⚠️ 这些方法**内部也是解析→修改AST→输出的流程**。

## 5. 函数查找 (acceptFunction)

遍历 AST 找到所有的函数调用。

```java
// 源码位置: Druid 的非静态方法（示例见测试）
List<SQLMethodInvokeExpr> functions = new ArrayList<>();
SQLUtils.acceptFunction(
    "select count(*) from t",
    DbType.odps,
    e -> functions.add(e),   // 找到函数时的回调
    e -> true                 // 过滤条件
);
// functions 中包含 count(*) 对应的 SQLMethodInvokeExpr
```

## 完整示例: 模拟 SQL-to-ES-DSL

将这个最简单的例子作为你去实现 SQL-to-ES DSL 的起点：

```java
public class SqlToDslDemo {
    public static void main(String[] args) {
        String sql = "SELECT id, name FROM users WHERE age > 18 AND status = 'active' ORDER BY id DESC LIMIT 10";

        // 1. 解析 SQL
        List<SQLStatement> stmts = SQLUtils.parseStatements(sql, DbType.mysql);
        SQLStatement stmt = stmts.get(0);

        // 2. 观察 AST 结构
        System.out.println("=== AST ===");
        System.out.println(SQLUtils.toSQLString(stmt, DbType.mysql));

        // 3. 用 SchemaStatVisitor 收集表和字段信息
        SchemaStatVisitor visitor = SQLUtils.createSchemaStatVisitor(DbType.mysql);
        stmt.accept(visitor);

        System.out.println("\n=== 涉及的表 ===");
        visitor.getTables().keySet().forEach(t -> System.out.println("  " + t.getName()));

        System.out.println("=== 涉及的列 ===");
        visitor.getColumns().forEach(c -> System.out.println("  " + c));

        System.out.println("=== 条件 ===");
        visitor.getConditions().forEach(c -> System.out.println("  " + c));
    }
}
```

## 思考题

1. `SQLUtils.format()` 和 `SQLUtils.toSQLString()` 有什么区别？它们各自在什么场景下使用？
2. 如果我想实现 SQL to ES DSL，应该从 `SQLUtils` 的哪个 API 入手？为什么？
3. 阅读 `SQLUtils.java:119` 的 `toSQLString` 方法，理解它是如何创建 Visitor 并驱动 AST 遍历的。

## 关键源码路径

| 文件 | 关键方法 | 行号 |
|------|---------|------|
| SQLUtils.java | `format()` | ~385 |
| SQLUtils.java | `parseStatements()` | ~680 |
| SQLUtils.java | `toSQLString()` | ~119 |
| SQLUtils.java | `addCondition()` | ~520 |
| SQLUtils.java | `createOutputVisitor()` | ~1156 |

## 下一课预告

**第 2 课：Token 体系详解** — 我们将深入 Lexer 的世界，看看 SQL 字符串是如何被拆解成一个个 Token 的。你会学到 Token 枚举的定义、Token 的类型体系，以及 Druid 是如何高效管理这些 Token 的。
