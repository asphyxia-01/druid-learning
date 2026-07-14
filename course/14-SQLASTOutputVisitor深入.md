---
lesson: 14
title: SQLASTOutputVisitor 深入
duration: 50 分钟
objectives:
  - 理解 SQLASTOutputVisitor 的工作机制
  - 掌握 SQL 格式化的实现原理
  - 学会自定义输出格式
  - 理解方言 OutputVisitor 的扩展方式
prerequisites: 第 13 课
---

# 第 14 课：SQLASTOutputVisitor 深入

## OutputVisitor 的角色

SQLASTOutputVisitor 是 Druid 中**最核心的 Visitor**。它的职责很简单：**将 AST 节点重新输出为 SQL 字符串**。

```
AST (结构化对象)
    ↓ SQLASTOutputVisitor
SQL 字符串 (文本)
```

**文件位置**: `druid/core/src/main/java/com/alibaba/druid/sql/visitor/SQLASTOutputVisitor.java`

约 5500+ 行，是 Druid SQL 模块中最长的文件之一。

## 核心设计

```java
public class SQLASTOutputVisitor extends SQLASTVisitorAdapter
        implements ParameterizedVisitor, PrintableVisitor {

    protected final StringBuilder appender;   // SQL 输出目标
    protected int indentCount;                // 缩进级别
    protected boolean ucase = true;           // 是否大写关键字
    protected boolean prettyFormat = true;    // 是否美化（换行缩进）
    protected boolean parameterized = false;  // 是否参数化
}
```

### 核心模式: 每个 visit() 输出对应的 SQL 片段

```java
// SQLIdentifierExpr → 直接输出名称
@Override
public boolean visit(SQLIdentifierExpr x) {
    printName(x.getName());
    return false;  // 标识符没有子节点
}

// SQLIntegerExpr → 输出数字
@Override
public boolean visit(SQLIntegerExpr x) {
    printNumber(x.getNumber().toString());
    return false;
}

// SQLBinaryOpExpr → 输出 "left op right"
@Override
public boolean visit(SQLBinaryOpExpr x) {
    SQLExpr left = x.getLeft();
    SQLExpr right = x.getRight();

    // 输出左操作数
    left.accept(this);

    // 输出操作符
    printBinaryOperator(x.getOperator().name);

    // 输出右操作数
    right.accept(this);

    return false;
}
```

## SELECT 语句的输出流程

以 `SELECT id, name FROM users` 为例，OutputVisitor 的遍历过程：

```
visit(SQLSelectStatement)
  ├── print("SELECT ")
  ├── visit(SQLSelectQueryBlock)
  │   ├── print("SELECT ")      ← 实际在 queryBlock 的 visit 中输出
  │   ├── visit(SQLSelectItem)
  │   │   ├── visit(SQLIdentifierExpr) → print("id")
  │   │   └── endVisit(SQLSelectItem)
  │   ├── print(", ")
  │   ├── visit(SQLSelectItem)
  │   │   ├── visit(SQLIdentifierExpr) → print("name")
  │   │   └── endVisit(SQLSelectItem)
  │   ├── println()
  │   ├── print("FROM ")
  │   ├── visit(SQLExprTableSource) → print("users")
  │   ├── print(";") (可选)
  │   └── endVisit(SQLSelectQueryBlock)
  └── endVisit(SQLSelectStatement)
```

### 格式化输出（prettyFormat）

当 `prettyFormat = true` 时，OutputVisitor 会在合适的位置插入换行和缩进：

```java
// SQLSelectQueryBlock 的 visit 方法（简化）
@Override
public boolean visit(SQLSelectQueryBlock x) {
    // 输出 SELECT
    print0(ucase ? "SELECT" : "select");
    println();  // ← 换行
    incrementIndent();

    // 输出 SELECT 列表
    for (int i = 0; i < selectList.size(); i++) {
        if (i > 0) {
            print(", ");
            println();  // ← 每个列后换行
        }
        selectList.get(i).accept(this);
    }

    // 输出 FROM
    decrementIndent();
    println();
    print0(ucase ? "FROM" : "from");
    println();  // ← FROM 后换行
    incrementIndent();
    x.getFrom().accept(this);

    // ... WHERE, ORDER BY, LIMIT...
}
```

## 参数化输出

当 `parameterized = true` 时，字面量会被替换为 `?`：

```java
@Override
public boolean visit(SQLIntegerExpr x) {
    if (this.parameterized) {
        print('?');           // 参数化：输出 ?
    } else {
        printNumber(x.getNumber().toString());  // 正常：输出数字
    }
    return false;
}

@Override
public boolean visit(SQLCharExpr x) {
    if (this.parameterized) {
        print('?');
    } else {
        printString(x.getText());  // 正常：输出 'xxx'
    }
    return false;
}
```

## 方言 OutputVisitor 的扩展

每种数据库方言有自己的格式化规则。以 `MySqlOutputVisitor` 为例：

