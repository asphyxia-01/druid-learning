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

### 3. 关键字匹配（accept）

```java
// SQLParser.java:913
// 接受一个关键字：如果当前是期望的 token 就消费它，否则报错
public void accept(Token token) {
    if (!lexer.nextIf(token)) {     // ★ nextIf 检查并消费
        setErrorEndPos(lexer.pos());
        printError(token);
    }
}
```

关键点是 `lexer.nextIf(token)`：这个方法检查当前 Token 是否匹配，如果匹配就**自动前进到下一个 Token**，返回 true。这个"检查+消费"二合一的模式在 Parser 中大量使用，代码更简洁。

```java
// 对应的 match 方法：只检查不消费
public void match(Token token) {
    if (lexer.token != token) {
        throw new ParserException("syntax error, expect " + token);
    }
}
```

`accept` 和 `match` 的区别：
- `accept(Token)` — 检查 + 消费（"我要吃掉这个 Token"）
- `match(Token)` — 只检查不消费（"我只想看看是不是这个"）

### 4. 错误处理

```java
// 记录最远的错误位置（辅助调试，给出更准确的错误信息）
protected void setErrorEndPos(int errPos) {
    if (errPos > errorEndPos) {
        errorEndPos = errPos;
    }
}

// 打印详尽的错误信息（包含上下文）
protected void printError(Token token) {
    // 截取错误位置附近的文本片段
    // 拼装错误信息，抛出 ParserException
}
```

### 5. 特性配置

```java
// 判断是否启用某个 feature
public final boolean isEnabled(SQLParserFeature feature) {
    return lexer.isEnabled(feature);
}

public void config(SQLParserFeature feature, boolean state) {
    this.lexer.config(feature, state);
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

以解析 `SELECT * FROM users` 为例，展示从创建 Parser 到产出 AST 的工作流程：

```
Step 1: new SQLStatementParser("SELECT * FROM users", DbType.mysql)
  → new Lexer(sql, null, dbType)     创建 Lexer，pos=0
  → lexer.nextToken()                 预读 → token=SELECT

Step 2: parser.parseStatementList()
  → lexer.token == SELECT
  → 调用 parseSelect()

Step 3: parseSelect()
  → createSelectParser()             创建 SQLSelectParser
  → selectParser.select()            解析 SELECT 各子句

Step 4: 返回 AST
  → SQLSelectStatement
    → SQLSelect
      → SQLSelectQueryBlock
        → selectList: [SQLAllColumnExpr]
        → from: SQLExprTableSource(users)
```

## 💡 设计思想

SQLParser 的设计体现了一个**分层抽象**的思路：

1. **SQLParser** — 提供解析的基础能力（Lexer 访问、Token 判等、错误处理），约 970 行
2. **SQLExprParser** — 处理表达式解析（`a + b > 1`, `func(x)`），约 6500 行
3. **SQLStatementParser** — 处理语句解析（SELECT/INSERT/UPDATE/DELETE）
4. **方言子类** — 处理方言特定的语法差异

每一层只关注自己职责范围内的事情，上层通过 `super` 调用和委托来复用下层能力。

注意 SQLParser 本身**不包含**表达式解析方法，表达式解析是通过 `SQLExprParser` 独立完成的，`SQLStatementParser` 和 `SQLSelectParser` 内部持有 `exprParser` 引用来委托表达式解析。

## 🔍 动手探索

1. 在 `SQLParser.java` 中找到 `accept(Token)` 和 `match(Token)`，理解它们的区别
2. 查看 `lexer.nextIf(token)` 的实现，理解"检查+消费"二合一的设计
3. 找到 `SQLParserFeature` 中所有枚举值，理解每个特性的作用

## 思考题

1. `accept(Token)` 和 `match(Token)` 有什么本质区别？在什么场景下用哪一个？
2. SQLParser 为什么不自己包含表达式解析方法，而是委托给 SQLExprParser？
3. 位标记（bit flag）相比 boolean 字段有什么优势？在 Parser 的实现中这种设计带来了什么好处？

## 关键源码路径

| 文件 | 方法 | 说明 |
|------|------|------|
| SQLParser.java:31 | 构造函数 | 创建 Lexer、预读 Token |
| SQLParser.java:64 | `identifierEquals()` | 标识符匹配 |
| SQLParser.java:913 | `accept(Token)` | 消费 Token |
| SQLParser.java:932 | `match(Token)` | 检查 Token |
| SQLParserFeature.java | 全篇 | 特性开关定义 |

## 思考题答案

<details>
<summary>点击展开</summary>

1. **`accept(Token)` 和 `match(Token)` 本质区别？**
   - `accept(Token)`：检查当前 Token 是否匹配，如果匹配就**消费掉**（lexer 前进到下一个）。用在"我期望这里出现某个关键字，吃掉它继续解析"的场景。
   - `match(Token)`：只检查当前 Token 是否匹配，**不消费**。用在"我只需要确认当前是什么"的场景。
   - `accept` 内部调用了 `lexer.nextIf(token)` 实现检查+消费二合一。

2. **SQLParser 为什么不自己包含表达式解析方法？**
   - 因为表达式解析是一个独立的关注点。SQLExprParser 约 6500 行，如果放在 SQLParser 中会使其臃肿不堪。更重要的是，不同方言需要覆盖表达式解析行为（如 MySQL 的 `@变量`），放在独立类中可以通过继承实现多态。

3. **位标记相比 boolean 字段的优势？**
   - (a) **节省内存**：32 个特性只需 1 个 int（4 字节），32 个 boolean 是 32 字节。
   - (b) **原子操作**：可以用 `features |= mask`、`features &= ~mask` 一次性开关多个特性。
   - (c) **传递方便**：一个 int 可以传给方法、存在配置里。
   - (d) **比较高效**：`(features & mask) != 0` 一次位运算就完成了检查。
</details>

## 下一课预告

**第 6 课：SQLExprParser 表达式解析** — 表达式是 SQL 的"细胞"。WHERE 子句中的比较、SELECT 列表中的函数调用、SET 语句中的赋值，全都是表达式。我们将深入 SQLExprParser，看看 Druid 是如何处理这些表达式的。
