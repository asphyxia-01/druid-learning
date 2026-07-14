---
lesson: 18
title: PagerUtils 与 SQL 分页
duration: 30 分钟
objectives:
  - 理解 PagerUtils 的分页原理
  - 掌握分页 SQL 的生成方法
  - 了解不同数据库的分页差异
  - 理解 Druid 在 SQL 改写场景的应用
prerequisites: 第 7-8 课
---

# 第 18 课：PagerUtils 与 SQL 分页

## PagerUtils 概述

**文件位置**: `druid/core/src/main/java/com/alibaba/druid/sql/PagerUtils.java`

PagerUtils 是 Druid 提供的一个实用工具，它可以**自动为 SQL 添加分页逻辑**。这在需要支持多种数据库分页的框架中非常有用（如 MyBatis 通用分页、Spring Data 等）。

```java
// MySQL: SELECT * FROM users LIMIT 10 OFFSET 20
// Oracle: SELECT * FROM (SELECT ..., ROWNUM AS rn FROM ...) WHERE rn > 20 AND rn <= 30
// SQL Server: SELECT TOP 10 * FROM ... (分页更复杂)
```

## 核心 API

```java
// 文件位置: PagerUtils.java
public class PagerUtils {

    // 统计 SQL（生成 COUNT 查询）
    public static String count(String sql, DbType dbType);

    // 分页 SQL（生成分页查询）
    public static String limit(String sql, DbType dbType, int offset, int count);

    // 还可以传入参数列表
    public static String limit(String sql, DbType dbType, int offset, int count, List<Object> parameters);
}
```

## 使用示例

### 1. 统计 SQL

```java
String sql = "SELECT id, name FROM users WHERE age > 18";

String countSql = PagerUtils.count(sql, DbType.mysql);
System.out.println(countSql);
// 输出: SELECT COUNT(*) FROM users WHERE age > 18
```

### 2. 分页 SQL

```java
String sql = "SELECT id, name FROM users WHERE age > 18 ORDER BY id DESC";

// MySQL 分页
String mysqlPage = PagerUtils.limit(sql, DbType.mysql, 0, 10);
System.out.println(mysqlPage);
// SELECT id, name FROM users WHERE age > 18 ORDER BY id DESC LIMIT 10

// Oracle 分页
String oraclePage = PagerUtils.limit(sql, DbType.oracle, 0, 10);
System.out.println(oraclePage);
// SELECT * FROM (
//   SELECT t.*, ROWNUM AS rn
//   FROM (SELECT id, name FROM users WHERE age > 18 ORDER BY id DESC) t
//   WHERE ROWNUM <= 10
// ) WHERE rn > 0

// PostgreSQL 分页
String pgPage = PagerUtils.limit(sql, DbType.postgresql, 0, 10);
System.out.println(pgPage);
// SELECT id, name FROM users WHERE age > 18 ORDER BY id DESC LIMIT 10
```

## 实现原理

PagerUtils 的实现也是在 AST 层面操作 SQL：

### count() 的实现

```java
// PagerUtils.java (简化)
public static String count(String sql, DbType dbType) {
    // 1. 解析 SQL
    List<SQLStatement> stmts = SQLUtils.parseStatements(sql, dbType);
    SQLSelectStatement stmt = (SQLSelectStatement) stmts.get(0);
    SQLSelectQueryBlock query = stmt.getSelect().getQueryBlock();

    // 2. 用 COUNT(*) 替换 SELECT 列表
    List<SQLSelectItem> selectList = new ArrayList<>();
    SQLAggregateExpr countExpr = new SQLAggregateExpr("COUNT");
    countExpr.addArgument(new SQLAllColumnExpr());
    selectList.add(new SQLSelectItem(countExpr));
    query.setSelectList(selectList);

    // 3. 移除 ORDER BY 和 LIMIT（对统计无意义）
    query.setOrderBy(null);
    query.setLimit(null);

    // 4. 输出为 SQL
    return SQLUtils.toSQLString(stmt, dbType);
}
```

### limit() 的方言差异

对于 MySQL，只需要简单添加 `LIMIT` 子句：

```java
// MySQL 分页
query.setLimit(new SQLLimit(count, offset));
```

对于 Oracle，需要包裹两层子查询：

```java
// Oracle 分页
// 原始: SELECT ... FROM ... WHERE ...
// 分页: SELECT * FROM (
//         SELECT t.*, ROWNUM AS rn FROM (原始) t WHERE ROWNUM <= offset + count
//       ) WHERE rn > offset
```

这展示了 Druid 在 AST 层面进行 **SQL 改写** 的能力——直接在 AST 树上插入/替换子句。

## 分页对不同方言的处理

| 数据库 | 分页方式 |
|--------|---------|
| MySQL | `LIMIT count OFFSET offset` |
| Oracle | 两层子查询 + ROWNUM |
| PostgreSQL | `LIMIT count OFFSET offset` |
| SQL Server | `TOP` + 子查询 |
| DB2 | `FETCH FIRST count ROWS ONLY` |
| H2 | 类似 MySQL |
| SQLite | 类似 MySQL |