```java
// mysql/visitor/MySqlOutputVisitor.java
public class MySqlOutputVisitor extends SQLASTOutputVisitor {

    public MySqlOutputVisitor(Appendable appender) {
        super(appender);
    }

    // MySQL 特有的 LIMIT 格式化
    @Override
    public boolean visit(SQLLimit x) {
        print0(ucase ? "LIMIT" : "limit");
        print(' ');
        x.getOffset().accept(this);
        print(", ");
        x.getRowCount().accept(this);
        return false;
    }

    // MySQL 特有的 INSERT 语法
    @Override
    public boolean visit(MySqlInsertStatement x) {
        // INSERT [IGNORE] [INTO] ...
        print0(ucase ? "INSERT" : "insert");

        if (x.isIgnore()) {
            print(' ');
            print0(ucase ? "IGNORE" : "ignore");
        }
        // ...
    }
}
```

### SQLUtils 中创建 OutputVisitor

```java
// SQLUtils.java:1156
public static SQLASTOutputVisitor createOutputVisitor(StringBuilder out, DbType dbType) {
    switch (dbType) {
        case mysql:
            return new MySqlOutputVisitor(out);
        case oracle:
            return new OracleOutputVisitor(out);
        case postgresql:
            return new PGOutputVisitor(out);
        // ...
        default:
            return new SQLASTOutputVisitor(out);
    }
}
```

## 自定义 OutputVisitor

如果你想让 AST 输出为 ES DSL 格式（而不是 SQL），可以这样：

```java
public class EsDslOutputVisitor extends SQLASTVisitorAdapter {
    private final StringBuilder builder = new StringBuilder();
    private int indent = 0;
    private boolean pretty = true;

    @Override
    public boolean visit(SQLBinaryOpExpr x) {
        // WHERE id = 1 → {"term": {"id": 1}}
        if (x.getOperator() == SQLBinaryOperator.Equality) {
            writeField("term");
            writeObject(() -> {
                writeField(x.getLeft().toString());
                writeValue(x.getRight());
            });
            return false;
        }
        // WHERE age > 18 → {"range": {"age": {"gt": 18}}}
        else if (x.getOperator() == SQLBinaryOperator.GreaterThan) {
            writeField("range");
            writeObject(() -> {
                writeField(x.getLeft().toString());
                writeObject(() -> {
                    writeField("gt");
                    writeValue(x.getRight());
                });
            });
            return false;
        }
        return true;
    }

    private void writeField(String name) {
        builder.append("\"").append(name).append("\": ");
    }

    private void writeValue(SQLExpr expr) {
        if (expr instanceof SQLIntegerExpr) {
            builder.append(((SQLIntegerExpr) expr).getNumber());
        } else if (expr instanceof SQLCharExpr) {
            builder.append("\"").append(((SQLCharExpr) expr).getText()).append("\"");
        }
    }

    private void writeObject(Runnable r) {
        builder.append("{");
        r.run();
        builder.append("}");
    }
}
```

## 底层工具方法

SQLASTOutputVisitor 提供了一系列 `print*` 方法：

```java
// 输出字符串
protected void print0(String text) { appender.append(text); }
protected void print(char ch) { appender.append(ch); }

// 输出换行
protected void println() {
    if (prettyFormat) {
        appender.append('\n');
        for (int i = 0; i < indentCount; i++) {
            appender.append('\t');
        }
    } else {
        appender.append(' ');
    }
}

// 带引号的名称（取决于方言）
protected void printName(String name) {
    if (quoteFlag && name.indexOf('.') == -1) {
        print0(quoteChars[0] + name + quoteChars[1]);
    } else {
        print0(name);
    }
}
```

## 🔍 动手探索

1. 在 `SQLASTOutputVisitor.java` 中找到 `visit(SQLSelectQueryBlock)` 方法，看完整的格式化逻辑
2. 找到 `visit(SQLBinaryOpExpr)` 方法，看它如何处理括号和优先级
3. 对比 `MySqlOutputVisitor` 和 `OracleOutputVisitor` 在 LIMIT/ROWNUM 处理上的差异

## 思考题

1. 为什么 `visit(SQLIdentifierExpr)` 要返回 `false`？如果返回 `true` 会发生什么？
2. `prettyFormat` 和 `parameterized` 这两种模式是如何共存的？可以同时开启吗？
3. 如果你想用 Druid 实现 SQL 到 ES DSL 的转换，你是选择继承 `SQLASTOutputVisitor` 还是自己写一个全新的 Visitor？为什么？

## 关键源码路径

| 文件 | 方法 | 说明 |
|------|------|------|
| SQLASTOutputVisitor.java | `visit(SQLSelectQueryBlock)` | SELECT 格式化 |
| SQLASTOutputVisitor.java | `visit(SQLBinaryOpExpr)` | 二元操作符输出 |
| SQLASTOutputVisitor.java | `visit(SQLIdentifierExpr)` | 标识符输出 |
| MySqlOutputVisitor.java | `visit(SQLLimit)` | MySQL LIMIT 格式化 |
| MySqlOutputVisitor.java | `visit(MySqlInsertStatement)` | MySQL INSERT 格式化 |

## 下一课预告

**第 15 课：SchemaStatVisitor 与 SQL 分析** — SchemaStatVisitor 是 Druid 中另一个非常重要的 Visitor，它可以自动遍历 AST 并收集 SQL 语句涉及的表、列、条件、关联关系等信息。这对 SQL 审核、防火墙、ES DSL 转换等场景都非常有用。
