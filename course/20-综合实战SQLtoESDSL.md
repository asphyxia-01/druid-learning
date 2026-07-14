---
lesson: 20
title: 综合实战 — SQL to ES DSL 转换器
duration: 60 分钟
objectives:
  - 综合运用 Druid SQL 解析的所有知识
  - 实现一个基本的 SQL-to-ES DSL 转换器
  - 掌握 Visitor 模式的实际应用
  - 为你的实际项目打下基础
prerequisites: 第 1-19 课
---

# 第 20 课：综合实战 — SQL to ES DSL 转换器

## 项目目标

将 SQL 查询转换为 Elasticsearch 的查询 DSL（JSON 格式）。

```
输入 SQL:  SELECT id, name, age FROM users WHERE age > 18 AND status = 'active' ORDER BY age DESC LIMIT 20

输出 ES DSL:
{
  "_source": ["id", "name", "age"],
  "query": {
    "bool": {
      "filter": [
        { "range": { "age": { "gt": 18 } } },
        { "term": { "status": "active" } }
      ]
    }
  },
  "sort": [{ "age": "desc" }],
  "size": 20
}
```

## 架构设计

```
SQL String
    ↓
SQLUtils.parseStatements()  ← 复用 Druid 的解析能力
    ↓
List<SQLStatement>
    ↓
EsDslVisitor (自定义 Visitor)  ← 核心转换逻辑
    ↓
ES DSL JSON String
```

## 第一步：创建 Visitor 骨架

```java
import com.alibaba.druid.DbType;
import com.alibaba.druid.sql.SQLUtils;
import com.alibaba.druid.sql.ast.*;
import com.alibaba.druid.sql.ast.expr.*;
import com.alibaba.druid.sql.ast.statement.*;
import com.alibaba.druid.sql.visitor.SQLASTVisitorAdapter;

import java.util.*;

public class EsDslVisitor extends SQLASTVisitorAdapter {

    // 输出结果
    private final Map<String, Object> root = new LinkedHashMap<>();
    private final Map<String, Object> query = new LinkedHashMap<>();
    private final Map<String, Object> bool = new LinkedHashMap<>();
    private final List<Object> filter = new ArrayList<>();

    // 当前构建上下文
    private final Deque<Object> contextStack = new ArrayDeque<>();

    public String toEsDsl() {
        if (!filter.isEmpty()) {
            bool.put("filter", filter);
            query.put("bool", bool);
            root.put("query", query);
        }
        return toJsonString(root);
    }

    @Override
    public boolean visit(SQLSelectStatement stmt) {
        // 重置状态
        filter.clear();
        bool.clear();
        query.clear();
        root.clear();
        return true;
    }

    // ... 后续添加 visit 方法
}
```

## 第二步：处理 SELECT 列表

```java
@Override
public boolean visit(SQLSelectItem item) {
    if (!(item.getExpr() instanceof SQLAllColumnExpr)) {
        // 非 * 的情况，记录字段
        @SuppressWarnings("unchecked")
        List<String> source = (List<String>) root.computeIfAbsent("_source", k -> new ArrayList<>());
        source.add(item.getExpr().toString());
    }
    return true;
}
```

## 第三步：处理 WHERE 条件

