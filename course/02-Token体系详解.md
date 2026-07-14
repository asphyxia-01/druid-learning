---
lesson: 2
title: Token 体系详解
duration: 40 分钟
objectives:
  - 理解 Token 枚举的定义和作用
  - 掌握 Token 的类型分类体系
  - 理解 Token 在解析流程中的角色
prerequisites: 第 1 课
---

# 第 2 课：Token 体系详解

## Token 是什么？

Token 是词法分析的输出产物。Lexer 将原始 SQL 字符串拆解为一个个 Token，Parser 再根据 Token 序列来构建 AST。

可以把 Token 理解为"SQL 语言的单词表"——任何一种 SQL 语法元素都能在这个表中找到对应的 Token。

## Token 枚举

**文件位置**: `druid/core/src/main/java/com/alibaba/druid/sql/parser/Token.java`

```java
// Token.java:23
public enum Token {
    SELECT("SELECT"),
    DELETE("DELETE"),
    INSERT("INSERT"),
    UPDATE("UPDATE"),
    FROM("FROM"),
    WHERE("WHERE"),
    // ... 共约 200+ 种 Token
}
```

每个 Token 只有一个属性:
- **name**: 构造时传入的字符串（如 `SELECT("SELECT")` 中的 `"SELECT"`），有些 Token 没有这个参数（如 `EOF`, `IDENTIFIER`），此时 `name = null`

看源码就能确认：
```java
// Token.java:409-417
public final String name;    // ← 唯一的属性

Token() {
    this(null);              // EOF、IDENTIFIER 等没有 name
}

Token(String name) {
    this.name = name;
}
```

### Token 的类型分类

我们可以把 Token 分为以下几大类：

#### 1. SQL 关键字 (Keywords)
```java
SELECT, DELETE, INSERT, UPDATE, FROM, WHERE, AND, OR, NOT, IN,
CREATE, ALTER, DROP, TABLE, INDEX, VIEW, JOIN, LEFT, RIGHT,
ORDER, BY, GROUP, HAVING, LIMIT, UNION, AS, ON, DISTINCT, ...
```

#### 2. 操作符 (Operators)
```java
// 比较运算
EQ   (=), GT(>), LT(<), GTEQ(>=), LTEQ(<=),
NEQ(<>), BANGEQ(!=), BANGGT(!>), BANGLT(!<)

// 算术运算
PLUS(+), SUB(-), STAR(*), DIV(/), MOD(%), CONCAT(||)

// 逻辑运算
AND, OR, XOR, NOT
```

#### 3. 分隔符 (Delimiters)
```java
LPAREN(() , RPAREN()), COMMA(,), DOT(.), SEMI(;),
COLON(:), LBRACKET([), RBRACKET(]), LBRACE({), RBRACE(})
```

#### 4. 字面量 (Literals)
```java
LITERAL_INT, LITERAL_FLOAT, LITERAL_CHARS, LITERAL_ALIAS,
LITERAL_HEX, LITERAL_NCHARS  -- N'xxx'
```

#### 5. 特殊标识符
```java
IDENTIFIER    -- 普通标识符（表名、列名等）
VARIANT       -- 变量（@xxx）
HINT          -- 优化器提示
EOF           -- 文件结束
ERROR         -- 词法错误
LITERAL_ALIAS -- 别名（如 t 中的别名部分）
```

注意：反引号标识符 `` `name` `` 和双引号标识符 `"name"` 没有独立的 Token。Lexer 在处理时直接解析为 `IDENTIFIER`，去掉引号后取值。

### Token 的结构很简单

Token.java 除了枚举定义，就是一个 `name` 字段和两个构造器，**没有其他属性**。哈希优化不在 Token 上，而是在 Keywords 和 Lexer 层面（后面第 4 课会讲）。

## Token 如何被使用

Token 在 Lexer 和 Parser 之间扮演着"信使"的角色：

### 1. Lexer 产出 Token

```java
// Lexer 将 SQL 字符串转换为 Token 序列
Lexer lexer = new Lexer("SELECT * FROM t", null, DbType.mysql);
lexer.nextToken();  // → Token.SELECT
lexer.nextToken();  // → Token.STAR
lexer.nextToken();  // → Token.FROM
lexer.nextToken();  // → Token.IDENTIFIER (值为 "t")
lexer.nextToken();  // → Token.EOF
```

### 2. Parser 消费 Token

```java
// SQLParser.java:64
protected void acceptIdentifier(String text) {
    if (lexer.identifierEquals(text)) {
        lexer.nextToken();  // 消费当前 Token，前进到下一个
    } else {
        throw new ParserException("syntax error, expect " + text + ", actual " + lexer.token);
    }
}
```

