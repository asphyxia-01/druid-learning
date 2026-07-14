---
lesson: 7
title: SQLStatementParser 语句解析
duration: 50 分钟
objectives:
  - 理解 SQLStatementParser 的分发机制
  - 掌握 DML/DDL 语句的解析入口
  - 理解 parseStatementList() 的多语句支持
  - 学会阅读 INSERT/UPDATE/DELETE 的解析流程
prerequisites: 第 6 课
---

# 第 7 课：SQLStatementParser 语句解析

## SQLStatementParser 概述

**文件位置**: `druid/core/src/main/java/com/alibaba/druid/sql/parser/SQLStatementParser.java`

SQLStatementParser 是 Druid 中最**重要**的解析器。它负责将 Token 序列解析为完整的 SQL 语句 AST。

```java
// SQLStatementParser.java:49
public class SQLStatementParser extends SQLParser {
    protected SchemaRepository repository;  // Schema 元数据仓库
    protected SQLExprParser exprParser;      // 表达式解析器委托
    protected SQLSelectListCache selectListCache;  // SELECT 列表缓存（性能优化）
}
```

## 语句分发机制

SQLStatementParser 根据**当前 Token** 来决定解析哪种语句：

```java
// SQLStatementParser.java (核心方法)
public SQLStatement parseStatement() {
    switch (lexer.token) {
        case SELECT:   return parseSelect();     // → SQLSelectParser
        case INSERT:   return parseInsert();     // → parseInsert()
        case UPDATE:   return parseUpdate();     // → parseUpdate()
        case DELETE:   return parseDelete();     // → parseDelete()
        case CREATE:   return parseCreate();     // → SQLDDLParser
        case ALTER:    return parseAlter();      // → parseAlter()
        case DROP:     return parseDrop();       // → parseDrop()
        case TRUNCATE: return parseTruncate();   // → parseTruncate()
        case SET:      return parseSet();        // → parseSet()
        default:
            // 尝试解析为表达式或其他
            if (lexer.token == Token.LPAREN) {
                return parseSelect();
            }
            throw new ParserException("not supported statement: " + lexer.token);
    }
}
```

### 多语句解析

```java
// SQLStatementParser.java
public void parseStatementList(List<SQLStatement> statements, int max, List<String> expectedTypes) {
    while (true) {
        // 处理空语句（连续分号）
        if (lexer.token == Token.SEMI) {
            lexer.nextToken();
            continue;
        }

        if (lexer.token == Token.EOF) {
            return;
        }

        // 解析一条语句
        SQLStatement stmt = parseStatement();

        // 处理语句后的分号
        if (lexer.token == Token.SEMI) {
            stmt.setAfterSemi(true);
            lexer.nextToken();
        }

        statements.add(stmt);

        // 达到最大数量则停止
        if (max > 0 && statements.size() >= max) {
            return;
        }
    }
}
```

## INSERT 语句解析

```java
// SQLStatementParser.java
public SQLInsertStatement parseInsert() {
    accept(Token.INSERT);

    if (lexer.token == Token.INTO) {
        lexer.nextToken();
    }

    // 解析表名
    SQLName tableName = exprParser.name();

    // 解析列名列表: (col1, col2, ...)
    List<SQLExpr> columns = new ArrayList<>();
    if (lexer.token == Token.LPAREN) {
        // 解析列列表
    }

    // 解析 VALUES 或 SELECT
    if (lexer.token == Token.VALUES) {
        // VALUES (v1, v2), (v3, v4)
        return parseInsertValues(stmt);
    } else if (lexer.token == Token.SELECT) {
        // INSERT INTO ... SELECT ...
        stmt.setQuery(selectParser.select());
    }

    return stmt;
}
```

## UPDATE 语句解析

```java
// SQLStatementParser.java
public SQLUpdateStatement parseUpdate() {
    accept(Token.UPDATE);

    // 解析表名
    SQLTableSource tableSource = parseTableSource();

    // 解析 SET 子句
    accept(Token.SET);
    List<SQLUpdateSetItem> items = new ArrayList<>();
    while (true) {
        SQLUpdateSetItem item = new SQLUpdateSetItem();
        item.setColumn(exprParser.expr());     // 列名
        accept(Token.EQ);
        item.setValue(exprParser.expr());      // 值
        items.add(item);

        if (lexer.token != Token.COMMA) break;
        lexer.nextToken();
    }

    // 可选的 WHERE 子句
    if (lexer.token == Token.WHERE) {
        stmt.setWhere(parseWhere());
    }

    return stmt;
}
```

## DELETE 语句解析

