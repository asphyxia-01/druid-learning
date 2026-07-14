---
lesson: 6
title: SQLExprParser 表达式解析
duration: 55 分钟
objectives:
  - 理解 SQLExprParser 的职责范围
  - 掌握 *Rest() 链式优先级解析机制
  - 理解 primary() 如何解析原子表达式
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

## 核心流程概览

表达式解析的入口是 `expr()` 方法，核心流程只有三步：

```java
// SQLExprParser.java:85
public SQLExpr expr() {
    // 1. 解析原子表达式（字面量、标识符、函数调用等）
    SQLExpr expr = primary();

    // 2. 处理后缀操作（.属性访问、[]下标、函数调用参数等）
    expr = primaryRest(expr);

    // 3. 按优先级链处理二元运算符（+、-、*、/、AND、OR 等）
    expr = exprRest(expr);

    return expr;
}
```

这一条表达式解析链覆盖了 SQL 中所有可能的表达式形式。

## 第一步：primary() — 原子表达式

**源码位置**: SQLExprParser.java:464

`primary()` 是表达式解析的"基础层"，它处理各种"不可再分"的原子表达式：

```java
public SQLExpr primary() {
    SQLExpr expr = null;
    final Token tok = lexer.token();

    switch (tok) {
        case LITERAL_INT:
            // 整数: 1, 100
            return new SQLIntegerExpr(lexer.integerValue());

        case LITERAL_FLOAT:
            // 浮点数: 3.14, 1e10
            return lexer.numberExpr();

        case LITERAL_CHARS:
            // 字符串: 'hello'
            expr = new SQLCharExpr(lexer.stringVal());
            lexer.nextToken();
            return primaryLiteralCharsRest(expr);

        case NULL:
            lexer.nextToken();
            return new SQLNullExpr();

        case TRUE:
        case FALSE:
            lexer.nextToken();
            return new SQLBooleanExpr(tok == Token.TRUE);

        case CASE:
            // CASE WHEN ... THEN ... ELSE ... END
            return parseCase();

        case LPAREN:
            // (expr) 或 (subquery)
            return primaryLParen();

        case IDENTIFIER:
            // 标识符/函数调用/关键字被用作标识符
            return primaryIdentifier();

        case VARIANT:
            // @变量
            return primaryVariant();

        case SUB:
            // 一元负号: -5
            lexer.nextToken();
            SQLExpr unaryExpr = primary();
            return new SQLUnaryExpr(SQLUnaryOperator.Negative, unaryExpr);

        case NOT:
            return primaryNot();

        case EXISTS:
            return primaryExists();

        case SELECT:
            // 子查询: (SELECT ...)
            return parseSelectSubQuery();

        default:
            // ... 其他 Token 类型
    }
}
```

`primary()` 的 switch 覆盖了约 30 种 Token 类型，几乎所有 SQL 中的"基础值"都能在这里找到对应的处理分支。

### 标识符处理：最重要的分支

当遇到 `IDENTIFIER` Token 时，调用 `primaryIdentifier()`（第 939 行，**私有方法**）：

```java
private SQLExpr primaryIdentifier() {
    long hash_lower = lexer.hashLCase();
    String ident = lexer.stringVal();
    lexer.nextToken();

    // 检查是否是特殊关键字（TRY_CAST, DATE, TIMESTAMP 等）
    SQLExpr expr = primaryIdentifierRest(hash_lower, ident);
    if (expr == null) {
        // 无特殊处理 → 普通标识符
        expr = new SQLIdentifierExpr(ident);
    }

    // 处理后缀操作（函数调用、属性访问等）
    expr = primaryRest(expr);
    return expr;
}
```

标识符解析的结果取决于**后续 context**（由 `primaryRest` 处理）：
- `COUNT(` → 左括号 → 函数调用 → `SQLMethodInvokeExpr`/`SQLAggregateExpr`
- `t.id` → 点号 → 属性访问 → `SQLPropertyExpr`
- `name` → 无特殊后缀 → 普通标识符 → `SQLIdentifierExpr`

### 区分聚合函数 vs 普通函数

Druid 将聚合函数名硬编码在哈希集合中：

```java
static {
    AGGREGATE_FUNCTIONS = {"AVG", "COUNT", "MAX", "MIN", "STDDEV", "SUM"};
}
```

