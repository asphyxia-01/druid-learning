---
lesson: 8
title: SQLSelectParser SELECT 解析深度解析
duration: 55 分钟
objectives:
  - 理解 SELECT 语句的完整解析流程
  - 掌握各个子句的解析顺序
  - 理解子查询和 JOIN 的处理
  - 学会阅读 SQLSelectQueryBlock 结构
prerequisites: 第 7 课
---

# 第 8 课：SQLSelectParser SELECT 解析深度解析

## SELECT 是 SQL 中最复杂的语句

```sql
SELECT [DISTINCT] select_list
  FROM table_source
  [JOIN ...]
  [WHERE search_condition]
  [GROUP BY group_by_expression]
  [HAVING search_condition]
  [ORDER BY order_expression [ASC|DESC]]
  [LIMIT {[offset,] count}]
```

**文件位置**: `druid/core/src/main/java/com/alibaba/druid/sql/parser/SQLSelectParser.java`

## SELECT 解析入口

```java
// SQLSelectParser.java:55
public SQLSelect select() {
    SQLSelect select = new SQLSelect();

    // 1. 处理 WITH 子句 (CTE)
    if (lexer.token == Token.WITH) {
        SQLWithSubqueryClause with = this.parseWith();
        select.setWithSubQuery(with);
    }

    // 2. 解析查询主体
    SQLSelectQuery query = query(select, true);
    select.setQuery(query);

    // 3. 处理 ORDER BY (有 UNION 时可能在外面)
    if (lexer.token == Token.ORDER) {
        select.setOrderBy(parseOrderBy());
    }

    // 4. 处理 LIMIT (有 UNION 时可能在外面)
    if (lexer.token == Token.LIMIT) {
        select.setLimit(parseLimit());
    }

    return select;
}
```

## 查询主体: query() 方法

`query()` 方法承担了两项任务：创建一个 `SQLSelectQueryBlock` 并解析其所有子句，然后检查是否有 UNION 进行组合。

```java
// SQLSelectParser.java:450
public SQLSelectQuery query(SQLObject parent, boolean acceptUnion) {
    SQLSelectQueryBlock queryBlock = createSelectQueryBlock();
    accept(Token.SELECT);

    // 处理 DISTINCT
    if (lexer.token == Token.DISTINCT || lexer.token == Token.DISTINCTROW) {
        queryBlock.setDistionOption(SQLSetQuantifier.DISTINCT);
        lexer.nextToken();
    }

    // 1. 解析 SELECT 列表
    if (lexer.token != Token.FROM) {
        parseSelectList(queryBlock);  // ★ 关键: 解析列
    }

    // 2. 解析 FROM 子句
    if (lexer.token == Token.FROM) {
        lexer.nextToken();
        queryBlock.setFrom(parseTableSource(), this.exprParser);  // ★ 解析表
    }

    // 3. 解析 WHERE
    if (lexer.token == Token.WHERE) {
        lexer.nextToken();
        queryBlock.setWhere(exprParser.expr());  // 条件表达式
    }

    // 4. 解析 GROUP BY
    if (lexer.token == Token.GROUP) {
        lexer.nextToken();
        accept(Token.BY);
        queryBlock.setGroupBy(parseGroupBy());
    }

    // 5. 解析 HAVING
    if (lexer.token == Token.HAVING) {
        lexer.nextToken();
        queryBlock.setHaving(exprParser.expr());
    }

    // 6. 解析 ORDER BY（内层 — 也支持外层）
    if (lexer.token == Token.ORDER) {
        queryBlock.setOrderBy(parseOrderBy());
    }

    // 7. 解析 LIMIT（内层 — 也支持外层）
    if (lexer.token == Token.LIMIT) {
        queryBlock.setLimit(exprParser.parseLimit());
    }

    // 返回后检查 UNION
    return queryRest(queryBlock, acceptUnion);
}
```

> 💡 Druid 没有单独的 `queryBlock()` 方法，SELECT 查询块的解析是 `query()` 方法的一部分。`query()` 方法先创建 SQLSelectQueryBlock，然后依次处理各子句。

### 解析 SELECT 列表

