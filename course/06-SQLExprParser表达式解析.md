---
lesson: 6
title: SQLExprParser 表达式解析
duration: 55 分钟
objectives:
  - 理解 SQLExprParser 的职责范围
  - 掌握运算符优先级处理
  - 理解聚合函数和非聚合函数的解析
  - 学会阅读表达式解析的核心流程
prerequisites: 第 5 课
---

# 第 6 课：SQLExprParser 表达式解析

## 什么是表达式？

在 SQL 中，几乎所有的"值"和"计算"都是表达式：

```sql
-- 列引用
id, name, t.age

-- 字面量
1, 'hello', 3.14, true

-- 运算
a + b, price * quantity, age > 18

-- 函数调用
COUNT(*), SUBSTR(name, 1, 3), NOW()

-- 子查询
(SELECT max(id) FROM users)

-- CASE WHEN
CASE WHEN score > 90 THEN 'A' ELSE 'B' END
```

**文件位置**: `druid/core/src/main/java/com/alibaba/druid/sql/parser/SQLExprParser.java`

## SQLExprParser 的继承链

```
SQLParser
  └── SQLExprParser       ← 通用表达式解析
        ├── MySqlExprParser
        ├── OracleExprParser
        ├── PGExprParser
        └── ... (每种方言一个)
```

## 核心入口: expr() 方法

```java
// SQLExprParser.java:82 (简化)
public SQLExpr expr() {
    if (lexer.token() == Token.SELECT || lexer.token() == Token.LPAREN) {
        // 可能是子查询
        // ...
    }
    return expr(0);  // 从最低优先级开始解析
}

public SQLExpr expr(int priorities) {
    // 1. 解析左操作数
    SQLExpr left = primaryExpr();

    // 2. 根据运算符优先级，决定是否继续解析二元运算
    if (left != null) {
        return exprRest(left, priorities);
    }
    return left;
}
```

### 优先级驱动

`exprRest()` 是表达式解析的**核心方法**，它使用运算符优先级来正确结合表达式：

```java
// SQLExprParser.java (简化)
protected SQLExpr exprRest(SQLExpr expr, int priorities) {
    while (true) {
        // 获取当前运算符的优先级
        int priority = getPriority(lexer.token());

        if (priority < priorities) {
            break;  // 优先级不够，返回
        }

        // 解析二元操作
        Token op = lexer.token();
        lexer.nextToken();

        // 解析右操作数（使用更高的优先级）
        SQLExpr right = expr(priority + 1);

        // 构建二元操作表达式
        expr = new SQLBinaryOpExpr(expr, op, right, dbType);
    }
    return expr;
}
```

### ⚠️ 优先级陷阱

考虑 `a + b * c`：
```
解析流程:
1. expr(0) → primaryExpr() → a
2. exprRest(a, 0):
   - token='+', priority=30
   - 30 >= 0, 消费 '+'
   - expr(31): primaryExpr() → b
   - exprRest(b, 31):
     - token='*', priority=40
     - 40 >= 31, 消费 '*'
     - expr(41): primaryExpr() → c
     - exprRest(c, 41): 没有更高优先级 → 返回 c
     - 返回 b * c  → SQLBinaryOpExpr(*, b, c)
   - 返回 a + (b * c) → SQLBinaryOpExpr(+, a, SQLBinaryOpExpr(*, b, c))
```

结果: `a + b * c` 被正确解析为 `a + (b * c)`。

## primaryExpr(): 解析原子表达式

`primaryExpr()` 是表达式解析的**基础方法**，它处理各种"原子"表达式：

```java
// SQLExprParser.java (简化)
public SQLExpr primaryExpr() {
    switch (lexer.token()) {
        case LITERAL_INT:
            return parseInteger();       // 整数: 1, 100
        case LITERAL_FLOAT:
            return parseFloat();         // 浮点数: 3.14
        case LITERAL_CHARS:
            return parseString();        // 字符串: 'hello'
        case NULL:
            return parseNull();          // NULL
        case TRUE:
        case FALSE:
            return parseBoolean();       // true/false
        case CASE:
            return parseCase();          // CASE WHEN ...
        case CAST:
            return parseCast();          // CAST(... AS ...)
        case LPAREN:
            return parseBracket();       // (expr)
        case IDENTIFIER:
            return parseIdentifier();    // 标识符/函数调用
        case VARIANT:
            return parseVariant();       // @变量
        // ...
    }
}
```

### 标识符解析（最重要的分支）

当遇到 `IDENTIFIER` Token 时，它可能是：
1. 普通列名 — `id`, `name`
2. 带限定的列名 — `t.id`
3. 函数调用 — `count(*)`
4. 聚合函数 — `SUM(price)`
5. 内置函数 — `NOW()`

```java
// SQLExprParser.java (简化)
protected SQLExpr parseIdentifier() {
    String ident = lexer.stringVal();
    lexer.nextToken();

    if (lexer.token() == Token.LPAREN) {
        // 是函数调用
        if (isAggregateFunction(ident)) {
            return parseAggregateFunction(ident);  // ★ 聚合函数
        } else {
            return parseFunction(ident);           // ★ 普通函数
        }
    } else if (lexer.token() == Token.DOT) {
        // 是限定列名: t.id
        lexer.nextToken();
        return new SQLPropertyExpr(ident, lexer.stringVal());
    }

    return new SQLIdentifierExpr(ident);  // 普通标识符
}
```

