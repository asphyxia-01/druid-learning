---
lesson: 19
title: SQL 参数化与模板
duration: 35 分钟
objectives:
  - 理解 SQL 参数化的概念和用途
  - 掌握 Druid 的参数化 API
  - 理解参数化 Visitor 的实现
  - 学会提取 SQL 参数
prerequisites: 第 14 课
---

# 第 19 课：SQL 参数化与模板

## 什么是 SQL 参数化？

将 SQL 中的字面量替换为占位符 `?`：

```sql
-- 原始 SQL（带具体值）
SELECT * FROM users WHERE id = 1 AND name = 'Alice'

-- 参数化 SQL（占位符）
SELECT * FROM users WHERE id = ? AND name = ?

-- 参数列表
[1, 'Alice']
```

## 为什么需要参数化？

1. **SQL 监控** — 将 `WHERE id = 1`、`WHERE id = 2` 归并为同一类查询
2. **SQL 防火墙** — 基于参数化后的 SQL 做黑白名单判断
3. **缓存** — 参数化 SQL 作为缓存 key
4. **性能分析** — 聚合统计同类 SQL 的执行时间

## Druid 中的参数化

### 1. 输出时参数化

```java
String sql = "SELECT * FROM users WHERE id = 1 AND name = 'Alice'";

// 使用 FormatOption 控制输出
FormatOption paramOption = new FormatOption();
paramOption.setParameterized(true);

String paramSql = SQLUtils.toSQLString(
    SQLUtils.parseStatements(sql, DbType.mysql),
    DbType.mysql,
    paramOption
);

System.out.println(paramSql);
// SELECT * FROM users WHERE id = ? AND name = ?
```

### 2. 使用 VisitorFeature

```java
String paramSql = SQLUtils.toSQLString(
    SQLUtils.parseStatements(sql, DbType.mysql),
    DbType.mysql,
    null,                                      // 使用默认 FormatOption
    VisitorFeature.OutputParameterized         // 开启参数化
);
```

### 3. 提取参数值

```java
String sql = "SELECT * FROM users WHERE id = 1 AND name = 'Alice'";

// 使用 ExportParameterVisitor 提取参数
List<Object> parameters = new ArrayList<>();
SQLStatementParser parser = SQLParserUtils.createSQLStatementParser(sql, DbType.mysql);
parser.parseStatementList().get(0).accept(new MySqlExportParameterVisitor(parameters));

System.out.println(parameters);
// [1, 'Alice']
```

## 参数化 Visitor 的实现

Druid 使用多个 Visitor 来实现参数化：

```
SQLASTParameterizedVisitor     — 通用参数化
MySqlParameterizedVisitor      — MySQL 参数化
ExportParameterVisitorUtils    — 参数提取工具
```

### SQLASTOutputVisitor 中的参数化逻辑

`SQLASTOutputVisitor` 内部通过 `parameterized` 标志控制输出：

```java
// SQLASTOutputVisitor.java
@Override
public boolean visit(SQLIntegerExpr x) {
    if (this.parameterized) {
        print('?');  // 输出 ?
    } else {
        printNumber(x.getNumber().toString());  // 输出具体值
    }
    return false;
}

@Override
public boolean visit(SQLCharExpr x) {
    if (this.parameterized) {
        print('?');
    } else {
        printString(x.getText());  // 输出 'xxx'
    }
    return false;
}
```

### ExportParameterVisitor 的参数提取

```java
// ExportParameterVisitor.java
public interface ExportParameterVisitor extends SQLASTVisitor {
    List<Object> getParameters();
    void setParameters(List<Object> parameters);
}
```

实现类会遍历 AST 并将所有字面量的值收集到参数列表中：

```java
// ExportParameterVisitorUtils.java
public static void exportParameter(SQLExpr expr, List<Object> params) {
    if (expr instanceof SQLIntegerExpr) {
        params.add(((SQLIntegerExpr) expr).getNumber());
    } else if (expr instanceof SQLCharExpr) {
        params.add(((SQLCharExpr) expr).getText());
    } else if (expr instanceof SQLBooleanExpr) {
        params.add(((SQLBooleanExpr) expr).getValue());
    }
    // ... 更多类型
}
```

## SQL 模板

**文件位置**: `druid/core/src/main/java/com/alibaba/druid/sql/template/SQLSelectQueryTemplate.java`

Druid 还支持 SQL 模板，用于预编译和缓存场景：

```java
public class SQLSelectQueryTemplate extends SQLObjectImpl {
    // 参数化后的 SQL 模板
    // 存储了 SQL 结构和参数占位符的对应关系
}
```

但这个功能在 Druid 中相对次要，主要用于内部优化。

## 实用示例：SQL 归一化

```java
public class SqlNormalizer {

    /**
     * 将 SQL 归一化（参数化 + 格式化）
     */
    public static String normalize(String sql, DbType dbType) {
        List<SQLStatement> stmts = SQLUtils.parseStatements(sql, dbType);

        FormatOption option = new FormatOption();
        option.setParameterized(true);   // 参数化
        option.setUppCase(true);         // 统一大小写
        option.setPrettyFormat(false);   // 不美化（单行输出）

        return SQLUtils.toSQLString(stmts, dbType, option);
    }

    /**
     * 提取参数
     */
    public static List<Object> extractParams(String sql, DbType dbType) {
        List<Object> parameters = new ArrayList<>();

        try {
            SQLStatementParser parser = SQLParserUtils.createSQLStatementParser(sql, dbType);
            SQLStatement stmt = parser.parseStatementList().get(0);

            ExportParameterVisitor visitor = ExportParameterVisitorUtils.createExportParameterVisitor(dbType);
            visitor.setParameters(parameters);
            stmt.accept(visitor);
        } catch (Exception e) {
            // 解析失败时返回空列表
        }

        return parameters;
    }

    // 验证归一化
    public static void main(String[] args) {
        String[] sqls = {
            "SELECT * FROM users WHERE id = 1",
            "SELECT * FROM users WHERE id = 2",
            "SELECT * FROM users WHERE id = 100 AND name = 'Alice'",
            "select * from USERS where ID = 1"
        };

        for (String sql : sqls) {
            String normalized = normalize(sql, DbType.mysql);
            List<Object> params = extractParams(sql, DbType.mysql);
            System.out.println("原始: " + sql);
            System.out.println("归一: " + normalized);
            System.out.println("参数: " + params);
            System.out.println();
        }
    }
}
```

