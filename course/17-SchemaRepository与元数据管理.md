---
lesson: 17
title: SchemaRepository 与元数据管理
duration: 40 分钟
objectives:
  - 理解 SchemaRepository 的设计目的
  - 掌握 Schema 元数据的管理方法
  - 理解类型推断的实现原理
  - 学会使用 SchemaRepository 辅助 SQL 分析
prerequisites: 第 15 课
---

# 第 17 课：SchemaRepository 与元数据管理

## SchemaRepository 是什么？

**文件位置**: `druid/core/src/main/java/com/alibaba/druid/sql/repository/SchemaRepository.java`

SchemaRepository 是 Druid 的"元数据大脑"。它管理了数据库的 Schema 信息（表结构、列类型、函数定义等），能够辅助 SQL 解析过程中的**语义分析**。

```java
public class SchemaRepository {
    private Schema defaultSchema;
    protected DbType dbType;
    protected SQLASTVisitor consoleVisitor;
}
```

简单来说：如果你告诉 SchemaRepository "users 表有 id(int)、name(varchar) 两列"，那么它就能在解析 `SELECT name FROM users WHERE id = 1` 时知道 `id` 是 int 类型、`name` 是 varchar 类型。

## 为什么要用 SchemaRepository？

没有 SchemaRepository：

```java
// 只能做语法分析，无法做语义分析
SQLStatement stmt = SQLUtils.parseSingleStatement("SELECT UPPER(name) FROM users", DbType.mysql);
// → 知道有 UPPER 函数和 name 列，但不知道 name 的数据类型
```

有 SchemaRepository：

```java
// 能做语义分析，知道列的类型
SchemaRepository repo = new SchemaRepository(DbType.mysql);
repo.console("CREATE TABLE users (id INT, name VARCHAR(100), age INT)");

SQLStatement stmt = repo.console("SELECT UPPER(name), age + 1 FROM users WHERE id = 1");
// → 知道 name 是 VARCHAR，UPPER 合法
// → 知道 age 是 INT，age + 1 合法
```

## 核心功能

### 1. 管理 Schema

```java
SchemaRepository repo = new SchemaRepository(DbType.mysql);

// 方式 1：通过 DDL 注册表结构
repo.console("CREATE TABLE users (id BIGINT AUTO_INCREMENT PRIMARY KEY, name VARCHAR(100), age INT, status VARCHAR(20))");
repo.console("CREATE TABLE orders (id BIGINT, user_id BIGINT, total DECIMAL(10,2), created_at DATETIME)");

// 方式 2：直接访问 Schema 对象
Schema schema = repo.getDefaultSchema();
schema.findTable("users");       // 找到 users 表
schema.findTableOrNull("users"); // 安全的查找方式

// 查看表结构
SQLCreateTableStatement tableStmt = schema.findTable("users").getStatement();
// → SQLCreateTableStatement 中包含了所有列定义
```

### 2. 解析 SQL 并关联 Schema

```java
// 解析 SQL 时自动关联表信息
List<SQLStatement> stmts = repo.console("SELECT * FROM users WHERE age > 18");
// → 等价于 parseStatements 但关联了 Schema 信息

// 再解析一个关联表 JOIN 的 SQL
List<SQLStatement> stmts2 = repo.console(
    "SELECT u.name, o.total FROM users u JOIN orders o ON u.id = o.user_id"
);
```

### 3. 列的类型解析

当 SchemaRepository 结合 `SchemaResolveVisitor` 使用时，可以自动解析列的数据类型：

```java
// SchemaRepository 中集成了 SchemaResolveVisitor
repo.setResolveVisitor(new MySqlSchemaResolveVisitor(repo));
repo.console("SELECT id, UPPER(name) FROM users");

// 此时 id 和 name 的数据类型已经通过 Schema 解析出来
```

## SchemaResolveVisitor

**文件位置**: `druid/core/src/main/java/com/alibaba/druid/sql/repository/SchemaResolveVisitor.java`

这个 Visitor 的任务是**将 Schema 信息"注入"到 AST 节点中**：

