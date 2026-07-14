# Druid SQL 模块源码分析课程

## 课程简介

本课程旨在帮助学员深入学习 Alibaba Druid 的 SQL 处理模块源码。从基础 API 使用到核心源码解析，从词法分析到访问者模式，循序渐进地掌握 Druid SQL 引擎的设计原理和实现细节。

**适用人群**:
- 正在做 SQL to ES DSL 转换的开发者
- 想深入学习 SQL 解析引擎原理的 Java 工程师
- Druid 源码贡献者或二次开发者

## 学习方法

1. **理论与实践结合** — 每节课都有可运行的代码示例
2. **源码优先** — 所有内容基于真实源码，标注了文件路径和行号
3. **循序渐进** — 从使用到原理，从单点到全景
4. **联系实际** — 每节课都结合 SQL-to-ES DSL 的场景

## 课程目录

### Phase 1: 基础入门
| # | 课程 | 内容 | 时间 |
|---|------|------|------|
| 0 | [课程概览与环境搭建](00-课程概览与环境搭建.md) | Druid SQL 架构总览、Hello World | 30min |
| 1 | [SQLUtils 核心 API 实战](01-SQLUtils核心API实战.md) | format/parse/toSQLString | 45min |

### Phase 2: 词法分析
| # | 课程 | 内容 | 时间 |
|---|------|------|------|
| 2 | [Token 体系详解](02-Token体系详解.md) | Token 枚举、分类体系 | 40min |
| 3 | [Lexer 词法分析器深度解析](03-Lexer词法分析器深度解析.md) | nextToken、scanIdentifier | 60min |
| 4 | [关键字管理与性能优化](04-关键字管理与性能优化.md) | Keywords、SymbolTable、FNV 哈希 | 35min |

### Phase 3: 语法分析
| # | 课程 | 内容 | 时间 |
|---|------|------|------|
| 5 | [SQLParser 基类设计](05-SQLParser基类设计.md) | 解析器基础设施、错误处理 | 40min |
| 6 | [SQLExprParser 表达式解析](06-SQLExprParser表达式解析.md) | 优先级、函数调用、聚合函数 | 55min |
| 7 | [SQLStatementParser 语句解析](07-SQLStatementParser语句解析.md) | 语句分发、DML/DDL 解析 | 50min |
| 8 | [SQLSelectParser 深度解析](08-SQLSelectParser深度解析.md) | SELECT 各子句解析、UNION | 55min |
| 9 | [方言解析器实现(MySQL)](09-方言解析器实现以MySQL为例.md) | 三层扩展机制、MySQL 特有语法 | 50min |

### Phase 4: AST 深入
| # | 课程 | 内容 | 时间 |
|---|------|------|------|
| 10 | [AST 核心类层次结构](10-AST核心类层次结构.md) | SQLObject/SQLExpr/SQLStatement | 45min |
| 11 | [表达式节点体系详解](11-表达式节点体系详解.md) | 72 种表达式节点的用途 | 50min |
| 12 | [语句节点体系详解](12-语句节点体系详解.md) | 284 种语句节点的结构 | 45min |

### Phase 5: 访问者模式
| # | 课程 | 内容 | 时间 |
|---|------|------|------|
| 13 | [SQLASTVisitor 接口设计](13-SQLASTVisitor接口设计.md) | 访问者模式、visit/endVisit | 50min |
| 14 | [SQLASTOutputVisitor 深入](14-SQLASTOutputVisitor深入.md) | SQL 格式化、参数化输出 | 50min |
| 15 | [SchemaStatVisitor 与 SQL 分析](15-SchemaStatVisitor与SQL分析.md) | 表/列/条件自动收集 | 45min |

### Phase 6: 高级进阶
| # | 课程 | 内容 | 时间 |
|---|------|------|------|
| 16 | [SQLBuilder 与编程式构建](16-SQLBuilder与编程式构建.md) | Builder 模式、动态 SQL 构建 | 35min |
| 17 | [SchemaRepository 与元数据管理](17-SchemaRepository与元数据管理.md) | 表结构管理、类型推断 | 40min |
| 18 | [PagerUtils 与 SQL 分页](18-PagerUtils与SQL分页.md) | AST 层面的 SQL 改写 | 30min |
| 19 | [SQL 参数化与模板](19-SQL参数化与模板.md) | 参数提取、SQL 归一化 | 35min |
| 20 | [综合实战: SQL to ES DSL](20-综合实战SQLtoESDSL.md) | 完整转换器实现、课程总结 | 60min |

## 学习路径图

```
使用层 (SQLUtils) ─────────────────── 第 0-1 课
    ↓
词法分析 (Lexer → Token) ────────── 第 2-4 课
    ↓
语法分析 (Parser → AST) ─────────── 第 5-9 课
    ↓
AST 结构 (SQLObject/Expr/Statement) ─ 第 10-12 课
    ↓
Visitor 处理 (Output/SchemaStat) ─── 第 13-15 课
    ↓
高级应用 (Builder/Repository/...) ── 第 16-19 课
    ↓
实战: SQL to ES DSL ──────────────── 第 20 课
```

## 项目结构

```
course/                          ← 课件目录
├── README.md                    课程索引
├── NN-课程主题.md               20 节课件
└── ...

CLAUDE.md                       开发指南
druid/                           Druid 源码
```