### 💡 区分聚合函数 vs 普通函数

```java
static {
    AGGREGATE_FUNCTIONS = {"AVG", "COUNT", "MAX", "MIN", "STDDEV", "SUM"};
    AGGREGATE_FUNCTIONS_CODES = FnvHash.fnv1a_64_lower(AGGREGATE_FUNCTIONS, true);
}

protected boolean isAggregateFunction(String name) {
    long hash = FnvHash.fnv1a_64_lower(name);
    return Arrays.binarySearch(aggregateFunctionHashCodes, hash) >= 0;
}
```

聚合函数和普通函数的返回类型不同：
- **聚合函数** → `SQLAggregateExpr`（有 `option` 字段：DISTINCT/ALL）
- **普通函数** → `SQLMethodInvokeExpr`（有 `resolvedReturnDataType`）

## 特殊表达式解析

### 1. BETWEEN

```java
// SQLExprParser.java
protected SQLExpr parseBetween() {
    // BETWEEN a AND b
    lexer.nextToken(); // 消费 BETWEEN
    SQLExpr begin = expr();
    accept(Token.AND);
    SQLExpr end = expr();
    return new SQLBetweenExpr(expr, begin, end);
}
```

### 2. IN

```java
protected SQLExpr parseIn() {
    // IN (v1, v2, ...) 或 IN (subquery)
    lexer.nextToken(); // 消费 IN
    accept(Token.LPAREN);
    if (lexer.token() == Token.SELECT) {
        SQLSelect select = selectParser.select();
        expr = new SQLInSubQueryExpr(expr, select);
    } else {
        List<SQLExpr> list = new ArrayList<>();
        // 解析列表
        expr = new SQLInListExpr(expr, list);
    }
    accept(Token.RPAREN);
    return expr;
}
```

## 方言扩展示例：MySqlExprParser

```java
// druid/core/src/main/java/.../mysql/parser/MySqlExprParser.java
public class MySqlExprParser extends SQLExprParser {
    public MySqlExprParser(String sql) {
        this(new MySqlLexer(sql));
    }

    // MySQL 特定的表达式解析
    protected SQLExpr parseType() {
        // MySQL 的类型转换: CAST(expr AS type)
        if (lexer.token() == Token.DATA_TYPE) {
            // ...
        }
    }

    // MySQL 的赋值运算 :=
    protected SQLExpr parseAssignment() {
        if (lexer.token() == Token.COLON && lexer.nextChar() == '=') {
            // := 赋值
        }
    }
}
```

## 完整的表达式解析示例

```java
String exprSql = "a + b * c > 10 AND status = 'active'";

SQLExprParser parser = new SQLExprParser(exprSql, DbType.mysql);
SQLExpr expr = parser.expr();

System.out.println("表达式类型: " + expr.getClass().getSimpleName());
// SQLBinaryOpExpr (AND)

System.out.println("表达式树: ");
System.out.println(SQLUtils.toSQLString(expr, DbType.mysql));
// a + b * c > 10 AND status = 'active'
```

`SQLUtils` 也提供了快捷方法：

```java
SQLExpr expr = SQLUtils.toSQLExpr("a + b * c > 10", DbType.mysql);
```

## 🔍 动手探索

1. 在 `SQLExprParser.exprRest()` 中打断点，观察优先级如何影响解析结果
2. 找到 `getPriority()` 方法，查看所有运算符的优先级定义
3. 在 `primaryExpr()` 中找到 `CASE WHEN` 的解析分支

## 思考题

1. 运算符优先级的数值是如何设计的？为什么特定数字被分配给特定的运算符？（提示：看 `getPriority()` 实现）
2. `parseIdentifier()` 中，为什么函数调用和列引用都在这个方法中处理？它们在 AST 中如何区分？
3. 如果我想让 Druid 支持 ES 的 `match(field, value)` 函数，需要修改哪些代码？

## 关键源码路径

| 文件 | 方法 | 说明 |
|------|------|------|
| SQLExprParser.java:82 | `expr()` | 入口：解析表达式 |
| SQLExprParser.java:~100 | `exprRest()` | 优先级驱动的二元运算解析 |
| SQLExprParser.java:~200 | `primaryExpr()` | 原子表达式解析 |
| SQLExprParser.java:~350 | `parseIdentifier()` | 标识符/函数调用解析 |
| SQLExprParser.java:~500 | `getPriority()` | 运算符优先级定义 |

## 下一课预告

**第 7 课：SQLStatementParser 语句解析** — 表达式解析是"零件"，语句解析就是"组装"。SQLStatementParser 负责将 Token 序列解析为完整的 SQL 语句 AST，包括 SELECT、INSERT、UPDATE、DELETE 等。我们将剖析它的分发机制和解析流程。
