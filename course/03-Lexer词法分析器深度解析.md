---
lesson: 3
title: Lexer 词法分析器深度解析
duration: 60 分钟
objectives:
  - 理解 Lexer 的核心状态管理
  - 掌握字符扫描算法
  - 理解关键字匹配与哈希优化机制
  - 学会阅读 Lexer 的 nextToken() 方法
prerequisites: 第 2 课
---

# 第 3 课：Lexer 词法分析器深度解析

## Lexer 的角色

**文件位置**: `druid/core/src/main/java/com/alibaba/druid/sql/parser/Lexer.java`

Lexer（词法分析器）是整个 SQL 解析流程的**第一站**。它的任务非常简单但又至关重要：

```
输入: "SELECT * FROM users WHERE id = 1"
输出: [SELECT] [STAR] [FROM] [IDENTIFIER:users] [WHERE] [IDENTIFIER:id] [EQ] [LITERAL_INT:1]
```

## Lexer 的核心字段

```java
// Lexer.java:41
public class Lexer {
    // 核心状态
    protected String text;       // 输入的 SQL 文本
    protected int pos;           // 当前扫描位置
    protected char ch;           // 当前字符
    protected Token token;       // 当前 Token
    protected String stringVal;  // 当前 Token 的字符串值

    // 缓冲区
    protected char[] buf;        // 标记缓冲
    protected int bufPos;        // 缓冲位置

    // 哈希优化
    protected long hashLCase;    // 小写哈希值 (fnv1a_64)
    protected long hash;         // 原始哈希值

    // 关键字表
    protected Keywords keywords; // 关键字 → Token 映射

    // 特性配置
    protected int features;      // SQLParserFeature 位标记
    protected boolean skipComment = true;  // 是否跳过注释
    protected boolean keepComments;        // 是否保留注释
}
```

## Token 的扫描算法

Lexer 的核心方法是 `nextToken()`。它通过识别**当前字符的类型**来决定如何扫描下一个 Token。

```java
// (简化版 — 用于理解流程，非完整源码)
public void nextToken() {
    startPos = pos;
    bufPos = 0;

    // 跳过空白字符
    while (Character.isWhitespace(ch)) {
        ch = text.charAt(++pos);
    }

    switch (ch) {
        case '0': case '1': ... case '9':
            scanNumber();          // 扫描数字字面量
            break;
        case '\'':
            scanString();          // 扫描字符串字面量
            break;
        case '`':
        case '"':
            scanAlias();           // 扫描别名/引号标识符
            break;
        case '/':
            scanComment();         // 扫描注释
            break;
        case '=':
            token = Token.EQ;
            ch = text.charAt(++pos);
            break;
        case '>':
            if (text.charAt(pos+1) == '=') {
                token = Token.GTEQ;
                pos += 2;
            } else {
                token = Token.GT;
                pos++;
            }
            ch = text.charAt(pos);
            break;
        default:
            if (Character.isJavaIdentifierStart(ch)) {
                scanIdentifier();  // 扫描标识符/关键字
            }
            break;
    }
}
```

### 1. 数字扫描 (scanNumber)

```java
// Lexer.java (简化)
protected void scanNumber() {
    bufPos = 0;
    boolean isFloat = false;

    // 扫描整数部分
    while (ch >= '0' && ch <= '9') {
        buf[bufPos++] = ch;
        ch = text.charAt(++pos);
    }

    // 扫描小数部分
    if (ch == '.') {
        isFloat = true;
        buf[bufPos++] = ch;
        ch = text.charAt(++pos);
        while (ch >= '0' && ch <= '9') {
            buf[bufPos++] = ch;
            ch = text.charAt(++pos);
        }
    }

    // 扫描科学计数法
    if (ch == 'e' || ch == 'E') { ... }

    stringVal = new String(buf, 0, bufPos);
    token = isFloat ? Token.LITERAL_FLOAT : Token.LITERAL_INT;
}
```

💡 可以看到数字 `1` 会被识别为 `LITERAL_INT`，`1.5` 被识别为 `LITERAL_FLOAT`，`1e10` 被识别为 `LITERAL_FLOAT`。

### 2. 标识符/关键字扫描 (scanIdentifier)

这是 Lexer 中**最核心**的方法。它需要区分：
- 普通标识符（表名、列名）
- SQL 关键字（SELECT、FROM、WHERE）
- 内置函数名（COUNT、SUM）

```java
// Lexer.java (简化)
protected void scanIdentifier() {
    bufPos = 0;

    // 收集标识符的所有字符
    while (ch == '_' || ch == '$' || Character.isJavaIdentifierPart(ch)) {
        buf[bufPos++] = ch;
        ch = text.charAt(++pos);
    }

    String ident = new String(buf, 0, bufPos);

    // ★ 计算哈希值，快速匹配关键字
    long hash = FnvHash.fnv1a_64_lower(ident);

    // 查关键字表
    Token tok = keywords.getKeyword(hash);
    if (tok != null) {
        token = tok;  // 是关键字
    } else {
        token = Token.IDENTIFIER;  // 是普通标识符
        stringVal = ident;
    }
}
```

### 💡 哈希优化

Druid 使用 **FNV-1a 64 位哈希** 来加速关键字匹配：
- 每个关键字预计算哈希值
- 标识符扫描完成后计算哈希值
- 在 `Keywords` 表中二分查找哈希值
- **时间复杂度**: O(log N) 而不是 O(N) 的字符串比较

```java
// Keywords.java
public class Keywords {
    private final long[] hashArray;  // 排序后的哈希数组
    private final Token[] tokens;    // 对应的 Token

