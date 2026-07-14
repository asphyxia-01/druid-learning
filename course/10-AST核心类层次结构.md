---
lesson: 10
title: AST 核心类层次结构
duration: 45 分钟
objectives:
  - 理解 Druid AST 的三层接口体系
  - 掌握 SQLObjectImpl 的 accept/acceptChild 机制
  - 理解 AST 节点的父子关系管理
  - 学会遍历 AST 树
prerequisites: 第 1 课
---

# 第 10 课：AST 核心类层次结构

## AST 是什么？

AST（Abstract Syntax Tree，抽象语法树）是 SQL 的**结构化表示**。

```
"SELECT id, name FROM users WHERE age > 18"
                    ↓
         SQLSelectStatement
              └── SQLSelect
                    └── SQLSelectQueryBlock
                          ├── selectList
                          │   ├── SQLIdentifierExpr("id")
                          │   └── SQLIdentifierExpr("name")
                          ├── from: SQLTableSource("users")
                          └── where: SQLBinaryOpExpr
                                ├── left: SQLIdentifierExpr("age")
                                ├── operator: >
                                └── right: SQLIntegerExpr(18)
```

## 三层核心接口

Druid 的 AST 由三层接口构成：

```
SQLObject (所有 AST 节点的根接口)
    ├── SQLExpr (表达式节点)
    │   └── SQLStatementImpl (语句节点，也实现了 SQLDbTypedObject)
    │
    SQLObjectImpl (抽象基类)
    ├── SQLExprImpl (表达式基类)
    └── SQLStatementImpl (语句基类)
```

### SQLObject — 一切 AST 节点的根

**文件位置**: `druid/core/src/main/java/com/alibaba/druid/sql/ast/SQLObject.java`

```java
public interface SQLObject {
    // ★ 接受 Visitor 访问（核心方法）
    void accept(SQLASTVisitor visitor);

    // 克隆
    SQLObject clone();

    // 父子关系
    SQLObject getParent();
    void setParent(SQLObject parent);

    // 属性系统（附加信息）
    Map<String, Object> getAttributes();
    void putAttribute(String name, Object value);

    // 输出到 StringBuilder
    void output(StringBuilder buf);

    // 源码位置（用于错误报告）
    int getSourceColumn();
    int getSourceLine();

    // 注释
    List<String> getBeforeCommentsDirect();
    List<String> getAfterCommentsDirect();
}
```

### SQLExpr — 表达式

**文件位置**: `druid/core/src/main/java/com/alibaba/druid/sql/ast/SQLExpr.java`

```java
public interface SQLExpr extends SQLObject, Cloneable {
    SQLExpr clone();
    SQLDataType computeDataType();  // 计算表达式类型
    List<SQLObject> getChildren();   // 获取子节点
}
```

`SQLExpr` 代表所有"有值"的节点：
- 字面量: `1`, `'hello'`, `true`
- 列引用: `id`, `t.name`
- 运算: `a + b`, `age > 18`
- 函数调用: `COUNT(*)`, `NOW()`

### SQLStatement — 语句

**文件位置**: `druid/core/src/main/java/com/alibaba/druid/sql/ast/SQLStatement.java`

```java
public interface SQLStatement extends SQLObject, SQLDbTypedObject {
    DbType getDbType();            // 语句所属方言
    void setAfterSemi(boolean afterSemi);
    List<SQLObject> getChildren();
}
```

## SQLObjectImpl — 抽象基类

**文件位置**: `druid/core/src/main/java/com/alibaba/druid/sql/ast/SQLObjectImpl.java`

```java
public abstract class SQLObjectImpl implements SQLObject {
    protected SQLObject parent;             // 父节点
    protected Map<String, Object> attributes;  // 属性映射
    protected SQLCommentHint hint;
    protected int sourceLine;               // 源码行
    protected int sourceColumn;             // 源码列

    // ★ Visitor 访问的核心机制
    public final void accept(SQLASTVisitor visitor) {
        if (visitor == null) {
            throw new IllegalArgumentException();
        }
        visitor.preVisit(this);     // 前处理
        accept0(visitor);           // 子类实现的具体访问逻辑
        visitor.postVisit(this);    // 后处理
    }

    // 子类实现: 访问自己的子节点
    protected abstract void accept0(SQLASTVisitor v);

    // 工具方法: 访问单个子节点
    protected final void acceptChild(SQLASTVisitor visitor, SQLObject child) {
        if (child == null) return;
        child.accept(visitor);
    }

    // 工具方法: 访问子节点列表
    protected final void acceptChild(SQLASTVisitor visitor, List<? extends SQLObject> children) {
        if (children == null) return;
        for (int i = 0; i < children.size(); i++) {
            acceptChild(visitor, children.get(i));
        }
    }

    // output() 默认实现
    public void output(StringBuilder buf) {
        // 创建一个新的 OutputVisitor，将 AST 输出为 SQL
        SQLASTOutputVisitor visitor = new SQLASTOutputVisitor(buf);
        this.accept(visitor);
    }
}
```

### 💡 accept() 的工作机制

`accept(visitor)` 是**访问者模式**的核心：

```
visitor.preVisit(this)
    ↓ 做准备工作
accept0(visitor)
    ↓ 通知 visitor "我到了"，然后让 visitor 继续访问我的孩子
visitor.postVisit(this)
    ↓ 做收尾工作
```

### 具体的 accept0 示例

以 `SQLBinaryOpExpr`（二元操作符表达式）为例：