函数调用的解析在 `primaryRest()` → `methodRest()` 中完成：遇到标识符后紧跟 `(`，就创建 `SQLMethodInvokeExpr`，再检查函数名是否是聚合函数，如果是则包装为 `SQLAggregateExpr`。

## 第二步：primaryRest() — 后缀操作

**源码位置**: SQLExprParser.java:1590

`primaryRest()` 处理表达式之后的后缀操作：

```java
public SQLExpr primaryRest(SQLExpr expr) {
    if (lexer.token() == Token.DOT) {
        // t.id — 属性访问
        expr = dotRest(expr);
    } else if (lexer.token() == Token.LBRACKET) {
        // arr[0] — 数组下标
        // ...
    } else if (lexer.token() == Token.LPAREN) {
        // func() — 函数调用
        expr = methodRest(expr, true);
    }
    // ... OVER、FILTER、COLLATE 等
    return expr;
}
```

## 第三步：exprRest() — 优先级链

**源码位置**: SQLExprParser.java:174

这是表达式解析的**核心机制**。Druid **没有**使用优先级数字，而是通过**方法链的顺序**来隐式表达优先级：

```java
public SQLExpr exprRest(SQLExpr expr) {
    // 优先级从高到低排列
    expr = bitXorRest(expr);        // ^
    expr = multiplicativeRest(expr); // * / % DIV MOD
    expr = additiveRest(expr);      // + - ||
    expr = shiftRest(expr);         // << >> >>>
    expr = bitAndRest(expr);        // &
    expr = bitOrRest(expr);         // |
    expr = inRest(expr);            // IN / NOT IN
    expr = relationalRest(expr);    // = < > <= >= <> != LIKE BETWEEN IS
//  expr = equalityRest(expr);      // (被注释掉, 合并到 relationalRest)
    expr = andRest(expr);           // AND
    expr = xorRest(expr);           // XOR
    expr = orRest(expr);            // OR

    return expr;
}
```

**规律很简单**：排在前面的方法优先级更高，先被解析。

### *Rest() 方法的内部结构

每个 `*Rest()` 方法的结构都遵循同一模式。以 `multiplicativeRest()` 为例：

```java
public SQLExpr multiplicativeRest(SQLExpr expr) {
    while (true) {
        if (lexer.token() == Token.STAR) {          // *
            lexer.nextToken();
            SQLExpr right = bitXor();               // ★ 用更高优先级方法解析右操作数
            expr = new SQLBinaryOpExpr(expr, SQLBinaryOperator.Multiply, right, dbType);
        } else if (lexer.token() == Token.SLASH) {  // /
            lexer.nextToken();
            SQLExpr right = bitXor();
            expr = new SQLBinaryOpExpr(expr, SQLBinaryOperator.Divide, right, dbType);
        } else if (lexer.token() == Token.PERCENT) { // %
            lexer.nextToken();
            SQLExpr right = bitXor();
            expr = new SQLBinaryOpExpr(expr, SQLBinaryOperator.Modulus, right, dbType);
        } else {
            break;
        }
    }
    return expr;
}
```

关键点：
1. **`while` 循环** — 同一优先级可以连续结合（如 `a * b * c`）
2. **`bitXor()` 解析右操作数** — 右操作数使用更高优先级方法解析，保证 `a + b * c` 的正确结合
3. **每个 `*Rest()` 对应一个同名的 `*()` 驱动方法**（如 `multiplicativeRest` 对应 `multiplicative()`），后者调用前者形成递归

### 完整的优先级层次

| 优先级（高 → 低） | *Rest 方法 | 运算符 |
|---|---|---|
| 1 (最高) | `bitXorRest` | `^`, `->`, `#>` |
| 2 | `multiplicativeRest` | `*`, `/`, `%`, `DIV`, `MOD` |
| 3 | `additiveRest` | `+`, `-`, `\|\|` |
| 4 | `shiftRest` | `<<`, `>>`, `>>>` |
| 5 | `bitAndRest` | `&` |
| 6 | `bitOrRest` | `\|` |
| 7 | `inRest` | `IN`, `NOT IN` |
| 8 | `relationalRest` | `=`, `<`, `>`, `<=`, `>=`, `<>`, `!=`, `LIKE`, `BETWEEN`, `IS` |
| 9 | `andRest` | `AND` |
| 10 | `xorRest` | `XOR` |
| 11 (最低) | `orRest` | `OR` |

