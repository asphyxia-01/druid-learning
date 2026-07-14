---
lesson: 9
title: 方言解析器实现（以 MySQL 为例）
duration: 50 分钟
objectives:
  - 理解 Druid 的方言架构设计
  - 掌握 MySQL 解析器的扩展方式
  - 理解 Lexer → ExprParser → StatementParser 的完整链
  - 学会对比不同方言的解析差异
prerequisites: 第 8 课
---

# 第 9 课：方言解析器实现（以 MySQL 为例）

## 为什么需要方言？

不同数据库有各自的 SQL 方言：

```sql
-- MySQL
SELECT * FROM users LIMIT 10

-- Oracle
SELECT * FROM users WHERE ROWNUM <= 10

-- SQL Server
SELECT TOP 10 * FROM users

-- PostgreSQL
SELECT * FROM users LIMIT 10 OFFSET 0
```

Druid 通过**三层扩展机制**来支持不同方言：
1. **Lexer 层** — 方言特有关键字、引号规则
2. **ExprParser 层** — 方言特有表达式
3. **StatementParser 层** — 方言特有语句

## MySQL 方言的三层结构

```
MySqlLexer              (Lexer 层)
    ↓ 产出 Token
MySqlExprParser         (表达式解析层)
    ↓ 表达式中可能嵌套语句
MySqlStatementParser    (语句解析层)
    ↓ 语句中可能包含表达式
MySqlSelectParser / MySqlSelectIntoParser (专用解析器)
```

## 第一层: MySqlLexer

**文件位置**: `druid/core/src/main/java/.../mysql/parser/MySqlLexer.java`

```java
public class MySqlLexer extends Lexer {
    public static final Keywords DEFAULT_MYSQL_KEYWORDS;

    static {
        Map<String, Token> map = new HashMap<String, Token>();
        // 继承所有默认关键字
        map.putAll(Keywords.DEFAULT_KEYWORDS.getKeywords());
        // MySQL 特有关键字
        map.put("AUTO_INCREMENT", Token.AUTO_INCREMENT);
        map.put("ENGINE", Token.ENGINE);
        map.put("CHARSET", Token.CHARSET);
        map.put("COLLATE", Token.COLLATE);
        map.put("AUTOEXTEND_SIZE", Token.AUTOEXTEND_SIZE);
        map.put("AVG_ROW_LENGTH", Token.AVG_ROW_LENGTH);
        map.put("CHECKSUM", Token.CHECKSUM);
        // ... 更多 MySQL 特有关键字
        DEFAULT_MYSQL_KEYWORDS = new Keywords(map);
    }

    public MySqlLexer(String input) {
        super(input);
        this.keywords = DEFAULT_MYSQL_KEYWORDS;
        // MySQL 特有引号字符: 反引号
        this.quoteChar = '`';
    }
}
```

### 💡 关键理解

`MySqlLexer` 本质上只做了两件事：
1. **注册 MySQL 特有的关键字** — 例如 `ENGINE`、`AUTO_INCREMENT`
2. **配置引号字符** — MySQL 使用反引号 `` ` `` 作为标识符引用符

## 第二层: MySqlExprParser

**文件位置**: `druid/core/src/main/java/.../mysql/parser/MySqlExprParser.java`

```java
public class MySqlExprParser extends SQLExprParser {
    public MySqlExprParser(String sql) {
        this(new MySqlLexer(sql));
    }

    public MySqlExprParser(Lexer lexer) {
        super(lexer);
        this.aggregateFunctions = AGGREGATE_FUNCTIONS;
        // MySQL 的聚合函数
        this.aggregateFunctionHashCodes = AGGREGATE_FUNCTIONS_CODES;
    }

    // MySQL 特有的表达式解析: @变量
    @Override
    protected SQLExpr parseVariant() {
        // MySQL 变量: @var_name, @@global.var_name
        if (lexer.token() == Token.VARIANT) {
            String varName = lexer.stringVal();
            lexer.nextToken();
            return new SQLVariantRefExpr(varName);
        }
        return super.parseVariant();
    }

    // MySQL 特有的赋值操作 :=
    @Override
    public SQLExpr primaryExpr() {
        if (lexer.token() == Token.COLON && lexer.stringVal().charAt(0) == ':') {
            // := 赋值
            lexer.nextToken();
            SQLExpr value = expr();
            return new SQLBinaryOpExpr(left, Token.COLON, value, DbType.mysql);
        }
        return super.primaryExpr();
    }
}
```

## 第三层: MySqlStatementParser

**文件位置**: `druid/core/src/main/java/.../mysql/parser/MySqlStatementParser.java`

这是最复杂的部分，处理 MySQL 特有的语句：

