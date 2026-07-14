---
lesson: 5
title: SQLParser 基类设计
duration: 40 分钟
objectives:
  - 理解 SQLParser 的职责边界
  - 掌握 SQLParser 封装的核心工具方法
  - 理解 Parser 中的错误处理机制
  - 学会创建和配置一个 Parser 实例
prerequisites: 第 3-4 课
---

# 第 5 课：SQLParser 基类设计

## SQLParser 概述

**文件位置**: `druid/core/src/main/java/com/alibaba/druid/sql/parser/SQLParser.java`

SQLParser 是所有解析器的**基类**。它本身不做具体的 SQL 解析，而是提供了解析器所需的**基础设施**：

```java
// SQLParser.java:27
public class SQLParser {
    protected final Lexer lexer;  // 持有 Lexer 引用
    protected DbType dbType;     // 数据库类型
}
```

整个解析器类层次结构：

```
SQLParser (基类)
├── SQLExprParser (表达式解析器)
│   ├── MySqlExprParser
│   ├── OracleExprParser
│   └── ...
└── SQLStatementParser (语句解析器)
    ├── SQLSelectParser (SELECT 解析器)
    └── MySqlStatementParser, OracleStatementParser, ...
```

## SQLParser 的核心能力

### 1. Lexer 封装

```java
// 获取 Lexer 实例
public final Lexer getLexer() {
    return lexer;
}

// 创建 Parser 时自动初始化 Lexer
public SQLParser(String sql, DbType dbType, SQLParserFeature... features) {
    this(new Lexer(sql, null, dbType), dbType);
    for (SQLParserFeature feature : features) {
        config(feature, true);
    }
    this.lexer.nextToken();  // ★ 预读第一个 Token
}
```

⚠️ 注意构造函数调用了 `lexer.nextToken()` — 这意味着 Parser 创建完成后，Lexer 已经指向了**第一个 Token**。

### 2. 标识符匹配

```java
// SQLParser.java:64
protected boolean identifierEquals(String text) {
    return lexer.identifierEquals(text);
}

protected void acceptIdentifier(String text) {
    if (lexer.identifierEquals(text)) {
        lexer.nextToken();
    } else {
        setErrorEndPos(lexer.pos());
        throw new ParserException(
            "syntax error, expect " + text + ", actual " + lexer.token);
    }
}

// 哈希版本的 acceptIdentifier（性能优化）
protected void acceptIdentifier(Long hash) {
    if (lexer.identifierEquals(hash)) {
        lexer.nextToken();
    } else {
        throw new ParserException("syntax error, expect ...");
    }
}
```

### 3. 关键字匹配

```java
// 接受一个关键字（消费掉）
protected void accept(Token token) {
    if (this.lexer.token() == token) {
        lexer.nextToken();
    } else {
        throw new ParserException("syntax error, expect " + token);
    }
}

// 检查当前 Token 是否匹配
protected boolean token(Token token) {
    return this.lexer.token() == token;
}
```

### 4. 表达式解析委托

```java
// 解析表达式（委托给 exprParser）
protected SQLExpr parseExpr() {
    if (exprParser == null) {
        exprParser = new SQLExprParser(lexer, dbType);
    }
    return exprParser.expr();
}

// 解析表达式（指定优先级）
protected SQLExpr parseExpr(int priorities) {
    return exprParser.expr(priorities);
}
```

### 5. 错误处理

```java
// 设置错误位置（辅助调试）
protected void setErrorEndPos(int pos) {
    // 记录出错位置
}

// 判断是否启用某个 feature
public boolean isEnabled(SQLParserFeature feature) {
    return SQLParserFeature.isEnabled(this.features, feature);
}

public void config(SQLParserFeature feature, boolean state) {
    features = SQLParserFeature.config(features, feature, state);
}
```

## SQLParserFeature：解析特性开关

**文件位置**: `druid/core/src/main/java/com/alibaba/druid/sql/parser/SQLParserFeature.java`

```java
public enum SQLParserFeature {
    KeepComments,               // 保留注释
    EnableSQLBinaryOpExprGroup, // 启用二元操作符分组
    KeepNameQuotes,             // 保留名称引号
    SkipInsertValue_NumberCheck, // 跳过插入值的数字检查
    // ...
}
```

这些特性通过位标记的方式传递给 Lexer 和 Parser，以极小的开销控制解析行为。

## 完整的解析流程

以解析 `SELECT * FROM users` 为例，展示 SQLParser 的工作流程：

```
Step 1: 创建 SQLStatementParser
  new SQLStatementParser("SELECT * FROM users", DbType.mysql)
    → 创建 Lexer，预读第一个 Token → token = SELECT

Step 2: 调用 parseStatementList()
  → 根据 token 类型分发
  → token = SELECT → 进入 parseSelect()

Step 3: 创建 SQLSelectParser
  → 调用 select() 方法
  → 解析 SELECT 子句、FROM 子句...

Step 4: 返回 AST
  → SQLSelectStatement
    → SQLSelect
      → SQLSelectQueryBlock
```

## 💡 设计思想

SQLParser 的设计体现了一个**分层抽象**的思路：

1. **SQLParser** — 提供解析的基础能力（Lexer 访问、Token 判等、错误处理）
2. **SQLExprParser** — 处理表达式解析（`a + b > 1`, `func(x)`）
3. **SQLStatementParser** — 处理语句解析（SELECT/INSERT/UPDATE/DELETE）
4. **方言子类** — 处理方言特定的语法差异

每一层只关注自己职责范围内的事情，上层通过 `super` 调用和委托来复用下层能力。

## 🔍 动手探索

1. 在 `SQLParser.java` 中找到所有标记为 `protected` 的方法，思考它们被哪些子类重写
2. 查看 `isEnabled()` 方法如何使用位运算检查特性开关
3. 找到 `SQLParserFeature` 中所有枚举值，理解每个特性的作用

## 思考题

1. SQLParser 为什么要持有 `exprParser` 的引用？改成每次都创建新的 exprParser 会有什么问题？
2. `accept(Token)` 和 `identifierEquals(String)` 有什么本质区别？什么情况下用哪个？
3. 位标记（bit flag）相比 boolean 字段有什么优势？在 Parser 的实现中这种设计带来了什么好处？

## 关键源码路径

| 文件 | 方法 | 说明 |
|------|------|------|
| SQLParser.java:31 | 构造函数 | 创建 Lexer、预读 Token |
| SQLParser.java:64 | `identifierEquals()` | 标识符匹配 |
| SQLParser.java:85 | `accept(Token)` | 消费 Token |
| SQLParser.java:105 | `parseExpr()` | 委托表达式解析 |
| SQLParserFeature.java | 全篇 | 特性开关定义 |

## 下一课预告

**第 6 课：SQLExprParser 表达式解析** — 表达式是 SQL 的"细胞"。WHERE 子句中的比较、SELECT 列表中的函数调用、SET 语句中的赋值，全都是表达式。我们将深入 SQLExprParser，看看 Druid 是如何处理这些表达式的。