### 解析示例：a + b * c > 10

```
expr()
  → primary() → SQLIdentifierExpr("a")
  → primaryRest("a") → 无后缀
  → exprRest("a"):

    bitXorRest("a")      → "a" (没有 ^)

    multiplicativeRest("a") → "a" (没有 * / %)

    additiveRest("a"):
      token=PLUS(+), 匹配!
      right = multiplicative()  ← 用更高优先级方法解析 b * c
        → primary() → "b"
        → multiplicativeRest("b"):
            token=STAR(*), 匹配!
            right = bitXor() → primary() → "c"
            返回 "b * c"
        返回 "b * c"
      返回 "a + b * c"

    inRest("a + b * c") → (没有 IN)

    relationalRest("a + b * c"):
      token=GT(>), 匹配!
      right = bitOr() → primary() → 10
      返回 "a + b * c > 10"

    andRest("a + b * c > 10") → (没有 AND)
    orRest("a + b * c > 10") → (没有 OR)

  返回 "a + b * c > 10"
```

结果：`(a + (b * c)) > 10` — 乘法的优先级高于加法，关系运算的优先级最低。

## 特殊表达式处理

### BETWEEN

BETWEEN 在 `relationalRest()` 中处理：

```java
// relationalRest 内部
if (lexer.token() == Token.BETWEEN) {
    lexer.nextToken();
    SQLExpr begin = bitOr();  // BETWEEN x AND y 中的 x
    accept(Token.AND);
    SQLExpr end = bitOr();    // BETWEEN x AND y 中的 y
    expr = new SQLBetweenExpr(expr, begin, end);
}
```

### IN

`inRest()` 处理 IN 表达式：

```java
public final SQLExpr inRest(SQLExpr expr) {
    if (lexer.token() == Token.IN) {
        lexer.nextToken();
        accept(Token.LPAREN);
        if (lexer.token() == Token.SELECT) {
            SQLSelect select = createSelectParser().select();
            expr = new SQLInSubQueryExpr(expr, select);
        } else {
            List<SQLExpr> list = new ArrayList<>();
            // 解析逗号分隔的表达式列表
            expr = new SQLInListExpr(expr, list);
        }
        accept(Token.RPAREN);
    }
    return expr;
}
```

## 方言扩展：MySqlExprParser

方言子类通过覆盖 `*Rest()` 方法来插入方言特定的运算符：

```java
// MySqlExprParser.java
public class MySqlExprParser extends SQLExprParser {

    public MySqlExprParser(String sql) {
        this(new MySqlLexer(sql));
    }

    // MySQL 特定：覆盖 primaryIdentifierRest 拦截特殊标识符
    protected SQLExpr primaryIdentifierRest(long hash_lower, String ident) {
        if (hash_lower == FnvHash.Constants.CURRENT_USER) {
            return new SQLCurrentUserExpr();  // current_user → 特殊表达式
        }
        return super.primaryIdentifierRest(hash_lower, ident);
    }

    // MySQL 特定：覆盖 additiveRest 处理赋值 :=
    @Override
    public SQLExpr additiveRest(SQLExpr expr) {
        expr = super.additiveRest(expr);
        if (lexer.token() == Token.COLONEQ) {
            // := 赋值操作
            lexer.nextToken();
            SQLExpr right = expr();
            expr = new SQLBinaryOpExpr(expr, SQLBinaryOperator.Assignment, right, dbType);
        }
        return expr;
    }
}
```

## 完整的表达式解析示例

```java
String exprSql = "a + b * c > 10 AND status = 'active'";

SQLExprParser parser = new SQLExprParser(exprSql, DbType.mysql);
SQLExpr expr = parser.expr();

System.out.println("表达式类型: " + expr.getClass().getSimpleName());
// SQLBinaryOpExpr (AND — 最外层运算符)

System.out.println("表达式树: ");
System.out.println(SQLUtils.toSQLString(expr, DbType.mysql));
// a + b * c > 10 AND status = 'active'
```

使用 `SQLUtils` 快捷方法：

```java
SQLExpr expr = SQLUtils.toSQLExpr("a + b * c > 10", DbType.mysql);
```

## 💡 设计思想：方法链 vs 优先级数字

很多表达式解析器使用"优先级数字表"（如 `*` 优先级 40，`+` 优先级 30），但 Druid 选择了**方法链**的方式。