```java
public class SchemaResolveVisitor extends SQLASTVisitorAdapter {
    // 遇到标识符时，尝试解析其数据类型
    @Override
    public boolean visit(SQLIdentifierExpr x) {
        // 从 Schema 中查找对应的列定义
        SchemaObject table = repository.findTable(x.getParent());
        if (table != null) {
            Column column = table.findColumn(x.getName());
            if (column != null) {
                // 将数据类型信息注入表达式节点
                x.setResolvedColumn(column);
                x.setResolvedTableSource(table);
            }
        }
        return true;
    }
}
```

## 数据类型的计算

`SQLExpr.computeDataType()` 是 Druid 提供的类型推断机制：

```java
SQLExpr expr = SQLUtils.toSQLExpr("age + 1", DbType.mysql);
SQLDataType dataType = expr.computeDataType();
// → age 是 INT, 1 是 INTEGER
// → age + 1 的结果类型是 BIGINT

// 在 SchemaRepository 的辅助下，类型推断会更准确
repo.console("SELECT age + 1 FROM users");
// 因为知道 age 是 INT，所以 age + 1 的结果类型也能确定
```

## Schema 访问和操作

```java
// 获取所有表
Schema schema = repo.getDefaultSchema();
Collection<SchemaObject> tables = schema.getObjects(SchemaObjectType.Table);

for (SchemaObject table : tables) {
    System.out.println("表: " + table.getName());

    // 获取 CREATE TABLE 语句（包含了所有列定义）
    SQLCreateTableStatement createStmt = table.getStatement();
    if (createStmt != null) {
        for (SQLTableElement element : createStmt.getTableElementList()) {
            if (element instanceof SQLColumnDefinition) {
                SQLColumnDefinition col = (SQLColumnDefinition) element;
                System.out.println("  列: " + col.getName() + " " + col.getDataType());
            }
        }
    }
}
```

## Spring 集成

在 Spring 中使用 Druid 时，可以通过 `StatFilter` 和 `WallFilter` 间接使用 SchemaRepository 的功能。但更常见的是在 SQL 分析和审核工具中直接使用：

```java
// SQL 审核示例
public class SqlAuditor {
    private final SchemaRepository repository;

    public SqlAuditor(DbType dbType) {
        this.repository = new SchemaRepository(dbType);
        // 注册业务表结构
        repository.console("CREATE TABLE users (id INT, name VARCHAR(100), age INT)");
        repository.console("CREATE TABLE orders (id INT, user_id INT, amount DECIMAL)");
    }

    public AuditResult audit(String sql) {
        try {
            List<SQLStatement> stmts = repository.console(sql);
            // 用 SchemaStatVisitor 分析
            SchemaStatVisitor visitor = new SchemaStatVisitor(repository);
            stmts.get(0).accept(visitor);

            // 检查是否涉及敏感列
            for (TableStat.Column column : visitor.getColumns()) {
                if (column.getName().equals("password")) {
                    return AuditResult.deny("禁止访问密码列");
                }
            }

            return AuditResult.approve(visitor);
        } catch (ParserException e) {
            return AuditResult.deny("SQL 语法错误: " + e.getMessage());
        }
    }
}
```

## 完整的 Schema 体系

```
SchemaRepository
  └── Schema (defaultSchema)
       ├── SchemaObject (Table)
       │   ├── name
       │   ├── type (SchemaObjectType)
       │   └── statement (SQLCreateTableStatement)
       │       └── SQLColumnDefinition[]
       │           ├── name
       │           ├── dataType (SQLDataType)
       │           └── constraints
       ├── SchemaObject (View)
       ├── SchemaObject (Sequence)
       └── ...
```

## 💡 对你的启发

在 SQL-to-ES DSL 场景中，SchemaRepository 的作用：

1. **类型映射**：知道 SQL 列的类型，可以更准确地映射到 ES 字段类型
2. **表名校验**：验证 SQL 中的表名是否对应 ES 索引
3. **函数处理**：知道列的类型后，可以更智能地处理函数调用