输出：
```
原始: SELECT * FROM users WHERE id = 1
归一: SELECT * FROM users WHERE id = ?
参数: [1]

原始: SELECT * FROM users WHERE id = 2
归一: SELECT * FROM users WHERE id = ?
参数: [2]

原始: SELECT * FROM users WHERE id = 100 AND name = 'Alice'
归一: SELECT * FROM users WHERE id = ? AND name = ?
参数: [100, 'Alice']

原始: select * from USERS where ID = 1
归一: SELECT * FROM users WHERE id = ?
参数: [1]
```

注意第 4 个例子：`USERS` 被标准化为 `users`，`ID` 被标准化为 `id`。这是因为 `FormatOption` 的标准化机制（通过 `SQLUtils.normalize()`）。

## 💡 在 SQL-to-ES DSL 中的应用

SQL 参数化对 SQL-to-ES DSL 的启示：

1. **查询模板化**：可以预定义常见的 SQL 查询模板，参数化后对应 ES 查询模板
2. **参数提取**：从 SQL 提取参数值，直接填入 ES 查询的相应位置

```java
public class EsSqlTemplate {
    public static void main(String[] args) {
        String sql = "SELECT * FROM users WHERE age > 18 AND status = 'active' LIMIT 20";

        // 提取参数
        List<Object> params = SqlNormalizer.extractParams(sql, DbType.mysql);
        System.out.println("参数: " + params);  // [18, 'active', 20]

        // 构建 ES 查询
        String esDsl = "{\n" +
            "  \"query\": {\n" +
            "    \"bool\": {\n" +
            "      \"filter\": [\n" +
            "        { \"range\": { \"age\": { \"gt\": " + params.get(0) + " } } },\n" +
            "        { \"term\": { \"status\": \"" + params.get(1) + "\" } }\n" +
            "      ]\n" +
            "    }\n" +
            "  },\n" +
            "  \"size\": " + params.get(2) + "\n" +
            "}";

        System.out.println(esDsl);
    }
}
```

## 思考题

1. SQL 参数化和 PreparedStatement 的参数占位符有什么区别？
2. `ExportParameterVisitor` 提取参数时，如何处理 `IN (1, 2, 3)` 这种情况？每个值是一个参数还是整体是一个参数？
3. 在 SQL-to-ES DSL 中，参数化能带来什么好处？如果不做参数化会有什么问题？

## 关键源码路径

| 文件 | 说明 |
|------|------|
| SQLASTOutputVisitor.java | `parameterized` 标志控制输出 |
| ExportParameterVisitor.java | 参数提取 Visitor 接口 |
| ExportParameterVisitorUtils.java | 参数提取工具方法 |
| MySqlExportParameterVisitor.java | MySQL 参数提取 |
| SQLASTParameterizedVisitor.java | 通用参数化 Visitor |
| template/SQLSelectQueryTemplate.java | SQL 模板 |

## 思考题答案

<details>
<summary>点击展开</summary>

1. **SQL 参数化和 PreparedStatement 的参数占位符有什么区别？**
   - **Druid 的参数化**：是**输出时**的格式化选项。把字面量统一替换为 `?`，用于 SQL 归并、统计、防火墙。参数化后的 SQL 不可执行。
   - **PreparedStatement 的 `?`**：是 **JDBC 协议**中的参数占位符。由数据库驱动处理，用于预编译和防 SQL 注入。带 `?` 的 SQL 可以执行（配合 setXXX 方法）。
   - 两者只是在"表现形式"上相同（都用 `?`），目的完全不同。

2. **`IN (1, 2, 3)` 在参数化时，每个值是一个参数还是一个整体？**
   - Druid 的 `ExportParameterVisitor` 中，`IN (1, 2, 3)` 的每个值**各自是一个参数**，参数列表为 `[1, 2, 3]`。对应的参数化 SQL 为 `IN (?, ?, ?)`。
   - 但如果用 `SQLASTOutputVisitor` 的 `parameterized` 模式，每个值会被输出为 `?`，即 `IN (?, ?, ?)`。和 ExportParameterVisitor 的行为一致。

3. **SQL-to-ES DSL 中参数化能带来什么好处？**
   - (a) **查询模板化**：预编译 SQL 模板 → 预生成 ES DSL 模板（JSON 骨架），运行时只替换参数值。
   - (b) **缓存命中**：相同的查询结构只构建一次 DSL，减少 JSON 序列化开销。
   - (c) **SQL 审计**：参数化后的 SQL 更容易做权限校验（"用户能否执行这类查询？"而不是"用户能否查询 id=1？"）。
</details>

## 下一课预告

**第 20 课（完结篇）：综合实战 — 用 Druid 实现 SQL to ES DSL 转换器** — 我们将综合运用前面学到的所有知识，从零实现一个简单的 SQL-to-ES DSL 转换器。这既是课程的总结，也是你实际项目的起点。
