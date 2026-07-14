---
lesson: 15
title: SchemaStatVisitor 与 SQL 分析
duration: 45 分钟
objectives:
  - 理解 SchemaStatVisitor 的工作原理
  - 掌握表、列、条件的收集方法
  - 学会使用 SchemaStatVisitor 分析 SQL
  - 理解条件提取和关系分析
prerequisites: 第 13 课
---

# 第 15 课：SchemaStatVisitor 与 SQL 分析

## SchemaStatVisitor 的作用

**文件位置**: `druid/core/src/main/java/com/alibaba/druid/sql/visitor/SchemaStatVisitor.java`

`SchemaStatVisitor` 是 Druid 中最重要的**分析型 Visitor**。它自动遍历 AST 并收集：

- **表** — 涉及了哪些表？操作类型是什么？（SELECT/INSERT/UPDATE/DELETE）
- **列** — 涉及了哪些列？哪些是查询列？哪些是条件列？
- **条件** — 有哪些 WHERE 条件？（列、操作符、值）
- **关系** — 表之间的关联关系（JOIN 条件）
- **排序** — ORDER BY 涉及的列
- **分组** — GROUP BY 涉及的列
- **聚合** — 使用了哪些聚合函数？

```java
public class SchemaStatVisitor extends SQLASTVisitorAdapter {
    protected final List<SQLName> originalTables = new ArrayList<>();
    protected final HashMap<TableStat.Name, TableStat> tableStats = new LinkedHashMap<>();
    protected final Map<Long, Column> columns = new LinkedHashMap<>();
    protected final List<Condition> conditions = new ArrayList<>();
    protected final Set<Relationship> relationships = new LinkedHashSet<>();
    protected final List<Column> orderByColumns = new ArrayList<>();
    protected final Set<Column> groupByColumns = new LinkedHashSet<>();
    protected final List<SQLAggregateExpr> aggregateFunctions = new ArrayList<>();
    protected final List<SQLMethodInvokeExpr> functions = new ArrayList<>();
}
```

## 核心 API 使用

```java
String sql = "SELECT u.id, u.name, o.total FROM users u " +
             "JOIN orders o ON u.id = o.user_id " +
             "WHERE u.age > 18 AND o.status = 'active' " +
             "ORDER BY u.id DESC";

// 创建 Visitor
SchemaStatVisitor visitor = new SchemaStatVisitor(DbType.mysql);

// 解析并应用 Visitor
List<SQLStatement> stmts = SQLUtils.parseStatements(sql, DbType.mysql);
stmts.get(0).accept(visitor);

// 1. 获取所有表
// HashMap<TableStat.Name, TableStat>
visitor.getTables().forEach((tableName, tableStat) -> {
    System.out.println("表: " + tableName + ", 操作: " + tableStat.getMode());
});
// 输出:
//   表: users, 操作: SELECT
//   表: orders, 操作: SELECT

// 2. 获取所有列
// Map<Long, Column>
visitor.getColumns().forEach(col -> {
    System.out.println("列: " + col.getTable() + "." + col.getName());
});
// 输出:
//   列: users.id
//   列: users.name
//   列: orders.total
//   列: users.age
//   列: orders.status

// 3. 获取所有条件
// List<Condition>
visitor.getConditions().forEach(cond -> {
    System.out.println("条件: " + cond.getColumn() + " " + cond.getOperator() + " " + cond.getValues());
});
// 输出:
//   条件: users.age > [18]
//   条件: orders.status = [active]

// 4. 获取关系 (JOIN)
visitor.getRelationships().forEach(rel -> {
    System.out.println("关系: " + rel.getLeft() + " = " + rel.getRight());
});
// 输出:
//   关系: users.id = orders.user_id
```

## 工作原理

### 1. 表信息收集

```java
// SchemaStatVisitor 在遇到 FROM/JOIN 子句时收集表信息
@Override
public boolean visit(SQLExprTableSource x) {
    String tableName = x.getTableName();
    if (tableName != null) {
        // 注册表
        tableStats.computeIfAbsent(new TableStat.Name(tableName), k -> new TableStat());
        // ...
    }
    return true;
}
```

### 2. 列信息收集

```java
@Override
public boolean visit(SQLIdentifierExpr x) {
    String name = x.getName();
    // 找出所属的表
    SQLObject parent = x.getParent();
    String tableName = resolveTableName(parent);

    Column column = new Column(tableName, name);
    columns.put(column.hashCode(), column);
    return true;
}
```

💡 这里的关键是 `resolveTableName()`：通过 AST 的父子关系和别名信息来解析列名所属的表。

### 3. 条件提取

```java
@Override
public boolean visit(SQLBinaryOpExpr x) {
    // 识别 "column op value" 的模式
    if (x.getLeft() instanceof SQLIdentifierExpr
        && isLiteral(x.getRight())) {
        // 收集条件: column = value
        conditions.add(new Condition(
            columnName, x.getOperator().name, value
        ));
    }
    return true;
}
```

## 实战: SQL 到 ES DSL 的条件提取

结合 SchemaStatVisitor，可以轻松提取 SQL 的查询条件并映射到 ES：