```java
// SQLBinaryOpExpr.java (位于 ast/expr/ 下)
protected void accept0(SQLASTVisitor visitor) {
    if (visitor == null) return;

    // 1. 先访问左操作数
    if (visitor.visit(this)) {  // ★ 询问 visitor "是否要进入这个节点？"
        acceptChild(visitor, this.left);
        acceptChild(visitor, this.right);
    }

    // 2. 离开时通知
    visitor.endVisit(this);
}
```

## AST 节点的父子关系

Druid 手动维护了每个 AST 节点的父子关系：

```java
// 以 SQLSelectQueryBlock 为例
public class SQLSelectQueryBlock extends SQLObjectImpl {
    private List<SQLSelectItem> selectList;  // 子节点
    private SQLTableSource from;             // 子节点
    private SQLExpr where;                   // 子节点

    // 添加 selectItem 时自动设置 parent
    public void addSelectItem(SQLSelectItem item) {
        if (item != null) {
            item.setParent(this);  // ★ 每个子节点都知道自己的父节点
        }
        this.selectList.add(item);
    }
}
```

## 常见的 AST 节点类型

### 表达式节点（72 种，位于 `ast/expr/`）

| 节点类 | 对应 SQL | 说明 |
|--------|---------|------|
| SQLIdentifierExpr | `id`, `name` | 标识符（列名/表名） |
| SQLPropertyExpr | `t.id` | 限定属性 |
| SQLIntegerExpr | `1`, `100` | 整数 |
| SQLCharExpr | `'hello'` | 字符串 |
| SQLBooleanExpr | `true`, `false` | 布尔值 |
| SQLBinaryOpExpr | `a + b`, `age > 18` | 二元操作 |
| SQLBinaryOpExprGroup | `a AND b AND c` | 二元操作分组（优化） |
| SQLMethodInvokeExpr | `NOW()`, `SUBSTR(x,1)` | 函数调用 |
| SQLAggregateExpr | `COUNT(*)`, `SUM(x)` | 聚合函数 |
| SQLCaseExpr | `CASE WHEN ... END` | CASE 表达式 |
| SQLInListExpr | `x IN (1,2,3)` | IN 列表 |
| SQLInSubQueryExpr | `x IN (SELECT...)` | IN 子查询 |
| SQLExistsExpr | `EXISTS (SELECT...)` | EXISTS |
| SQLCastExpr | `CAST(x AS INT)` | 类型转换 |
| SQLBetweenExpr | `x BETWEEN a AND b` | BETWEEN |

### 语句节点（284 种，位于 `ast/statement/`）

| 节点类 | 对应 SQL | 说明 |
|--------|---------|------|
| SQLSelectStatement | `SELECT ...` | 查询 |
| SQLInsertStatement | `INSERT INTO ...` | 插入 |
| SQLUpdateStatement | `UPDATE ... SET ...` | 更新 |
| SQLDeleteStatement | `DELETE FROM ...` | 删除 |
| SQLCreateTableStatement | `CREATE TABLE ...` | 建表 |
| SQLDropTableStatement | `DROP TABLE ...` | 删表 |
| SQLAlterTableStatement | `ALTER TABLE ...` | 改表 |

## 遍历 AST 的多种方式

### 方式 1：使用 Visitor

```java
// 官方推荐，功能最强
SQLASTVisitor visitor = new SQLASTVisitorAdapter() {
    @Override
    public boolean visit(SQLIdentifierExpr x) {
        System.out.println("找到列引用: " + x.getName());
        return true;  // 继续访问子节点
    }
};
stmt.accept(visitor);
```

### 方式 2：手动遍历

```java
if (stmt instanceof SQLSelectStatement) {
    SQLSelectStatement selectStmt = (SQLSelectStatement) stmt;
    SQLSelect select = selectStmt.getSelect();
    SQLSelectQuery query = select.getQuery();
    if (query instanceof SQLSelectQueryBlock) {
        SQLSelectQueryBlock block = (SQLSelectQueryBlock) query;
        // 手动访问各个子句
        System.out.println("WHERE: " + block.getWhere());
        System.out.println("FROM: " + block.getFrom());
    }
}
```

### 方式 3：利用 getChildren()

```java
List<SQLObject> children = stmt.getChildren();
for (SQLObject child : children) {
    System.out.println(child.getClass().getSimpleName());
    // 递归
    if (child.getChildren() != null) {
        traverse(child);
    }
}
```

## 🔍 动手探索

1. 在 `SQLObjectImpl.java` 中找到 `accept()` 方法的实现，理解 preVisit/postVisit 的作用
2. 找一个具体的 AST 类（如 `SQLBinaryOpExpr.java`），看它的 `accept0()` 实现
3. 查看 `SQLExprImpl` 的源码，看它是如何实现 `computeDataType()` 的

## 思考题

1. 为什么 `accept0()` 的设计是 `protected abstract`？为什么不直接在 `accept()` 中做实现？
2. `getChildren()` 方法在哪些场景下特别有用？它和 Visitor 模式有什么区别和联系？
3. 在你实现 SQL to ES DSL 的过程中，你最需要关注哪些 AST 节点类型？

## 关键源码路径

| 文件 | 说明 |
|------|------|
| SQLObject.java | AST 根接口 |
| SQLObjectImpl.java | 抽象基类，accept/acceptChild |
| SQLExpr.java | 表达式接口 |
| SQLStatement.java | 语句接口 |
| ast/expr/SQLBinaryOpExpr.java | 二元操作符（参考实现） |
| ast/statement/SQLSelectStatement.java | SELECT 语句（参考实现） |

## 下一课预告

**第 11 课：表达式节点体系详解** — 我们将深入 Druid 的 72 种表达式节点，系统学习各类表达式在 AST 中的表示方式，包括字面量、标识符、运算、函数调用、CASE WHEN 等。
