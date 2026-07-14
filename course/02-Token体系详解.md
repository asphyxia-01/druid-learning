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

每个 Token 包含两个属性:
- **name**: 枚举名称本身（如 `SELECT`）
- **upperName**: 构造时传入的字符串（如 `"SELECT"`）

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
QUOTE         -- 引号标识符（`xxx` 或 "xxx"）
HINT          -- 优化器提示
EOF           -- 文件结束
```

### 💡 设计亮点：Token 的值类型

Druid 的 Token 不仅是枚举，还携带了一个 `otherValues` 列表：

```java
// Token.java 内部
public List<String> otherValues = null;
```

某些 Token 可以有别名。例如 `TINYINT` 和 `BYTE`、`INT1` 等是同一个 Token 的不同写法。这个机制让 Druid 能灵活处理不同数据库方言的差异。

### Token 的 hash 优化

```java
// Token.java 内部
public final long nameHashCode64;
```

每个 Token 都预计算了 64 位的 FNV-1a 哈希值。这在 Lexer 中进行关键字匹配时大大提升了效率——比较哈希值比比较字符串快得多。

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
IDNETIFIER : 'id' at pos 7
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
2. 为什么某些 Token（如 TINYINT）有 `otherValues`？这给方言扩展带来了什么便利？
3. 如果我要为 ES DSL 添加一个特殊的 Token（如 `MATCH`），应该怎么做？这会影响到哪些组件？

## 关键源码路径

| 文件 | 内容 | 行号 |
|------|------|------|
| Token.java | Token 枚举定义 | 全篇 |
| Token.java | nameHashCode64 预计算 | ~末尾 |
| Lexer.java | nextToken() 产出 Token | ~146 |

## 下一课预告

**第 3 课：Lexer 词法分析器深度解析** — 我们将深入 Lexer 的内部，看看它是如何将原始的 SQL 字符流转换为 Token 序列的。你会学到 Lexer 的状态管理、字符扫描算法、关键字匹配、哈希优化等核心技术。
