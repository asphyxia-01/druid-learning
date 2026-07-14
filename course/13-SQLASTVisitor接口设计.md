---
lesson: 13
title: SQLASTVisitor 接口设计
duration: 50 分钟
objectives:
  - 深入理解访问者模式在 SQL 处理中的应用
  - 掌握 SQLASTVisitor 的接口设计
  - 理解 visit/endVisit 的回调机制
  - 学会编写自定义 Visitor
prerequisites: 第 10 课
---

# 第 13 课：SQLASTVisitor 接口设计

## 访问者模式（Visitor Pattern）回顾

访问者模式允许你在**不修改现有类**的前提下，为这些类添加新的操作。

```
AST 节点树:              Visitor:
SQLSelectStatement       ┌─────────────────┐
  ├── SQLSelect          │ visit(SQLSelect) │
  │   └── SQLQueryBlock  │ → 处理查询块     │
  │       ├── selectList │ → 处理列列表     │
  │       ├── from       │ → 处理表源       │
  │       └── where      │ → 处理条件       │
  └──                    └─────────────────┘
```

在 Druid 中：
- **AST 节点** 是稳定的（固定的 72+284 种节点）
- **操作** 是可变的（格式化、统计、转换、SQL 检查...）

## SQLASTVisitor 接口

**文件位置**: `druid/core/src/main/java/com/alibaba/druid/sql/visitor/SQLASTVisitor.java`

```java
public interface SQLASTVisitor {
    // ★ 每个 AST 节点类型对应两个方法

    // 进入节点时调用
    default boolean visit(SQLSelectStatement x) { return true; }

    // 离开节点时调用
    default void endVisit(SQLSelectStatement x) {}
}
```

### visit() 方法的返回值

`visit()` 的返回值控制遍历行为：

```java
// 返回值 = true  → 继续遍历子节点（默认）
// 返回值 = false → 跳过子节点遍历（剪枝）

// 示例：找到所有的表引用
@Override
public boolean visit(SQLExprTableSource x) {
    System.out.println("找到表: " + x.getTableName());
    return false;  // 表引用没有需要遍历的子节点，直接返回
}
```

### 完整的方法体系

SQLASTVisitor 定义了数百个方法，覆盖所有 AST 节点类型：

| 分类 | 方法前缀 | 示例 |
|------|---------|------|
| 表达式 | `visit(SQLxxxExpr)` | `visit(SQLBinaryOpExpr)` |
| 语句 | `visit(SQLxxxStatement)` | `visit(SQLSelectStatement)` |
| 表源 | `visit(SQLxxxTableSource)` | `visit(SQLJoinTableSource)` |
| 数据类型 | `visit(SQLxxxDataType)` | `visit(SQLArrayDataType)` |
| 分区 | `visit(SQLxxxPartition)` | `visit(SQLPartitionByHash)` |

## 调用顺序

当 `stmt.accept(visitor)` 被调用时，执行顺序如下：

```
stmt.accept(visitor)
  → visitor.preVisit(stmt)
  → stmt.accept0(visitor)              // ★ 实际调用 visit()/endVisit()
  │   → visitor.visit(stmt)           // 1. visit → true 继续
  │   → stmt.acceptChild(visitor, child1)  // 2. 遍历子节点
  │   │   → child1.accept(visitor)
  │   │       → visitor.visit(child1)
  │   │       → child1.accept0(visitor)    // 子节点的子节点...
  │   │       → visitor.endVisit(child1)
  │   → stmt.acceptChild(visitor, child2)
  │   → visitor.endVisit(stmt)        // 3. endVisit
```

### 具体示例：SQLSelectStatement 的遍历