## 自定义分页

PagerUtils 的实现思路可以借鉴来实现其他 SQL 改写场景：

```java
// 在 AST 层面做 SQL 改写的通用模式
public String rewriteSql(String sql, DbType dbType) {
    // 1. 解析
    List<SQLStatement> stmts = SQLUtils.parseStatements(sql, dbType);
    SQLStatement stmt = stmts.get(0);

    // 2. 修改 AST
    if (stmt instanceof SQLSelectStatement) {
        SQLSelectQueryBlock query = ((SQLSelectStatement) stmt).getSelect().getQueryBlock();
        // 在这里修改 AST...
    }

    // 3. 输出
    return SQLUtils.toSQLString(stmt, dbType);
}
```

## 💡 给你的启发

PagerUtils 展示了一种重要的能力：**AST 级别的 SQL 改写**。同样的思路可以用在：

1. **SQL 审计**：自动添加 WHERE 条件（行级权限）
2. **SQL 脱敏**：替换 SELECT 列表中的敏感列
3. **SQL 迁移**：改写方言不兼容的语法
4. **SQL to ES DSL**：解析 SQL 再转化为 ES 查询

```java
// SQL 改写示例：自动添加租户隔离条件
public String addTenantFilter(String sql, String tenantId) {
    List<SQLStatement> stmts = SQLUtils.parseStatements(sql, DbType.mysql);
    SQLStatement stmt = stmts.get(0);

    if (stmt instanceof SQLSelectStatement) {
        SQLSelectQueryBlock query = ((SQLSelectStatement) stmt).getSelect().getQueryBlock();

        // 添加 WHERE tenant_id = 'xxx'
        SQLExpr tenantCondition = new SQLBinaryOpExpr(
            new SQLIdentifierExpr("tenant_id"),
            SQLBinaryOperator.Equality,
            new SQLCharExpr(tenantId)
        );

        if (query.getWhere() == null) {
            query.setWhere(tenantCondition);
        } else {
            // AND 连接到现有条件
            query.setWhere(new SQLBinaryOpExpr(
                query.getWhere(),
                SQLBinaryOperator.BooleanAnd,
                tenantCondition
            ));
        }
    }

    return SQLUtils.toSQLString(stmt, DbType.mysql);
}
```

## 思考题

1. PagerUtils 中的 `count()` 方法是直接拼接 `COUNT(*)` 还是通过 AST 操作？这两种方式各有什么优缺点？
2. Oracle 的分页为什么需要两层子查询？这个 SQL 改写逻辑在 AST 层面是如何实现的？
3. 如果你要在 SQL-to-ES DSL 中支持从 SQL 的 LIMIT/OFFSET 到 ES 的 from/size 的映射，你会怎么做？

## 关键源码路径

| 文件 | 方法 | 说明 |
|------|------|------|
| PagerUtils.java | `count()` | 生成 COUNT SQL |
| PagerUtils.java | `limit()` | 生成分页 SQL |
| PagerUtils.java | `limit(Oracle)` | Oracle 分页处理 |

## 思考题答案

<details>
<summary>点击展开</summary>

1. **PagerUtils 的 `count()` 是用字符串拼接还是 AST 操作？**
   - **AST 操作**。`PagerUtils.count()` 内部先 parse SQL 成 AST，然后替换 SELECT 列表为 `COUNT(*)`，移除 ORDER BY 和 LIMIT，最后通过 OutputVisitor 输出。这种方式比正则替换更可靠，能正确处理子查询、UNION 等复杂场景。

2. **Oracle 的分页为什么需要两层子查询？这个在 AST 层面怎么实现的？**
   - Oracle 不支持 `LIMIT` 和 `OFFSET`。分页需要：
     - 内层子查询：`SELECT t.*, ROWNUM AS rn FROM (原查询) t WHERE ROWNUM <= offset + count`
     - 外层子查询：`SELECT * FROM (内层) WHERE rn > offset`
   - 在 AST 层面：创建一个新的 `SQLSelectStatement`，把原查询作为 `SQLExprTableSource` 放入子查询的 FROM，在 SELECT 列表添加 `ROWNUM AS rn`，在外层再加 WHERE 过滤 `rn > offset`。

3. **SQL-to-ES DSL 中 LIMIT/OFFSET 到 ES from/size 的映射？**
   - `SQLSelectQueryBlock.getLimit()` → `SQLLimit.getRowCount()` → ES 的 `"size"` 字段
   - `SQLLimit.getOffset()`（如果没有 offset，默认为 0）→ ES 的 `"from"` 字段
   - 示例：`LIMIT 10 OFFSET 20` → `{"from": 20, "size": 10}`
</details>

## 下一课预告

**第 19 课：SQL 参数化与模板** — Druid 支持将 SQL 中的字面量替换为参数占位符（`?`），这就是 SQL 参数化。这在 SQL 监控、防火墙、缓存等场景中非常重要。