```java
@Override
public boolean visit(SQLBinaryOpExpr x) {
    SQLExpr left = x.getLeft();
    SQLExpr right = x.getRight();
    SQLBinaryOperator op = x.getOperator();

    switch (op) {
        case Equality:
            addFilter("term", left.toString(), toEsValue(right));
            return false;  // 不继续遍历子节点

        case GreaterThan:
            addFilter("range", left.toString(), Map.of("gt", toEsValue(right)));
            return false;

        case GreaterThanOrEqual:
            addFilter("range", left.toString(), Map.of("gte", toEsValue(right)));
            return false;

        case LessThan:
            addFilter("range", left.toString(), Map.of("lt", toEsValue(right)));
            return false;

        case LessThanOrEqual:
            addFilter("range", left.toString(), Map.of("lte", toEsValue(right)));
            return false;

        case NotEqual:
            addFilter("bool", null, Map.of("must_not",
                Map.of("term", Map.of(left.toString(), toEsValue(right)))));
            return false;

        case BooleanAnd:
            return true;  // AND 需要继续遍历左右子节点

        case Like: {
            String pattern = right.toString();
            // SQL: 'abc%' → ES: abc*
            String esPattern = pattern
                .replace("'", "")
                .replace("%", "*")
                .replace("_", "?");
            addFilter("wildcard", left.toString(), esPattern);
            return false;
        }

        default:
            return true;
    }
}

@SuppressWarnings("unchecked")
private void addFilter(String type, String field, Object value) {
    Map<String, Object> clause = new LinkedHashMap<>();
    if ("range".equals(type) || "term".equals(type)) {
        clause.put(type, Map.of(field, value));
    } else {
        clause.put(type, value);
    }
    filter.add(clause);
}

private Object toEsValue(SQLExpr expr) {
    if (expr instanceof SQLIntegerExpr) {
        return ((SQLIntegerExpr) expr).getNumber();
    } else if (expr instanceof SQLCharExpr) {
        return ((SQLCharExpr) expr).getText();
    } else if (expr instanceof SQLBooleanExpr) {
        return ((SQLBooleanExpr) expr).getValue();
    } else if (expr instanceof SQLNullExpr) {
        return null;
    }
    return expr.toString();
}
```

## 第四步：处理 IN 条件

```java
@Override
public boolean visit(SQLInListExpr x) {
    List<Object> values = new ArrayList<>();
    for (SQLExpr item : x.getTargetList()) {
        values.add(toEsValue(item));
    }

    Map<String, Object> terms = new LinkedHashMap<>();
    terms.put(x.getExpr().toString(), values);
    filter.add(Map.of("terms", terms));
    return false;
}
```

## 第五步：处理 LIKE

Druid 中 LIKE 是用 `SQLBinaryOpExpr` + `SQLBinaryOperator.Like` 表示的（没有单独的 `SQLLikeExpr` 类）。已在第三步的 `visit(SQLBinaryOpExpr)` switch 中添加了 `Like` 分支，将 SQL 的 `%` 和 `_` 通配符映射为 ES wildcard 查询的 `*` 和 `?`。

## 第六步：处理 ORDER BY

```java
@Override
public boolean visit(SQLSelectQueryBlock queryBlock) {
    // 处理 ORDER BY
    SQLOrderBy orderBy = queryBlock.getOrderBy();
    if (orderBy != null) {
        List<Map<String, String>> sortList = new ArrayList<>();
        for (SQLSelectOrderByItem item : orderBy.getItems()) {
            Map<String, String> sortItem = new LinkedHashMap<>();
            String direction = item.getType() != null
                ? item.getType().toLowerCase()
                : "asc";
            sortItem.put(item.getExpr().toString(), direction);
            sortList.add(sortItem);
        }
        root.put("sort", sortList);
    }

    // 处理 LIMIT
    SQLLimit limit = queryBlock.getLimit();
    if (limit != null) {
        root.put("size", ((SQLIntegerExpr) limit.getRowCount()).getNumber());
    }

    return true;
}
```

## 第七步：处理 FROM（确定索引名）

```java
@Override
public boolean visit(SQLExprTableSource x) {
    root.put("_index", x.getTableName());
    return true;
}
```

## 第八步：JSON 输出

```java
private String toJsonString(Object obj) {
    if (obj == null) return "null";
    if (obj instanceof Number || obj instanceof Boolean) return obj.toString();
    if (obj instanceof String) return "\"" + escapeJson((String) obj) + "\"";
    if (obj instanceof Map) {
        @SuppressWarnings("unchecked")
        Map<String, Object> map = (Map<String, Object>) obj;
        StringBuilder sb = new StringBuilder("{");
        boolean first = true;
        for (Map.Entry<String, Object> entry : map.entrySet()) {
            if (!first) sb.append(", ");
            first = false;
            sb.append("\"").append(entry.getKey()).append("\": ");
            sb.append(toJsonString(entry.getValue()));
        }
        sb.append("}");
        return sb.toString();
    }
    if (obj instanceof List) {
        List<?> list = (List<?>) obj;
        StringBuilder sb = new StringBuilder("[");
        for (int i = 0; i < list.size(); i++) {
            if (i > 0) sb.append(", ");
            sb.append(toJsonString(list.get(i)));
        }
        sb.append("]");
        return sb.toString();
    }
    return "\"" + escapeJson(obj.toString()) + "\"";
}

private String escapeJson(String s) {
    return s.replace("\\", "\\\\")
            .replace("\"", "\\\"")
            .replace("\n", "\\n")
            .replace("\r", "\\r")
            .replace("\t", "\\t");
}
```