```java
// SQLSelectStatement.accept0()
protected void accept0(SQLASTVisitor visitor) {
    if (visitor == null) return;

    if (visitor.visit(this)) {          // → visit(SQLSelectStatement)
        acceptChild(visitor, select);   // → 遍历 SQLSelect
    }
    visitor.endVisit(this);              // → endVisit(SQLSelectStatement)
}

// SQLSelect.accept0()
protected void accept0(SQLASTVisitor visitor) {
    if (visitor.visit(this)) {          // → visit(SQLSelect)
        acceptChild(visitor, query);    // → 遍历 SQLSelectQuery
        acceptChild(visitor, withSubQuery);
    }
    visitor.endVisit(this);              // → endVisit(SQLSelect)
}
```

## SQLASTVisitorAdapter — 适配器基类

**文件位置**: `druid/core/src/main/java/com/alibaba/druid/sql/visitor/SQLASTVisitorAdapter.java`

```java
public class SQLASTVisitorAdapter implements SQLASTVisitor {
    protected DbType dbType;
    protected int features;

    // 默认所有 visit() 返回 true（继续遍历）
    // 默认所有 endVisit() 空实现（什么也不做）

    // 其他工具方法
    public boolean isEnabled(VisitorFeature feature) { ... }
    public void config(VisitorFeature feature, boolean state) { ... }
}
```

**永远继承 SQLASTVisitorAdapter**，而不是直接实现 SQLASTVisitor。这样你只需要覆盖你关心的方法。

## 扩展：visitor/functions 包 — 表达式求值的插件体系

在 Visitor 体系下，还有一个小而精的扩展点：**`visitor/functions/`** 包。

### 它解决了什么问题？

当 Druid 需要在 AST 层面**计算表达式的值**时（例如 SQL 防火墙判断参数化结果、SQL 模拟执行），它需要知道 `NOW()` 返回当前时间、`LENGTH('abc')` 返回 3。但几百个 SQL 函数的求值逻辑不能全塞在 Visitor 里，所以设计了一个微型的"函数插件系统"。

### 架构

```
SQLEvalVisitor (接口)
    ↓ 实现
SQLEvalVisitorImpl (遍历 AST，遇到函数调用时查表)
    ↓ 查找
SQLEvalVisitorUtils (全局函数注册表: Map<String, Function>)
    ↓ 调用
visitor/functions/  (各个函数的独立实现)
├── Function.java          ← 函数接口
├── Now.java               ← NOW() → new Date()
├── Length.java            ← LENGTH(s) → s.length()
├── Substring.java         ← SUBSTR(s,1,3)
├── Concat.java            ← CONCAT(a,b)
└── ... (30+ 个函数)
```

### 核心代码

```java
// visitor/functions/Function.java:21 — 就 1 个方法
public interface Function {
    Object eval(SQLEvalVisitor visitor, SQLMethodInvokeExpr x);
}

// SQLEvalVisitorUtils.java:48 — 全局注册表
private static Map<String, Function> functions = new HashMap<>();
static {
    functions.put("now", Now.instance);        // 单例注册
    functions.put("length", Length.instance);
    functions.put("substr", Substring.instance);
    functions.put("concat", Concat.instance);
    // ...
}

// Now.java:23 — 每个函数一个独立类
public class Now implements Function {
    public static final Now instance = new Now();
    public Object eval(SQLEvalVisitor visitor, SQLMethodInvokeExpr x) {
        return new Date();  // NOW() 求值就是返回当前时间
    }
}
```

### 调用链路

```
AST 中有 SQLMethodInvokeExpr("NOW")
  → SQLEvalVisitorImpl.visit(SQLMethodInvokeExpr x)
    → SQLEvalVisitorUtils.visit(visitor, x)
      → functions.get("now")
        → Now.instance.eval(visitor, x)
          → return new Date()
```

### 💡 设计模式

这是**策略模式 + 注册表模式**的组合：
- **策略模式**：每个函数一个类，各自实现 `eval` 方法
- **注册表模式**：`Map<String, Function>` 统一管理，函数名做 key
- **单例模式**：每个 Function 实现类用 `instance` 静态字段

如果你想添加新函数的求值支持，只需加一个类 + 一行注册，**不需要改任何 existing 代码**。