方法链的优势：
1. **自文档化** — 方法名 `multiplicativeRest` 清楚地表明它处理乘法运算
2. **方言扩展友好** — 子类只需要覆盖某个 `*Rest()` 方法就能调整优先级或添加运算符，不需要动全局优先级表
3. **类型安全** — 没有数字常量容易写错的隐患
4. **粒度精细** — 每个优先级层级有独立的方法，可以单独调试

## 🔍 动手探索

1. 在 `SQLExprParser.java:174` 的 `exprRest()` 打断点，观察 `a + b * c` 的解析链
2. 对比 `additiveRest()` 和 `multiplicativeRest()` 的结构异同
3. 查看 `relationalRest()` 中 LIKE、BETWEEN、IS NULL 的处理分支
4. 找到 `primary()` 中 CASE WHEN 的分支（`case CASE`）

## 思考题

1. Druid 为什么选择方法链（chain of `*Rest()`）而不是优先级数字表？这对方言扩展有什么好处？
2. `primary()` 和 `primaryRest()` 的职责如何划分？为什么需要拆成两个方法？
3. 如果要在表达式中添加 ES 的 `match(field, value)` 函数支持，需要改什么？

## 关键源码路径

| 文件 | 方法 | 说明 |
|------|------|------|
| SQLExprParser.java:85 | `expr()` | 入口：解析完整表达式 |
| SQLExprParser.java:174 | `exprRest()` | 优先级链入口 |
| SQLExprParser.java:199 | `bitXorRest()` | 按位异或（最高优先级） |
| SQLExprParser.java:292 | `multiplicativeRest()` | 乘除模 |
| SQLExprParser.java:3410 | `additiveRest()` | 加减 |
| SQLExprParser.java:3742 | `relationalRest()` | 关系运算 |
| SQLExprParser.java:3527 | `andRest()` | AND |
| SQLExprParser.java:3642 | `orRest()` | OR（最低优先级） |
| SQLExprParser.java:464 | `primary()` | 原子表达式 |
| SQLExprParser.java:1590 | `primaryRest()` | 后缀操作 |

## 思考题答案

<details>
<summary>点击展开</summary>

1. **Druid 为什么选择方法链而不是优先级数字表？**
   - **方言扩展友好**：方言子类只需覆盖某个 `*Rest()` 方法就能修改该优先级层级的运算符行为，不影响其他层级。如果使用全局优先级数字表，方言要修改优先级就得改表的数字配置，破坏性更大。
   - **自文档化**：方法名 `multiplicativeRest` 自带语义，阅读 `exprRest()` 的方法调用链就能一目了然所有运算符的优先级顺序。
   - **类型安全**：不会出现优先级数字写错、冲突或越界的问题。
   - **粒度控制**：每个方法可以独立打断点调试，方便排查问题。

2. **`primary()` 和 `primaryRest()` 的职责如何划分？**
   - `primary()` 处理"原子"表达式：字面量、标识符、子查询、CASE WHEN 等。它是递归下降的"最内层"。
   - `primaryRest()` 处理**后缀**操作：`.` 属性访问、`[]` 下标、`()` 函数调用、`OVER` 窗口函数等。这些操作附着在原子表达式之后。
   - 拆分的意义：`primaryRest` 可以被递归调用（如 `a.b.c` 需要两次 `dotRest`），而 `primary()` 不需要处理这种后缀递归。

3. **如果要添加 `match(field, value)` 函数支持，需要改什么？**
   - **最简方案：什么都不用改**。`match(field, value)` 会被 `primary()` 识别为 IDENTIFIER，然后 `primaryRest()` → `methodRest()` 解析为 `SQLMethodInvokeExpr`，函数名 "match"，参数 [field, value]。在你的自定义 Visitor 中检测函数名即可输出 ES 的 match query。
   - **需要特殊语法时**：如果 `match` 有特殊的 SQL 语法（如 `MATCH(field) AGAINST(value)`），才需要在 `primaryIdentifierRest()` 中拦截并创建自定义 AST 节点。
</details>

## 下一课预告

**第 7 课：SQLStatementParser 语句解析** — 表达式解析是"零件"，语句解析就是"组装"。SQLStatementParser 负责将 Token 序列解析为完整的 SQL 语句 AST，包括 SELECT、INSERT、UPDATE、DELETE 等。我们将剖析它的分发机制和解析流程。
