---
lesson: 16
title: SQLBuilder 与编程式 SQL 构建
duration: 35 分钟
objectives:
  - 理解 SQLBuilder 的设计和使用场景
  - 掌握编程式构建 SQL AST 的方法
  - 学会用 Builder 构建 SELECT/INSERT/UPDATE
prerequisites: 第 10-12 课
---

# 第 16 课：SQLBuilder 与编程式 SQL 构建

## 两种创建 AST 的方式

```java
// 方式 1: 从 SQL 字符串解析（解析方式）
SQLStatement stmt = SQLUtils.parseSingleStatement("SELECT id, name FROM users", DbType.mysql);

// 方式 2: 编程构建（构建方式）
SQLSelectStatement stmt = new SQLSelectStatement();
SQLSelectQueryBlock query = new SQLSelectQueryBlock();
query.addSelectItem(new SQLIdentifierExpr("id"));
// ...
```

Druid 的 `builder/` 包为方式 2 提供了更友好的 API。

## SQLBuilder 体系

**文件位置**: `druid/core/src/main/java/com/alibaba/druid/sql/builder/`

```
SQLBuilder.java         → Builder 标记接口（空接口）
SQLSelectBuilder.java   → SELECT 构建器
SQLUpdateBuilder.java   → UPDATE 构建器
SQLDeleteBuilder.java   → DELETE 构建器
SQLFunctionBuilder.java → 函数构建器
```

## SQLSelectBuilder 使用

```java
import com.alibaba.druid.sql.builder.impl.SQLSelectBuilderImpl;

SQLSelectBuilder builder = new SQLSelectBuilderImpl(DbType.mysql);

// 链式调用构建 SELECT 语句
String sql = builder
    .select("id", "name", "age")
    .from("users")
    .whereAnd("age > 18")           // ⚠️ where() 会覆盖已有条件
    .whereAnd("status = 'active'")  // 应使用 whereAnd() 追加条件
    .orderBy("id DESC")
    .limit(10)
    .toString();

System.out.println(sql);
// 输出:
// SELECT id, name, age
// FROM users
// WHERE age > 18
//   AND status = 'active'
// ORDER BY id DESC
// LIMIT 10
```

### SQLSelectBuilder 接口

```java
public interface SQLSelectBuilder {
    SQLSelectBuilder select(String... columns);
    SQLSelectBuilder from(String table);
    SQLSelectBuilder where(String conditions);
    SQLSelectBuilder whereAnd(String conditions);
    SQLSelectBuilder whereOr(String conditions);
    SQLSelectBuilder groupBy(String expr);
    SQLSelectBuilder having(String expr);
    SQLSelectBuilder orderBy(String expr);
    SQLSelectBuilder limit(int rowCount);
    SQLSelectBuilder limit(int rowCount, int offset);

    String toString();
    SQLSelectStatement getSQLSelectStatement();
}
```

## SQLUpdateBuilder 使用

```java
SQLUpdateBuilder builder = new SQLUpdateBuilderImpl(DbType.mysql);

String sql = builder
    .from("users")
    .set("name = 'Bob'")
    .set("age = 25")
    .where("id = 1")
    .toString();

System.out.println(sql);
// 输出:
// UPDATE users
// SET name = 'Bob', age = 25
// WHERE id = 1
```

## SQLDeleteBuilder 使用

```java
SQLDeleteBuilder builder = new SQLDeleteBuilderImpl(DbType.mysql);

String sql = builder
    .from("users")
    .where("status = 'inactive'")
    .where("created_at < '2020-01-01'")
    .toString();

System.out.println(sql);
// 输出:
// DELETE FROM users
// WHERE status = 'inactive'
//   AND created_at < '2020-01-01'
```

## 底层构建方式（不使用 Builder）

Builder 只是语法糖，底层还是直接操作 AST 节点：

```java
// 手动构建 SELECT 语句
SQLSelectStatement stmt = new SQLSelectStatement();
SQLSelect select = new SQLSelect();
SQLSelectQueryBlock query = new SQLSelectQueryBlock();

// SELECT id, name
query.addSelectItem(new SQLSelectItem(new SQLIdentifierExpr("id")));
query.addSelectItem(new SQLSelectItem(new SQLIdentifierExpr("name")));

// FROM users
query.setFrom(new SQLExprTableSource(new SQLIdentifierExpr("users")));

// WHERE age > 18
SQLBinaryOpExpr where = new SQLBinaryOpExpr(
    new SQLIdentifierExpr("age"),
    SQLBinaryOperator.GreaterThan,
    new SQLIntegerExpr(18)
);
query.setWhere(where);

// 组装
select.setQuery(query);
stmt.setSelect(select);

// 输出
String sql = SQLUtils.toSQLString(stmt, DbType.mysql);
```