## 编写自定义 Visitor

### 示例 1：查找所有表名和列名

```java
public class TableAndColumnFinder extends SQLASTVisitorAdapter {
    private final List<String> tables = new ArrayList<>();
    private final List<String> columns = new ArrayList<>();

    @Override
    public boolean visit(SQLExprTableSource x) {
        tables.add(x.getTableName());
        return true;
    }

    @Override
    public boolean visit(SQLIdentifierExpr x) {
        // 注意：这里包括了表名中的标识符，需要判断 parent
        columns.add(x.getName());
        return true;
    }

    public void printResult() {
        System.out.println("表: " + tables);
        System.out.println("列: " + columns);
    }
}

// 使用
TableAndColumnFinder finder = new TableAndColumnFinder();
SQLUtils.parseStatements(sql, DbType.mysql).get(0).accept(finder);
finder.printResult();
```

⚠️ `SQLIdentifierExpr` 可能会捕获到表名中的标识符。更好的做法是结合 `SchemaStatVisitor`。

### 示例 2：提取 WHERE 条件

```java
public class WhereConditionExtractor extends SQLASTVisitorAdapter {
    private final List<SQLExpr> conditions = new ArrayList<>();

    @Override
    public boolean visit(SQLBinaryOpExpr x) {
        // 只提取 AND 连接的最外层条件
        if (x.getOperator() == SQLBinaryOperator.BooleanAnd) {
            return true;  // 继续深入
        }

        // 收集非 AND 的条件
        if (isComparison(x.getOperator())) {
            conditions.add(x);
        }
        return true;
    }

    private boolean isComparison(SQLBinaryOperator op) {
        return op == SQLBinaryOperator.Equality
            || op == SQLBinaryOperator.GreaterThan
            || op == SQLBinaryOperator.LessThan
            || op == SQLBinaryOperator.GreaterThanOrEqual
            || op == SQLBinaryOperator.LessThanOrEqual
            || op == SQLBinaryOperator.NotEqual;
    }

    public void printConditions() {
        for (SQLExpr cond : conditions) {
            System.out.println("  " + cond);
        }
    }
}
```

### 示例 3：SQL to ES DSL（简化版）

```java
public class SimpleEsDslVisitor extends SQLASTVisitorAdapter {
    private final StringBuilder json = new StringBuilder();

    @Override
    public boolean visit(SQLSelectStatement x) {
        json.append("{\"query\": ");
        return true;
    }

    @Override
    public boolean visit(SQLBinaryOpExpr x) {
        if (x.getOperator() == SQLBinaryOperator.Equality) {
            json.append("{\"term\": {");
            json.append("\"").append(x.getLeft()).append("\": ");
            json.append(x.getRight());
            json.append("}}");
            return false;  // 不继续遍历子节点
        } else if (x.getOperator() == SQLBinaryOperator.GreaterThan) {
            json.append("{\"range\": {");
            json.append("\"").append(x.getLeft()).append("\": {\"gt\": ");
            json.append(x.getRight());
            json.append("}}}");
            return false;
        }
        // 其他操作符...
        return true;
    }

    @Override
    public void endVisit(SQLSelectStatement x) {
        json.append("}");
    }

    public String getResult() {
        return json.toString();
    }
}
```

## 常用的内置 Visitor

| Visitor 类 | 用途 |
|-----------|------|
| `SQLASTOutputVisitor` | SQL 格式化输出（AST → SQL） |
| `SchemaStatVisitor` | 收集表、列、条件、关系等统计信息 |
| `SQLEvalVisitor` | 表达式求值（如 `1+1` → `2`） |
| `SQLTransformVisitor` | SQL 方言转换（MySQL → Oracle） |
| `ExportParameterVisitor` | 提取 SQL 参数 |
| `SQLTableAliasCollectVisitor` | 收集表别名 |

## 💡 设计思想

Druid 选择访问者模式而非在 AST 节点上直接添加方法的原因：