```java
public class MySqlStatementParser extends SQLStatementParser {
    public MySqlStatementParser(String sql) {
        super(new MySqlExprParser(sql));
    }

    @Override
    public SQLStatement parseStatement() {
        // 先检查 MySQL 特有语句
        switch (lexer.token) {
            case SHOW:
                return parseShow();        // SHOW TABLES, SHOW DATABASES
            case DESCRIBE:
                return parseDescribe();     // DESCRIBE table
            case EXPLAIN:
                return parseExplain();      // EXPLAIN SELECT ...
            case CALL:
                return parseCall();         // CALL procedure()
            case REPLACE:
                return parseReplace();      // REPLACE INTO ...
            case TRUNCATE:
                return parseTruncate();     // TRUNCATE TABLE
        }
        // 否则使用通用解析
        return super.parseStatement();
    }

    // MySQL 特有: SHOW 语句
    public SQLStatement parseShow() {
        accept(Token.SHOW);

        if (lexer.token() == Token.TABLES) {
            return parseShowTables();
        } else if (lexer.token() == Token.DATABASES) {
            return parseShowDatabases();
        } else if (lexer.token() == Token.CREATE) {
            return parseShowCreate();
        }
        // ...
    }

    // MySQL 特有: INSERT 语句的扩展
    @Override
    public SQLInsertStatement parseInsert() {
        // MySQL 支持 INSERT IGNORE, INSERT ... ON DUPLICATE KEY UPDATE
        MySqlInsertStatement stmt = new MySqlInsertStatement();

        // 解析 INSERT [IGNORE] [INTO]
        accept(Token.INSERT);
        if (lexer.token() == Token.IGNORE) {
            stmt.setIgnore(true);
            lexer.nextToken();
        }

        // ... 调用 super 的解析逻辑
        // ... 再处理 ON DUPLICATE KEY UPDATE

        return stmt;
    }

    // MySQL 特有: SELECT ... INTO ...
    @Override
    public SQLSelectParser createSelectParser() {
        return new MySqlSelectParser(exprParser, selectListCache);
    }
}
```

## MySqlSelectParser 的扩展

```java
// MySqlSelectParser.java
public class MySqlSelectParser extends SQLSelectParser {
    public MySqlSelectParser(SQLExprParser exprParser, SQLSelectListCache selectListCache) {
        super(exprParser, selectListCache);
    }

    @Override
    public SQLSelectQuery query(SQLSelect select, boolean parent) {
        if (lexer.token() == Token.LPAREN) {
            // MySQL 允许 (SELECT ...) 作为子查询
        }
        return super.query(select, parent);
    }

    @Override
    public SQLSelectQueryBlock queryBlock() {
        SQLSelectQueryBlock queryBlock = super.queryBlock();
        // 处理 MySQL 特有的 SQL_CALC_FOUND_ROWS
        // 处理 MySQL 特有的 SQL_NO_CACHE
        return queryBlock;
    }
}
```

## 完整的方言加载流程

```java
// SQLParserUtils.java 中的工厂方法
public static SQLStatementParser createSQLStatementParser(String sql, DbType dbType, boolean keepComments) {
    switch (dbType) {
        case mysql:
        case mariadb:
            return new MySqlStatementParser(sql);
        case oracle:
            return new OracleStatementParser(sql);
        case postgresql:
            return new PGStatementParser(sql);
        case sqlserver:
            return new SQLServerStatementParser(sql);
        // ... 30+ 方言
        default:
            // 如果 dialect 目录下有 META-INF/druid/parser/{dbType}/dialect.properties
            // 则动态加载对应的解析器类
            return createSQLStatementParserByReflect(sql, dbType);
    }
}
```

### META-INF 驱动的方言

从 Druid 1.2.25+ 开始，方言可以通过 SPI 加载：

```
META-INF/druid/parser/{dbType}/
├── dialect.properties    # 引号设置
├── keywords             # 关键字列表
├── alias_keywords       # 别名字
├── builtin_datatypes    # 内置数据类型
├── builtin_functions    # 内置函数
└── builtin_tables       # 内置表
```

详见 `SQLDialect.java:85` 的 `create()` 方法。

## 💡 对你（SQL to ES DSL）的启发

如果想用 Druid 实现 SQL to ES DSL，你有两种路径：

### 路径 A：复用现有解析器
```java
// 直接解析 MySQL SQL，然后用自定义 Visitor 输出 ES DSL
List<SQLStatement> stmts = SQLUtils.parseStatements(sql, DbType.mysql);
// 用自定义 Visitor 遍历 AST
MyEsDslVisitor visitor = new MyEsDslVisitor();
stmts.forEach(stmt -> stmt.accept(visitor));
String esDsl = visitor.getResult();
```

### 路径 B：注册新的"ES 方言"
```
1. 创建 ESDslLexer（若需要特殊关键字）
2. 创建 ESDslExprParser（若需要特殊表达式）
3. 创建 ESDslStatementParser（若需要特殊语句）
4. 创建 ESDslOutputVisitor（将 AST 输出为 ES DSL JSON）
```

**推荐路径 A**，因为 ES 查询 DSL 本质上是 JSON，和 SQL 的语法差异较大，更适合用 Visitor 做"翻译"。

## 🔍 动手探索

1. 对比 `MySqlLexer` 和 `OracleLexer`，看看它们扩展的关键字有何不同
2. 在 `MySqlStatementParser` 中找到 `parseShow()` 的所有分支
3. 查看 `SQLParserUtils.createSQLStatementParser()` 中所有方言的映射关系

## 思考题

1. MySqlLexer 中的 `quoteChar = '` ` ` 对 Lexer 的扫描逻辑有什么影响？
2. 为什么 `MySqlStatementParser` 选择覆盖 `parseStatement()` 而不是只覆盖个别方法？
3. 如果要添加一个新的数据库方言（如"esdsl"），需要在 `SQLParserUtils` 中做哪些修改？

## 关键源码路径

| 文件 | 说明 |
|------|------|
| mysql/parser/MySqlLexer.java | MySQL 词法分析器 |
| mysql/parser/MySqlExprParser.java | MySQL 表达式解析 |
| mysql/parser/MySqlStatementParser.java | MySQL 语句解析 |
| mysql/parser/MySqlSelectParser.java | MySQL SELECT 解析 |
| mysql/visitor/MySqlOutputVisitor.java | MySQL 输出访问者 |
| SQLParserUtils.java | 方言工厂方法 |

## 下一课预告

**第 10 课：AST 核心类层次结构** — 从 Parser 回到 AST。我们将系统学习 Druid AST 的类层次设计，包括 SQLObject、SQLExpr、SQLStatement 三大接口和它们的实现类体系。