```java
// SQLStatementParser.java
public SQLDeleteStatement parseDelete() {
    accept(Token.DELETE);

    // 可选的 FROM
    if (lexer.token == Token.FROM) {
        lexer.nextToken();
    }

    // 解析表名
    SQLName tableName = exprParser.name();

    // 可选的 WHERE
    if (lexer.token == Token.WHERE) {
        stmt.setWhere(parseWhere());
    }

    return stmt;
}
```

## 通用 WHERE 解析

WHERE 子句在 SELECT/UPDATE/DELETE 中都会出现，因此提取为公共方法：

```java
// SQLStatementParser.java
protected SQLExpr parseWhere() {
    if (lexer.token == Token.WHERE) {
        lexer.nextToken();
        return exprParser.expr();  // ★ 委托给表达式解析器
    }
    return null;
}
```

💡 WHERE 条件的解析完全委托给 `exprParser.expr()`，因为 `age > 18 AND status = 'active'` 本质上就是一个**布尔表达式**。

## CREATE/ALTER/DROP 语句解析

这些 DDL 语句在 SQLDDLParser 中处理：

```java
// SQLDDLParser.java
public class SQLDDLParser extends SQLStatementParser {
    public SQLCreateTableStatement parseCreateTable() {
        // CREATE TABLE ...
    }

    public SQLAlterTableStatement parseAlterTable() {
        // ALTER TABLE ...
    }
}
```

而 `SQLCreateTableParser` 更进一步处理建表语句的细节：

```java
// SQLCreateTableParser.java
public class SQLCreateTableParser extends SQLDDLParser {
    public void parseCreateTable(SQLCreateTableStatement stmt) {
        // 解析列定义: col_name data_type [constraints]
        // 解析表选项: ENGINE=InnoDB, CHARSET=utf8
        // 解析索引定义: INDEX idx_name (col)
    }
}
```

## 完整的语句解析流程

```java
String dml = "INSERT INTO users (id, name) VALUES (1, 'Alice'); UPDATE users SET name='Bob' WHERE id = 1";

// Druid 内部处理流程:
// 1. Lexer: INSERT → INTO → IDENTIFIER(users) → LPAREN → ...
// 2. parseStatementList():
//    - token=INSERT → parseInsert()
//      → SQLInsertStatement
//    - token=SEMI → 消费分号
//    - token=UPDATE → parseUpdate()
//      → SQLUpdateStatement
//    - token=EOF → 返回
// 3. 返回 [SQLInsertStatement, SQLUpdateStatement]
```

## 💡 设计思想

SQLStatementParser 的设计遵循了 **Template Method** 模式：

1. **parseStatement()** 是模板方法，负责分发
2. **parseInsert/Update/Delete** 是具体步骤，各司其职
3. **方言子类**（如 MySqlStatementParser）可以覆盖特定方法添加方言特性

这种设计让基类处理 80% 的通用逻辑，方言子类只需关注 20% 的特有语法。

## 🔍 动手探索

1. 在 `SQLStatementParser.java` 中找到 `parseStatement()` 的完整 switch 语句
2. 找到 `parseInsert()` 方法，跟踪它对多行 INSERT 的处理
3. 查看 `SQLDDLParser` 和 `SQLCreateTableParser` 的继承关系

## 思考题

1. `parseStatementList()` 中如何处理空语句（连续分号）和语法错误？
2. `parseInsert()` 中 `VALUES (...)` 和 `SELECT` 分支的处理有什么不同？
3. 如果你要添加一个 `MERGE INTO` 语句的解析，需要修改哪些文件？流程是怎样的？

## 关键源码路径

| 文件 | 方法 | 说明 |
|------|------|------|
| SQLStatementParser.java | `parseStatement()` | 语句分发 |
| SQLStatementParser.java | `parseStatementList()` | 多语句解析 |
| SQLStatementParser.java | `parseInsert()` | INSERT 解析 |
| SQLStatementParser.java | `parseUpdate()` | UPDATE 解析 |
| SQLStatementParser.java | `parseDelete()` | DELETE 解析 |
| SQLStatementParser.java | `parseWhere()` | WHERE 解析 |
| SQLDDLParser.java | `parseCreateTable()` | CREATE TABLE 解析 |

## 下一课预告

**第 8 课：SQLSelectParser SELECT 解析深度解析** — SELECT 是 SQL 中最复杂的语句，涉及子句多、嵌套层级深。我们将深入 SQLSelectParser，分析它是如何处理 WITH、SELECT 列表、FROM、JOIN、WHERE、GROUP BY、HAVING、ORDER BY、LIMIT 等子句的。