1. **开放封闭原则**: AST 节点是稳定的，操作是可扩展的
2. **职责分离**: 每个 Visitor 只做一件事（格式化、统计、检查...）
3. **避免类型判断**: 不需要 `instanceof` 链，多态自动分发

⚠️ 但也带来一个问题：如果 AST 节点发生变化，所有 Visitor 都需要更新。Druid 通过 `default` 方法（Java 8+）降低了这个影响。

## 🔍 动手探索

1. 在 `SQLASTVisitor.java` 中搜索 `default boolean visit`，看有多少个 visit 方法
2. 找到 `SQLASTVisitorAdapter.java`，看它默认实现了哪些行为
3. 查看 `SQLEvalVisitor` 的实现，看它如何对表达式求值

## 思考题

1. `visit()` 方法返回 `false` 时会发生什么？什么场景下需要返回 `false`？
2. 访问者模式和 `instanceof` 链相比，有哪些优势和劣势？
3. 如果你要把 SQL 的 WHERE 条件翻译成 Elasticsearch 的 bool 查询，你的 Visitor 需要处理哪些节点类型？哪种遍历顺序是正确的？

## 关键源码路径

| 文件 | 说明 |
|------|------|
| SQLASTVisitor.java | 访问者接口，数百个 visit/endVisit 方法 |
| SQLASTVisitorAdapter.java | 适配器基类，所有 visit 默认返回 true |
| SQLObjectImpl.java:51 | accept() 实现，驱动遍历 |
| SQLEvalVisitor.java | 表达式求值 Visitor（参考实现） |
| ExportParameterVisitor.java | 参数提取 Visitor（参考实现） |

## 思考题答案

<details>
<summary>点击展开</summary>

1. **`visit()` 返回 `false` 时会发生什么？什么场景下需要返回 `false`？**
   - 返回 `false` 时，**当前节点的子节点不会被遍历**（剪枝）。
   - 典型场景：`SQLIdentifierExpr` 没有有意义的子节点，`visit(SQLIdentifierExpr)` 返回 `false` 避免不必要的递归；`SQLExprTableSource` 的表名已经提取完毕，不需要再遍历内部的标识符。剪枝可以提升性能。

2. **访问者模式和 `instanceof` 链相比，优势在哪里？**
   - **开放封闭原则**：新增操作只需要加一个新的 Visitor 类，不需要改任何 AST 节点。`instanceof` 链需要在每个需要新操作的地方加 if-else 分支。
   - **编译期检查**：如果 AST 节点新增了类型，Visitor 接口新增了方法，所有未实现的 Visitor 会被编译器标记（Java 8 的 default 方法缓解了这个问题，但仍然是显式的）。
   - **分派效率**：`node.accept(visitor)` 是**双分派**（double dispatch）——运行时同时确定 node 类型和 visitor 类型。`instanceof` 链是线性的，性能 O(n) vs O(1)。

3. **ES bool 查询的 Visitor 需要处理哪些节点？正确的遍历顺序是什么？**
   - 关键节点：`SQLBinaryOpExpr`（=→term, >→range, AND→bool.must, OR→bool.should, NOT→bool.must_not）、`SQLInListExpr`（IN→terms）、`SQLLikeExpr`（LIKE→wildcard）、`SQLBetweenExpr`（BETWEEN→range）、`SQLMethodInvokeExpr`（match/match_phrase 等）。
   - 遍历顺序：先解析叶子条件（term/range），然后逐层组合 AND/OR/NOT 为 bool 查询。`SQLBinaryOpExpr(BooleanAnd)` 应该返回 `true` 继续遍历子节点，让子节点的 visit 分别生成各自的查询片段，然后在父节点组合。
</details>

## 下一课预告

**第 14 课：SQLASTOutputVisitor 深入** — 输出访问者是 Druid 中最重要的 Visitor 之一。它负责将 AST 重新转换为格式化的 SQL 字符串。我们将深入它的实现，理解 SQL 格式化的工作原理。