### 3. Parser 根据 Token 做分支

```java
// SQLStatementParser.java 中解析语句类型
public SQLStatement parseStatement() {
    switch (lexer.token) {
        case SELECT:
            return parseSelect();
        case INSERT:
            return parseInsert();
        case UPDATE:
            return parseUpdate();
        case DELETE:
            return parseDelete();
        case CREATE:
            return parseCreate();
        // ...
    }
}
```

## Token 连接 Lexer 和 Parser 的完整示例

看看一条简单的 SQL 在 Druid 中的 Token 流程：

```java
Lexer lexer = new Lexer("SELECT id, name FROM users", null, DbType.mysql);

// 逐个打印 Token
while (lexer.token() != Token.EOF) {
    System.out.println(
        lexer.token() + " : '" + lexer.stringVal() + "' at pos " + lexer.pos()
    );
    lexer.nextToken();
}
```

输出：
```
SELECT : 'SELECT' at pos 0
IDENTIFIER : 'id' at pos 7
COMMA : ',' at pos 9
IDENTIFIER : 'name' at pos 11
FROM : 'FROM' at pos 16
IDENTIFIER : 'users' at pos 21
EOF : '' at pos 27
```

Parser 消费这些 Token 的逻辑大致是：

```
1. 遇到 SELECT → 开始解析查询
2. 解析 selectList：[id, name]（遇到 COMMA 则继续）
3. 遇到 FROM → 开始解析表
4. 解析 tableSource：users
5. 遇到 EOF → 解析完成
```

## 🔍 探索：查看完整的 Token 列表

```java
// 打印所有 Token
for (Token t : Token.values()) {
    System.out.printf("%-30s name=%s%n", t, t.name);
}
```

## 思考题

1. Token 枚举为什么不直接用字符串，而要定义成枚举类型？枚举比字符串有什么优势？
2. Token 枚举为什么只有 `name` 一个字段？数据类型别名（如 TINYINT 表示多种整数类型）在 Druid 中是如何处理的？
3. 如果我要为 ES DSL 添加一个特殊的 Token（如 `MATCH`），应该怎么做？这会影响到哪些组件？

## 关键源码路径

| 文件 | 内容 | 行号 |
|------|------|------|
| Token.java | Token 枚举定义 | 全篇 |
| Lexer.java | nextToken() 产出 Token | ~146 |

## 思考题答案

<details>
<summary>点击展开</summary>

1. **Token 枚举为什么要定义成枚举类型，而不是直接用字符串？**
   - (a) **类型安全**：枚举值在编译期就确定了，IDE 可以自动补全和检查。用字符串的话 `lexer.token == "SELECt"` 这种拼写错误要到运行时才能发现。
   - (b) **性能**：枚举比较是整数比较（`ordinal`），字符串比较是 O(n)。
   - (c) **switch 支持**：Java 的 `switch` 对枚举有专门的优化（`tableswitch`/`lookupswitch`），Parser 里大量的 `switch(lexer.token)` 就是依赖这个。

2. **数据类型的别名（如 TINYINT/BYTE/INT1）Druid 怎么处理的？**
   - 这些类型名不在 Token.java 中定义。实际上 TINYINT 根本**不是 Token 枚举值**，它们是在 `SQLExprParser.parseDataType()` 方法中通过 FNV 哈希值匹配识别的。`BYTE`、`INT1` 等别名各自有不同的哈希值，但都被映射到同一个 `SQLDataType` 类型对象。这种设计的好处是新增一个数据类型别名不需要改 Token.java。

3. **如果我要为 ES DSL 添加一个 `MATCH` Token，需要怎么做？**
   - 如果只是要识别 `MATCH` 关键字：不需要改 Token.java。在 Keywords 中注册 `"MATCH" → Token.IDENTIFIER` 的映射（实际上默认的 Identifier 处理就够了，`MATCH` 目前可能不在 DEFAULT_KEYWORDS 中，会被识别为普通 IDENTIFIER）。只有当 `MATCH` 需要作为一个**特殊的语法结构**（不同于普通函数调用）时才需要新 Token。
</details>

## 下一课预告

**第 3 课：Lexer 词法分析器深度解析** — 我们将深入 Lexer 的内部，看看它是如何将原始的 SQL 字符流转换为 Token 序列的。你会学到 Lexer 的状态管理、字符扫描算法、关键字匹配、哈希优化等核心技术。