## 完整使用

```java
public class SqlToEsDslDemo {
    public static void main(String[] args) {
        // 测试用例
        String[] testCases = {
            "SELECT * FROM users",
            "SELECT id, name, age FROM users WHERE age > 18",
            "SELECT * FROM users WHERE status = 'active' AND age BETWEEN 18 AND 60",
            "SELECT * FROM users WHERE city IN ('北京', '上海', '广州')",
            "SELECT id, name FROM users WHERE name LIKE '%张%' ORDER BY id DESC LIMIT 20",
            "SELECT * FROM users WHERE age > 18 AND status = 'active' ORDER BY age DESC LIMIT 10",
            "SELECT u.id, u.name, o.total FROM users u JOIN orders o ON u.id = o.user_id WHERE u.age > 18"
        };

        for (String sql : testCases) {
            System.out.println("=== SQL ===");
            System.out.println(sql);
            System.out.println("=== ES DSL ===");
            try {
                String esDsl = convert(sql);
                System.out.println(esDsl);
            } catch (Exception e) {
                System.out.println("转换失败: " + e.getMessage());
            }
            System.out.println();
        }
    }

    public static String convert(String sql) {
        List<SQLStatement> stmts = SQLUtils.parseStatements(sql, DbType.mysql);
        EsDslVisitor visitor = new EsDslVisitor();
        stmts.get(0).accept(visitor);
        return visitor.toEsDsl();
    }
}
```

## 进阶：使用 SchemaStatVisitor 辅助

更健壮的做法是结合 `SchemaStatVisitor` 来分析 SQL：

```java
public class AdvancedEsDslConverter {

    public static EsQueryInfo analyze(String sql) {
        List<SQLStatement> stmts = SQLUtils.parseStatements(sql, DbType.mysql);
        SQLStatement stmt = stmts.get(0);

        // 1. 用 SchemaStatVisitor 收集信息
        SchemaStatVisitor analyzer = new SchemaStatVisitor(DbType.mysql);
        stmt.accept(analyzer);

        // 2. 构建 ES 查询信息
        EsQueryInfo info = new EsQueryInfo();

        // 索引名
        analyzer.getTables().keySet().stream().findFirst()
            .ifPresent(t -> info.index = t.getName());

        // 字段列表（从 SELECT 中提取）
        stmt.accept(new SQLASTVisitorAdapter() {
            @Override
            public boolean visit(SQLSelectItem item) {
                String name = item.getExpr().toString();
                if (!"*".equals(name)) {
                    info.fields.add(name);
                }
                return true;
            }
        });

        // 条件
        for (TableStat.Condition cond : analyzer.getConditions()) {
            info.conditions.add(new EsCondition(
                cond.getColumn().getFullName(),
                cond.getOperator(),
                cond.getValues()
            ));
        }

        return info;
    }

    public static class EsQueryInfo {
        public String index;
        public List<String> fields = new ArrayList<>();
        public List<EsCondition> conditions = new ArrayList<>();
    }

    public static class EsCondition {
        public String field;
        public String operator;
        public List<Object> values;

        public EsCondition(String field, String operator, List<Object> values) {
            this.field = field;
            this.operator = operator;
            this.values = values;
        }
    }
}
```

## 继续完善的方向

这个简单的转换器还有很多可以改进的地方：