```java
public class EsQueryBuilder {
    public static void main(String[] args) {
        String sql = "SELECT * FROM users WHERE age > 18 AND status = 'active' AND city IN ('北京', '上海') ORDER BY age DESC LIMIT 20";

        // 解析 SQL
        List<SQLStatement> stmts = SQLUtils.parseStatements(sql, DbType.mysql);
        SQLSelectStatement stmt = (SQLSelectStatement) stmts.get(0);

        // 用 SchemaStatVisitor 分析
        SchemaStatVisitor visitor = new SchemaStatVisitor(DbType.mysql);
        stmt.accept(visitor);

        // 打印 ES 查询结构
        System.out.println("=== ES Query ===");
        printEsQuery(stmt, visitor);
    }

    static void printEsQuery(SQLSelectStatement stmt, SchemaStatVisitor visitor) {
        // Index
        SQLTableSource from = stmt.getSelect().getQueryBlock().getFrom();
        System.out.println("索引: " + extractTableName(from));

        // Size & Sort
        SQLLimit limit = stmt.getSelect().getQueryBlock().getLimit();
        if (limit != null) {
            System.out.println("Size: " + limit.getRowCount());
        }

        SQLOrderBy orderBy = stmt.getSelect().getQueryBlock().getOrderBy();
        if (orderBy != null) {
            System.out.println("Sort: " + orderBy);
        }

        // Query (conditions)
        System.out.println("Query:");
        for (TableStat.Condition cond : visitor.getConditions()) {
            String col = extractColumnName(cond.getColumn());
            String op = cond.getOperator();
            Object val = cond.getValues().isEmpty() ? null : cond.getValues().get(0);

            if ("=".equals(op)) {
                System.out.println("  term: { \"" + col + "\": \"" + val + "\" }");
            } else if (">".equals(op)) {
                System.out.println("  range: { \"" + col + "\": { \"gt\": " + val + " } }");
            } else if ("<".equals(op)) {
                System.out.println("  range: { \"" + col + "\": { \"lt\": " + val + " } }");
            }
        }
    }
}
```

## SchemaStatVisitor 的方言版本

各数据库方言有自己的 SchemaStatVisitor，处理方言特有的表/列引用：

```java
// MySQL 版本
new MySqlSchemaStatVisitor();

// Oracle 版本
new OracleSchemaStatVisitor();

// PG 版本
new PGSchemaStatVisitor();
```

```java
// 便捷创建
SchemaStatVisitor visitor = SQLUtils.createSchemaStatVisitor(DbType.mysql);
```

## 与 SQLUtils 的整合

SQLUtils 提供了便捷方法：

```java
// 方式 1：手动创建 Visitor
SchemaStatVisitor visitor = SQLUtils.createSchemaStatVisitor(DbType.mysql);
List<SQLStatement> stmts = SQLUtils.parseStatements(sql, DbType.mysql);
stmts.get(0).accept(visitor);

// 方式 2：获取表统计
Map<TableStat.Name, TableStat> tables = visitor.getTables();

// 方式 3：获取列统计
Collection<Column> cols = visitor.getColumns();
```

## 💡 在 SQL-to-ES DSL 中的应用

SchemaStatVisitor 是你实现 SQL-to-ES DSL 的**最强助手**：

1. **表的提取** → 确定 ES 索引名
2. **列的提取** → 确定 _source 字段
3. **条件的提取** → 构建 ES bool/filter/term/range 查询
4. **关系的提取** → 处理 ES nested/has_child 查询
5. **ORDER BY** → ES sort
6. **LIMIT** → ES size

```java
// 典型的映射流程
SQL Select → ES SearchRequest
  columns → _source
  from → index
  where → query.bool
  order by → sort
  limit → size
```

## 🔍 动手探索

1. 在 `SchemaStatVisitor.java` 中找到 `visit(SQLBinaryOpExpr)` 方法，看条件提取的具体逻辑
2. 找到 `visit(SQLExprTableSource)` 方法，看表信息如何被注册
3. 查看 `MySqlSchemaStatVisitor`，看它和基类有何不同

## 思考题

1. SchemaStatVisitor 如何解决列名歧义（如 `u.id` 和 `o.id`）？它通过什么机制确定别名对应的真实表名？
2. SchemaStatVisitor 收集的 `conditions` 中，`=` 条件是否包含 `JOIN ... ON` 中的条件？为什么？
3. 如果你要写一个 SchemaStatVisitor 用于 ES DSL，需要扩展它的哪些能力？

## 关键源码路径

| 文件 | 方法 | 说明 |
|------|------|------|
| SchemaStatVisitor.java | `visit(SQLExprTableSource)` | 表信息收集 |
| SchemaStatVisitor.java | `visit(SQLIdentifierExpr)` | 列信息收集 |
| SchemaStatVisitor.java | `visit(SQLBinaryOpExpr)` | 条件提取 |
| SchemaStatVisitor.java | `getTables/getColumns/getConditions` | 访问收集结果 |

## 下一课预告

**第 16 课：SQLBuilder 与编程式 SQL 构建** — 除了从 SQL 字符串解析得到 AST，Druid 还支持通过编程方式直接构建 AST（即 Builder 模式）。这在需要动态构建 SQL 的场景下非常有用。