## 动态构建的场景

Builder 在需要根据**运行时条件**动态拼接 SQL 时特别有用：

```java
public String buildDynamicQuery(UserQueryParam param) {
    SQLSelectBuilder builder = new SQLSelectBuilderImpl(DbType.mysql);
    builder.select("*").from("users");

    // 根据参数动态添加条件
    if (param.getName() != null) {
        builder.whereAnd("name LIKE '%" + param.getName() + "%'");
        // ⚠️ 实际使用请用参数化查询
    }
    if (param.getMinAge() != null) {
        builder.whereAnd("age >= " + param.getMinAge());
    }
    if (param.getCity() != null) {
        builder.whereAnd("city = '" + param.getCity() + "'");
    }
    if (param.getSortBy() != null) {
        builder.orderBy(param.getSortBy() + " " + param.getSortOrder());
    }
    if (param.getLimit() != null) {
        builder.limit(param.getLimit());
    }

    return builder.toString();
}
```

## 💡 对你的启发

Builder 模式在你实现 SQL-to-ES DSL 时有两种用途：

1. **构建测试 SQL**：用 Builder 快速生成各种 SQL 来测试你的转换器
2. **构建 DSL 结果**：可以借鉴 Builder 的链式设计，让你的 ES DSL 生成器也支持链式调用

```java
// 你可以提供一个类似的 ES DSL Builder
EsQueryBuilder builder = new EsQueryBuilder("users");
String esDsl = builder
    .term("status", "active")
    .range("age", 18, 60)
    .sort("id", "desc")
    .size(20)
    .build();
```

## 🔍 动手探索

1. 查看 `builder/SQLSelectBuilder.java` 的完整接口定义
2. 阅读 `builder/impl/SQLSelectBuilderImpl.java` 的实现，看它如何将 Builder 调用转化为 AST 节点操作
3. 查看 `SQLBuilderFactory` 中如何创建不同类型的 Builder

## 思考题

1. Builder 模式在构建复杂 SQL 时有什么优势？在什么场景下你会选择 Builder 而非直接解析 SQL？
2. `where()` 方法如何将字符串 `"age > 18"` 转换为 `SQLBinaryOpExpr` 节点？它内部使用了什么机制？
3. 如果让你设计一个 ES DSL Builder，你的 API 应该是什么样的？

## 关键源码路径

| 文件 | 说明 |
|------|------|
| builder/SQLSelectBuilder.java | SELECT Builder 接口 |
| builder/SQLUpdateBuilder.java | UPDATE Builder 接口 |
| builder/SQLDeleteBuilder.java | DELETE Builder 接口 |
| builder/impl/SQLSelectBuilderImpl.java | SELECT Builder 实现 |
| builder/SQLBuilderFactory.java | Builder 工厂 |

## 思考题答案

<details>
<summary>点击展开</summary>

1. **Builder 模式和直接解析 SQL 各有什么适用场景？**
   - **Builder 模式**：适用于**动态构建**场景，如根据用户传入的参数组合不同的 WHERE 条件、SELECT 列。你可以在运行时决定加不加某个条件。
   - **直接解析**：适用于**静态分析和转换**场景，如 SQL 审核、SQL 格式化、方言转换——你有一串现成的 SQL，需要读懂它。
   - 两者不互斥：你可以先用 Builder 构建 SQL，再用 Druid 解析它做进一步分析。

2. **`where()` 方法如何将 `"age > 18"` 转换为 `SQLBinaryOpExpr`？**
   - `SQLSelectBuilderImpl.where(String conditions)` 内部调用了 `SQLUtils.toSQLExpr(conditions, dbType)`，这个方法和 Parser 中的 `exprParser.expr()` 是同一套逻辑——把字符串解析成表达式 AST。所以 `"age > 18"` 字符串被解析成了 `SQLBinaryOpExpr(> , SQLIdentifierExpr(age), SQLIntegerExpr(18))`。

3. **设计 ES DSL Builder，API 应该是什么样的？**
   ```java
   // 可以参考 Druid 的 SQLBuilder 设计
   EsDslBuilder builder = new EsDslBuilder("users");
   String dsl = builder
       .term("status", "active")         // term 精确匹配
       .range("age", 18, null)            // range gt
       .range("age", null, 60)            // range lt
       .match("name", "张三")             // match 全文搜索
       .sort("id", "desc")                // 排序
       .size(20)                          // 分页
       .source("id", "name")              // _source 字段
       .build();                          // 输出 JSON
   ```
</details>

## 下一课预告

**第 17 课：SchemaRepository 与元数据管理** — SchemaRepository 是 Druid 的"元数据大脑"。它管理了 Schema、表定义、列信息等元数据，可以辅助 SQL 解析过程中的类型推断和语义分析。