1. **嵌套查询支持** — `nested` 和 `has_child` 查询
2. **聚合支持** — GROUP BY → ES 聚合
3. **子查询支持** — WHERE IN (SELECT ...)
4. **JOIN 支持** — 转化为 ES 的 `has_child`/`has_parent`
5. **函数支持** — `COUNT` → `value_count`，`UPPER` → `normalizer`
6. **全文搜索** — `MATCH` → ES `match` 查询
7. **错误处理** — 不支持的 SQL 语法给出清晰提示
8. **测试覆盖** — 全面的单元测试

## 课程总结

恭喜你完成了全部 20 课的学习！让我们回顾一下 Druid SQL 模块的完整知识体系：

```
┌─────────────────────────────────────────────────────┐
│                    SQLUtils                         │
│              统一入口 (format/parse/output)          │
├─────────────────────────────────────────────────────┤
│  Lexer              →         Parser                │
│  (词法分析)                   (语法分析)              │
│  Token/Keywords/     →    SQLParser/SQLExprParser    │
│  SymbolTable              SQLStatementParser         │
│  CharTypes                SQLSelectParser            │
│                                                    │
├─────────────────────────────────────────────────────┤
│                       AST                           │
│              (抽象语法树)                            │
│  SQLObject → SQLExpr/SQLStatement                   │
│  72 种表达式 + 284 种语句节点                       │
├─────────────────────────────────────────────────────┤
│                     Visitor                         │
│             (访问者模式处理 AST)                     │
│  SQLASTOutputVisitor → 格式化 SQL                   │
│  SchemaStatVisitor  → 统计信息                      │
│  SQLEvalVisitor     → 表达式求值                    │
│  ExportParameterVisitor → 参数提取                  │
├─────────────────────────────────────────────────────┤
│                 高级应用                             │
│  SQLBuilder        → 编程式构建                     │
│  SchemaRepository  → 元数据管理                     │
│  PagerUtils        → SQL 分页                       │
│  参数化/模板        → SQL 归一化                     │
├─────────────────────────────────────────────────────┤
│             你的目标: SQL to ES DSL                  │
└─────────────────────────────────────────────────────┘
```

### 你的下一步

1. **动手练习** — 基于本课的示例，完善你自己的 SQL-to-ES DSL 转换器
2. **阅读源码** — 遇到问题时直接阅读 Druid 源码中的对应实现
3. **渐进增强** — 从一个功能点开始（如 WHERE 条件转换），逐步覆盖更多 SQL 语法
4. **测试驱动** — 为每个转换规则编写测试用例

## 推荐阅读

- Druid 源码: `druid/core/src/main/java/com/alibaba/druid/sql/`
- 设计模式: 《设计模式：可复用面向对象软件的基础》— 访问者模式章节
- 编译原理: 《编译器设计》— 词法分析、语法分析章节
- Elasticsearch: Elasticsearch 官方文档 — Query DSL 章节

---

**课程完结，祝你 SQL-to-ES DSL 项目顺利！**

## 思考题答案

<details>
<summary>点击展开</summary>

本课是实战课，没有预设思考题。不过在实现你的 SQL-to-ES DSL 转换器时，有几个关键问题值得思考：

1. **如何选择转换的入口点？**
   - 优先从 WHERE 条件开始实现。因为 ES 查询中最核心的就是 `query` 部分，而 SQL 的 WHERE 子句和 ES 的 bool/term/range 查询有直接的映射关系。SELECT 列表（`_source`）、ORDER BY（`sort`）、LIMIT（`size`）的映射相对简单。

2. **如何处理不支持的 SQL 语法？**
   - 提供清晰的错误提示。在 Visitor 中遇到不支持的节点类型时，抛出 `UnsupportedOperationException("暂不支持子查询: " + x.getClass().getSimpleName())`。这比静默忽略要好得多。

3. **如何验证转换结果的正确性？**
   - (a) 构造 SQL，执行转换，人工检查 ES DSL 是否正确。
   - (b) 把生成的 ES DSL 发给 Elasticsearch 的 `_validate/query` API 验证。
   - (c) 写单元测试：SQL 输入 → 预期 ES DSL JSON 输出。
</details>

---

**课程完结，祝你 SQL-to-ES DSL 项目顺利！**