```java
public class EsSchemaManager {
    private final SchemaRepository repository;

    public EsSchemaManager() {
        this.repository = new SchemaRepository(DbType.mysql);
    }

    // 注册 ES 索引映射为 Schema
    public void registerIndex(String indexName, Map<String, String> fieldTypeMap) {
        StringBuilder ddl = new StringBuilder();
        ddl.append("CREATE TABLE ").append(indexName).append(" (");
        fieldTypeMap.forEach((field, type) -> {
            ddl.append(field).append(" ").append(type).append(", ");
        });
        ddl.setLength(ddl.length() - 2);
        ddl.append(")");
        repository.console(ddl.toString());
    }

    // 映射时利用类型信息
    public String mapToEsType(String sqlColumnType) {
        switch (sqlColumnType.toUpperCase()) {
            case "INT": case "BIGINT": return "long";
            case "VARCHAR": case "TEXT": return "keyword";
            case "DECIMAL": case "DOUBLE": return "double";
            case "DATETIME": case "DATE": return "date";
            case "BOOLEAN": return "boolean";
            default: return "text";
        }
    }
}
```

## 思考题

1. SchemaRepository 中 `console()` 方法返回的是 `List<SQLStatement>`，它和 `SQLUtils.parseStatements()` 返回的结果有什么不同？
2. `SchemaResolveVisitor` 在解析列类型时，如何处理别名（如 `u.id` 中的 `u`）？
3. 如果你需要在 SQL-to-ES DSL 中支持 ES 的 `nested` 类型（对应 SQL 的 JOIN），你需要如何扩展 SchemaRepository？

## 关键源码路径

| 文件 | 说明 |
|------|------|
| SchemaRepository.java | 元数据仓库，管理 Schema |
| repository/Schema.java | Schema 定义，管理表/视图等对象 |
| repository/SchemaObject.java | Schema 对象（表/视图/序列） |
| repository/SchemaObjectType.java | Schema 对象类型枚举 |
| repository/SchemaResolveVisitor.java | 列类型解析 Visitor |
| repository/SchemaResolveVisitorFactory.java | 创建方言对应 SchemaResolveVisitor |

## 思考题答案

<details>
<summary>点击展开</summary>

1. **`console()` 和 `SQLUtils.parseStatements()` 的返回结果有什么不同？**
   - `console()` 返回的 `SQLStatement` 经过了 **SchemaResolveVisitor** 的处理。AST 中的 `SQLIdentifierExpr` 节点被注入了 resolvedColumn 和 resolvedTableSource 信息。所以 `console()` 返回的 AST 不仅知道"有个标识符叫 id"，还知道"id 是 users 表的 INT 类型列"。
   - 而 `SQLUtils.parseStatements()` 只做语法解析，不做语义解析。

2. **SchemaResolveVisitor 如何通过别名解析列类型？**
   - 维护一个 `aliasMap`：遇到 `FROM users u` 时记录 `u → users`；遇到 `JOIN orders o ON ...` 时记录 `o → orders`。
   - 遍历到 `u.id` 时：找 `u` 的 alias 映射 → `users` 表 → 在 Schema 中找 `users` 表的 `id` 列 → 获取其 SQLDataType → 注入到 `SQLPropertyExpr` 的 resolvedColumn。

3. **要支持 ES 的 nested 类型（对应 SQL JOIN），SchemaRepository 需要怎么扩展？**
   - 在注册表结构时，允许标记某些列为 nested 类型（如 `"address" NESTED`）。
   - 扩展 `SchemaResolveVisitor`，当遇到 `JOIN` 时，检查关联的表/列是否标记为 nested，如果是则注入 nested 路径信息。
   - 在你的 ES DSL Visitor 中，检测到 nested 类型时，输出 ES 的 nested query 格式 `{"nested": {"path": "address", "query": {...}}}`。
</details>

## 下一课预告

**第 18 课：PagerUtils 与 SQL 分页** — PagerUtils 是 Druid 提供的一个实用工具，可以自动为 SQL 添加分页逻辑。它的实现原理是在 AST 层面操作 SQL，这也是 Druid SQL 引擎能力的重要体现。