    public Token getKeyword(long hash) {
        int index = Arrays.binarySearch(hashArray, hash);
        if (index >= 0) {
            return tokens[index];
        }
        return null;
    }
}
```

## 完整的扫描流程

以 SQL `SELECT * FROM users` 为例，逐字符展示 Lexer 的执行过程：

```
初始状态: pos=0, ch='S'

第1次 nextToken():
  跳过空白: ch='S'
  不是数字，不是引号，不是特殊字符
  进入 scanIdentifier():
    收集: S, E, L, E, C, T → "SELECT"
    计算 hash → 匹配 → token=Token.SELECT

第2次 nextToken():
  跳过空白: ch='*'
  匹配 '*' → token=Token.STAR, pos++

第3次 nextToken():
  跳过空白: ch='F'
  scanIdentifier() → "FROM" → token=Token.FROM

第4次 nextToken():
  跳过空白: ch='u'
  scanIdentifier() → "users" → 不是关键字 → token=Token.IDENTIFIER, stringVal="users"

第5次 nextToken():
  ch=EOF(26) → token=Token.EOF
```

## 方言 Lexer 的扩展

每种数据库方言都可以扩展 Lexer，添加方言特定的扫描逻辑。

```java
// druid/core/src/main/java/.../mysql/parser/MySqlLexer.java
public class MySqlLexer extends Lexer {
    static final Keywords MYSQL_KEYWORDS;

    static {
        Map<String, Token> map = new HashMap<>();
        map.putAll(Keywords.DEFAULT_KEYWORDS.getKeywords());
        // 增加 MySQL 特有的关键字
        map.put("AUTO_INCREMENT", Token.AUTO_INCREMENT);
        map.put("ENGINE", Token.ENGINE);
        // ...
        MYSQL_KEYWORDS = new Keywords(map);
    }

    @Override
    protected Keywords loadKeywords() {
        return MYSQL_KEYWORDS;
    }

    public MySqlLexer(String input) {
        super(input);
        this.keywords = DEFAULT_MYSQL_KEYWORDS;
    }
}
```

## 🔍 动手探索

1. 在 `Lexer.java` 中找到 `nextToken()` 的完整实现（约 200 行）
2. 找到 `scanNumber()` 方法，观察它对科学计数法的处理
3. 找到 `scanString()` 方法，看它如何处理转义字符
4. 找到 `scanAlias()` 方法，看反引号的处理

## 思考题

1. Druid Lexer 为什么不使用 ANTLR/JavaCC 等工具生成，而是手写？
2. `scanIdentifier()` 中使用了 `fnv1a_64_lower` 哈希，为什么既要原始哈希又要小写哈希？
3. 如果我要为 ES DSL 写一个"ES 字段匹配器"（识别 `field:value` 语法），需要修改 Lexer 吗？还是在 Parser 层处理？

## 关键源码路径

| 文件 | 方法 | 说明 |
|------|------|------|
| Lexer.java | `nextToken()` | 核心方法：识别下一个 Token |
| Lexer.java | `scanIdentifier()` | 标识符/关键字扫描 |
| Lexer.java | `scanNumber()` | 数字字面量扫描 |
| Lexer.java | `scanString()` | 字符串字面量扫描 |
| Lexer.java | `scanAlias()` | 引号标识符扫描 |
| Keywords.java | `getKeyword()` | 哈希匹配 |

## 延伸阅读

- **FNV 哈希**: https://en.wikipedia.org/wiki/Fowler%E2%80%93Noll%E2%80%93Vo_hash_function
- **手写 Lexer vs 生成器**: 手写 Lexer 的优势在于灵活性和性能可控

## 思考题答案

<details>
<summary>点击展开</summary>

1. **Druid 为什么不使用 ANTLR/JavaCC，而是手写 Lexer？**
   - (a) **方言差异极细**：30 种方言，每个方言可能有细微的字符处理差异。手写 Lexer 可以在 `nextToken()` 的 `switch(ch)` 中精确控制每一个字符的处理逻辑。
   - (b) **性能要求高**：Druid 被用在 SQL 防火墙、慢 SQL 采集等场景，需要高吞吐解析。手写递归下降解析器通常比生成的解析器快。
   - (c) **无需文法文件**：ANTLR 的 .g4 文件和代码生成增加了构建步骤。Druid 的定位是一个库，不希望引入额外的构建依赖。
   - (d) **增量解析能力**：Druid 的 Lexer 支持 save point、mark/reset 等操作，这在手写实现中更容易控制。

2. **`scanIdentifier()` 中为什么要同时计算 `hash` 和 `hashLCase`？**
   - `hash`（原始哈希）用于匹配大小写敏感的关键字。
   - `hashLCase`（小写哈希）用于匹配不区分大小写的关键字（SQL 关键字默认不区分大小写，`select` 和 `SELECT` 应该匹配同一个 Token）。
   - 两者配合：`hashLCase` 查关键字表，`hash` 做其他用途（如函数名识别）。

3. **"ES 字段匹配器"（`field:value`）要在哪一层处理？**
   - **在 Parser 层处理，而不是 Lexer 层**。因为 `field:value` 涉及语义关联（知道 field 是字段名，value 是值），这不是词法分析器该管的事。词法分析器只需要把 `:` 识别为 COLON Token，Parser 在看到 `IDENTIFIER COLON ...` 的模式时再特殊处理。
</details>

## 下一课预告

**第 4 课：关键字管理与性能优化** — 我们将深入 Keywords 和 SymbolTable 的内部，了解 Druid 如何通过哈希表和符号表技术来优化词法分析的性能。