```java
// SQLSelectParser.java
protected void parseSelectList(SQLSelectQueryBlock queryBlock) {
    while (true) {
        // 解析单个列
        SQLSelectItem item = parseSelectItem();
        queryBlock.getSelectList().add(item);

        if (lexer.token == Token.COMMA) {
            lexer.nextToken();    // 逗号分隔
        } else {
            break;                // 列表结束
        }
    }
}

// 解析单个列: expr [AS] alias
protected SQLSelectItem parseSelectItem() {
    SQLExpr expr = exprParser.expr();  // 解析表达式

    // 处理 AS 别名
    String alias = null;
    if (lexer.token == Token.AS) {
        lexer.nextToken();
        alias = lexer.stringVal();
        lexer.nextToken();
    } else if (lexer.token == Token.LITERAL_ALIAS) {
        alias = lexer.stringVal();
        lexer.nextToken();
    }

    return new SQLSelectItem(expr, alias);
}
```

## FROM 子句与表源解析

FROM 子句涉及表引用、JOIN、子查询等多种情况：

```java
// SQLStatementParser.java
protected SQLTableSource parseTableSource() {
    // 1. 解析当前表
    SQLTableSource tableSource = parseSingleTableSource();

    // 2. 检查是否有 JOIN
    while (true) {
        if (lexer.token == Token.COMMA) {
            // 隐式 JOIN (逗号分隔)
            lexer.nextToken();
            SQLTableSource right = parseSingleTableSource();
            tableSource = new SQLJoinTableSource(tableSource, right, JoinType.COMMA);
        } else if (lexer.token == Token.LEFT || lexer.token == Token.RIGHT
                || lexer.token == Token.FULL || lexer.token == Token.INNER
                || lexer.token == Token.JOIN || lexer.token == Token.STRAIGHT_JOIN) {
            // 显式 JOIN
            SQLJoinTableSource.JoinType joinType = parseJoinType();
            // ...
        } else {
            break;
        }
    }

    return tableSource;
}
```

## UNION 解析

```java
// SQLSelectParser.java:query 方法的后半段
public SQLSelectQuery query(SQLSelect select, boolean parent) {
    SQLSelectQuery query = queryBlock();

    if (lexer.token == Token.UNION) {
        lexer.nextToken();
        SQLUnionQuery union = new SQLUnionQuery();
        union.setLeft(query);

        if (lexer.token == Token.ALL) {
            union.setOperator(SQLUnionOperator.UNION_ALL);
            lexer.nextToken();
        } else {
            union.setOperator(SQLUnionOperator.UNION);
        }

        union.setRight(query(select, false));  // 递归解析
        query = union;
    }

    return query;
}
```

## ORDER BY 解析

```java
// SQLSelectParser.java
protected SQLOrderBy parseOrderBy() {
    accept(Token.ORDER);
    accept(Token.BY);

    SQLOrderBy orderBy = new SQLOrderBy();
    while (true) {
        SQLExpr expr = exprParser.expr();
        String type = null;

        if (lexer.token == Token.ASC) {
            type = "ASC";
            lexer.nextToken();
        } else if (lexer.token == Token.DESC) {
            type = "DESC";
            lexer.nextToken();
        }

        SQLSelectOrderByItem item = new SQLSelectOrderByItem(expr, type);
        orderBy.addItem(item);

        if (lexer.token == Token.COMMA) {
            lexer.nextToken();
        } else {
            break;
        }
    }

    return orderBy;
}
```

## 完整的 SELECT 解析流程

以 `SELECT t.id, COUNT(*) AS cnt FROM users t JOIN orders o ON t.id = o.user_id WHERE t.age > 18 GROUP BY t.id HAVING cnt > 1 ORDER BY cnt DESC LIMIT 10` 为例：

