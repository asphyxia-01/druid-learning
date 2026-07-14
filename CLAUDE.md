# CLAUDE.md - Druid SQL 源码分析课程开发指南

## 项目概述
本课程旨在帮助学员深入学习 Alibaba Druid 的 SQL 处理模块源码。
课程聚焦于 `druid/core/src/main/java/com/alibaba/druid/sql/` 路径下的核心代码。
目标是让学员从"会使用 Druid SQL 工具"到"理解 SQL 解析引擎的设计原理"。

## 源码目录结构
```
sql/
├── SQLUtils.java          # 核心工具类，对外统一入口
├── SQLDialect.java         # 方言定义（关键字、引号、内置函数等）
├── PagerUtils.java         # 分页工具
├── parser/                 # 词法分析 + 语法分析
│   ├── Lexer.java          # 词法分析器（核心）
│   ├── Token.java          # Token 枚举
│   ├── Keywords.java       # 关键字管理
│   ├── SymbolTable.java    # 符号表（哈希优化）
│   ├── CharTypes.java      # 字符类型判断工具
│   ├── SQLParser.java      # 解析器基类
│   ├── SQLExprParser.java  # 表达式解析器
│   ├── SQLStatementParser.java  # 语句解析器
│   ├── SQLSelectParser.java     # SELECT 解析器
│   └── ...                 # 其他 DDL/DML 解析器
├── ast/                    # 抽象语法树
│   ├── SQLObject.java      # AST 节点根接口
│   ├── SQLObjectImpl.java  # AST 节点抽象基类
│   ├── SQLExpr.java        # 表达式接口
│   ├── SQLStatement.java   # 语句接口
│   ├── expr/               # 表达式节点（72 个文件）
│   └── statement/          # 语句节点（284 个文件）
├── visitor/                # 访问者模式
│   ├── SQLASTVisitor.java  # 访问者接口（核心）
│   ├── SQLASTVisitorAdapter.java  # 适配器基类
│   ├── SQLASTOutputVisitor.java   # 输出/格式化访问者
│   ├── SchemaStatVisitor.java     # 模式统计访问者
│   └── ...
├── dialect/                # 各数据库方言
│   ├── mysql/              # MySQL（最完整的方言示例）
│   ├── oracle/
│   ├── postgresql/
│   └── ... (共 30 种)
├── builder/                # SQL 编程式构建
├── repository/             # Schema 元数据仓库
└── template/               # SQL 模板
```

## 课件编写规范

### 1. 课件格式
- 课件使用 Markdown 编写，存放在 `course/` 目录下
- 每节课一个文件，命名格式: `NN-主题.md`
- 文件头部包含 metadata 区:

```markdown
---
lesson: 序号
title: 课程标题
duration: 预计学习时间
objectives:
  - 学习目标1
  - 学习目标2
prerequisites: 前置课程序号列表
---
```

### 2. 每节课的标准结构
1. **学习目标** - 本节学完后能做什么
2. **前置知识** - 需要先掌握的内容
3. **正文内容**（包含关键源码引用）
4. **代码示例** - 可运行的示例（基于 Druid 源码）
5. **运行验证** - 如何验证学习成果
6. **思考题** - 加深理解的提问
7. **下一课预告** - 引起兴趣

### 3. 源码引用格式
- 引用源码时标明文件路径和行号
- 例如: `SQLUtils.java:385` 表示 `SQLUtils.format()` 方法
- 重要代码段使用代码块并标注语言 `java`

### 4. 课件质量控制
- **内容正确性**: 每个源码引用必须与仓库代码一致
- **渐进性**: 从使用入手，先讲"怎么用"再讲"为什么这么设计"
- **完整性**: 覆盖 lexer → parser → ast → visitor 全流程
- **可验证性**: 每节课提供可运行的代码示例

### 5. 课程回顾机制
每次开始新课件编写前：
1. 回顾已完成的课件目录结构，确保连续性
2. 检查上节课的思考题是否在后续有呼应
3. 验证源码路径是否仍然有效（避免重构导致的路径变化）
4. 确认课程难度递进合理

### 6. Git 使用规范
- 每完成一节课程内容后创建一个 commit
- commit message 格式: `docs: 第N课 - 课程主题`
- 提交前检查: 课件内容完整、代码示例可读、源码引用准确

### 7. 特殊标记
- ⚠️ 表示需要注意的陷阱或易错点
- 💡 表示设计思想或扩展思考
- 🔍 表示需要学员自己探索的内容
- 📖 表示延伸阅读材料

## 课程路线图

### Phase 1: 基础入门（第 1-3 课）
目标: 建立直观认识，学会使用 Druid SQL 工具
- 环境搭建与 Druid SQL 的 Hello World
- SQLUtils 的核心 API 使用
- 初识 SQL AST - 观察解析结果

### Phase 2: 词法分析（第 4-6 课）
目标: 理解 SQL 字符串如何被拆解为 Token
- Token 体系详解
- Lexer 词法分析器深度解析
- 关键字管理与性能优化

### Phase 3: 语法分析（第 7-11 课）
目标: 掌握 Token 序列到 AST 的转换
- SQLParser 基类设计
- 表达式解析（SQLExprParser）
- 语句解析（SQLStatementParser）
- SELECT 语句解析（SQLSelectParser）
- 方言解析器实现（以 MySQL 为例）

### Phase 4: AST 深入（第 12-15 课）
目标: 全面掌握 Druid AST 节点体系
- AST 核心类层次结构
- 表达式节点体系
- 语句节点体系
- AST 节点的生命周期

### Phase 5: 访问者模式（第 16-20 课）
目标: 掌握 Visitor 模式的精妙应用
- SQLASTVisitor 接口设计哲学
- SQLASTOutputVisitor - 从 AST 回到 SQL
- SchemaStatVisitor - 模式/统计信息收集
- 方言 Visitor 实现
- SQL 方言转换原理

### Phase 6: 高级进阶（第 21-25 课）
目标: 全面掌握 Druid SQL 的高级能力
- SQLBuilder - 编程式 SQL 构建
- SchemaRepository - 元数据管理
- SQL 参数化与模板
- SQL 格式化与分页
- 实战: 扩展一个自定义函数支持

## 学习路径图

```
使用层 (SQLUtils)
    ↓
词法分析 (Lexer → Token)
    ↓
语法分析 (Parser → AST)
    ↓
AST 结构 (SQLObject/SQLExpr/SQLStatement)
    ↓
Visitor 处理 (Output/SchemaStat/Transform)
    ↓
高级应用 (Builder/Repository/Parameterized)
```