```
SQLSelectParser.select()
  ├── WITH? → 无
  ├── query()
  │   └── queryBlock():
  │       ├── SELECT → 解析 selectList
  │       │   ├── SQLAllColumnExpr(t.id)
  │       │   └── SQLAggregateExpr(COUNT(*)) AS "cnt"
  │       ├── FROM → 解析 tableSource
  │       │   ├── users t
  │       │   └── JOIN orders o ON ...
  │       ├── WHERE → exprParser.expr() → t.age > 18
  │       ├── GROUP BY → t.id
  │       ├── HAVING → exprParser.expr() → cnt > 1
  │       ├── ORDER BY → cnt DESC
  │       └── LIMIT → 10
  ├── 外部 ORDER BY? → 无
  └── 外部 LIMIT? → 无
```

## 💡 设计思想

SQLSelectParser 的关键设计决策：

1. **分层解析**: 先解析"查询块"（query 方法内部），再处理"组合"(UNION)
2. **懒惰委托**: WHERE/HAVING 条件完全交给 `exprParser.expr()`
3. **内外分层**: ORDER BY 和 LIMIT 可以出现在 queryBlock 内部或外部（UNION 时在外部）

## 🔍 动手探索

1. 在 `SQLSelectParser.query()` 方法中找到所有子句的解析顺序
2. 找到 UNION 的解析逻辑，理解左递归的处理
3. 查看 `parseTableSource()` 中 JOIN 的处理逻辑

## 思考题

1. 为什么 WHERE 和 HAVING 的解析完全委托给 `exprParser.expr()`？它们之间有什么约束？
2. UNION 解析中 `union.setRight(query(select, false))` 为什么递归调用 `query()`？
3. 如果你的 SQL to ES DSL 需要解析 `SELECT`，你会复用 SQLSelectParser 还是自己实现一个简化版？为什么？

## 关键源码路径

| 文件 | 方法 | 说明 |
|------|------|------|
| SQLSelectParser.java:55 | `select()` | SELECT 入口 |
| SQLSelectParser.java:450 | `query()` | 查询块解析（含各子句） |
| SQLSelectParser.java:~150 | `parseSelectList()` | SELECT 列表 |
| SQLStatementParser.java | `parseTableSource()` | FROM + JOIN |
| SQLSelectParser.java:~200 | `parseOrderBy()` | ORDER BY |
| SQLSelectParser.java:~250 | `query()` UNION | UNION 处理 |

## 思考题答案

<details>
<summary>点击展开</summary>

1. **为什么 WHERE 和 HAVING 的解析完全委托给 `exprParser.expr()`？**
   - WHERE 和 HAVING 的条件本质上就是**布尔表达式**：`age > 18 AND status = 'active'`。这和 `a + b * c` 在解析逻辑上没有本质区别——都是操作符、操作数、优先级的问题。
   - 区别在于语义层面（WHERE 是行级过滤，HAVING 是分组后过滤），但这不影响解析。语义检查可以在后续的 Visitor 或 SchemaRepository 阶段完成。

2. **UNION 解析中为什么递归调用 `query()`？**
   - 因为 UNION 可以链式组合：`SELECT a UNION SELECT b UNION SELECT c`。
   - 第一次 UNION：left=SELECT a, right=SELECT b
   - 第二次 UNION：left=(SELECT a UNION SELECT b), right=SELECT c
   - 每次遇到 UNION 时，把当前 query 作为 left，递归解析下一个 query 作为 right，然后把 UNION 整体作为新的 left。
   - 这形成了右递归：`Union(Union(SELECT a, SELECT b), SELECT c)`。

3. **你的 SQL to ES DSL，会复用 SQLSelectParser 还是自己实现简化版？**
   - **复用 SQLSelectParser**。原因：SQL 的完整 SELECT 语法（子查询、JOIN、UNION、CTE）写一个简化版 Parser 也要处理同样多的边界情况。Druid 已经把这些都处理好了，你只需要在生成的 AST 上做 Visitor 翻译。唯一需要自己实现的是那些 ES DSL 特有但 SQL 不直接表达的概念（如 `match_phrase`、`nested` 查询），这些不需要改 Parser，在 Visitor 里做映射就行。
</details>

## 下一课预告

**第 9 课：方言解析器实现（以 MySQL 为例）** — 我们将深入 MySQL 方言的解析器，看看 Druid 如何通过继承和扩展来支持不同数据库的特有语法。这为你将来扩展自己的"ES DSL 方言"提供了直接的参考。
