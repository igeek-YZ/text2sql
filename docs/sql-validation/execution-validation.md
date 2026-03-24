# 执行检测

**层次**：运行级别验证

**核心作用**：通过数据库实际执行或计划分析，验证SQL的可执行性、性能风险和结果正确性。

## 检测内容

| 检测项 | 说明 | 示例 |
|--------|------|------|
| 语法解析 | 数据库驱动级解析 | 执行 PREPARE 语句 |
| 执行计划 | 分析 EXPLAIN 输出 | 检测全表扫描 |
| 超时风险 | 评估查询耗时 | 大表 JOIN |
| 资源消耗 | 预估内存/CPU使用 | 大数据量排序 |
| 结果格式 | 验证返回结果结构 | 列数、类型匹配 |
| 权限检查 | 用户权限验证 | SELECT 权限 |

## 验证流程

```
SQL输入
   │
   ▼
┌──────────────┐
│ 执行计划分析 │  EXPLAIN QUERY PLAN
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ 全表扫描检测 │  查找 TABLE SCAN
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ 索引使用检测 │  查找 INDEX 相关
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ 性能评估    │  评估复杂度
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ 试执行验证  │  LIMIT 0 执行
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  错误报告    │
└──────────────┘
```

---

## 1. ExecutionValidator 接口

**核心作用**：定义SQL执行验证的统一接口，支持执行计划分析和性能评估。

### 1.1 接口方法

| 方法 | 参数 | 返回值 | 说明 |
|------|------|--------|------|
| validate | sql, context | ValidationResult | 执行完整验证流程 |
| analyzePlan | sql | ExecutionPlan | 分析执行计划 |
| estimateCost | sql | CostEstimate | 预估执行成本 |
| dryRun | sql, context | DryRunResult | 试执行验证 |

### 1.2 执行步骤

```
【validate 方法】
1. 调用 analyzePlan 分析执行计划
         ↓
2. 检查计划警告（警告信息）
         ↓
3. 评估总成本是否超过阈值
         ↓
4. 检测全表扫描风险
         ↓
5. 调用 dryRun 执行验证
         ↓
6. 收集错误和警告
         ↓
7. 返回 ValidationResult

【analyzePlan 方法】
1. 构建 EXPLAIN SQL 语句
         ↓
2. 创建数据库 Statement
         ↓
3. 设置查询超时
         ↓
4. 执行 EXPLAIN 查询
         ↓
5. 解析计划步骤
         ↓
6. 计算总成本
         ↓
7. 返回 ExecutionPlan
```

---

## 2. ExecutionContext 结构

**核心作用**：封装执行验证的上下文信息，包括数据库连接和超时配置。

### 字段说明

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| connection | Connection | - | 数据库连接对象 |
| timeout | int | - | 查询超时时间（秒） |
| maxRows | int | 1000 | 最大返回行数 |

### 执行步骤

```
1. 接收数据库连接对象
         ↓
2. 设置查询超时时间
         ↓
3. 配置最大返回行数限制
         ↓
4. 传入 validate 方法使用
```

---

## 3. ExecutionPlan 结构

**核心作用**：表示SQL语句的执行计划，包含步骤详情和成本估算。

### 3.1 字段说明

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| raw | String | - | 原始 EXPLAIN 输出 |
| steps | List<PlanStep> | - | 执行计划步骤列表 |
| totalCost | double | - | 总成本估算值 |
| estimatedRows | int | - | 预估影响行数 |
| warnings | List<String> | - | 警告信息列表 |

### 3.2 PlanStep 结构

**核心作用**：表示执行计划中的单个步骤。

#### 字段说明

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| detail | String | - | 步骤详情描述 |
| cost | double | - | 该步骤的成本值 |
| rows | int | - | 预估行数 |
| operation | String | - | 操作类型标识 |

#### 操作类型枚举

| 操作类型 | 说明 |
|----------|------|
| TABLE_SCAN | 全表扫描 |
| INDEX_SCAN | 索引扫描 |
| INDEX_SEARCH | 索引查找 |
| JOIN | 表连接 |
| SORT | 排序操作 |
| OTHER | 其他操作 |

---

## 4. DryRunResult 结构

**核心作用**：封装试执行验证的结果。

### 字段说明

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| success | boolean | - | 是否成功执行 |
| error | String | - | 错误信息 |
| warnings | List<String> | - | 警告列表 |

### 工厂方法

| 方法 | 参数 | 说明 |
|------|------|------|
| success | - | 创建成功结果 |
| error | message | 创建错误结果 |

---

## 5. CostEstimate 结构

**核心作用**：表示SQL执行的资源成本估算。

### 字段说明

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| complexity | String | - | 复杂度等级（LOW/MEDIUM/HIGH） |
| executionTime | double | - | 预估执行时间（秒） |
| memoryUsage | String | - | 内存使用等级 |
| warnings | List<String> | - | 警告信息 |

### 复杂度等级说明

| 等级 | 成本范围 | 说明 |
|------|----------|------|
| LOW | < 10 | 简单查询，性能优良 |
| MEDIUM | 10-100 | 中等复杂度 |
| HIGH | > 100 | 高复杂度，存在性能风险 |

---

## 6. 数据库方言支持

**核心作用**：支持多种数据库的 EXPLAIN 语法差异。

### 方言映射表

| 数据库 | EXPLAIN 语法 |
|--------|--------------|
| SQLite | `EXPLAIN QUERY PLAN sql` |
| MySQL | `EXPLAIN sql` |
| PostgreSQL | `EXPLAIN (ANALYZE false, COSTS true) sql` |
| SQL Server | `SET SHOWPLAN_TEXT ON; sql` |

### 执行步骤

```
1. 根据方言构建对应的 EXPLAIN SQL
         ↓
2. 执行 EXPLAIN 查询
         ↓
3. 解析返回结果
         ↓
4. 统一转换为 ExecutionPlan 结构
```

---

## 执行计划示例

### SQLite EXPLAIN QUERY PLAN

```
sqlite> EXPLAIN QUERY PLAN SELECT * FROM users WHERE id = 1;
QUERY PLAN
`--SEARCH TABLE users USING INTEGER PRIMARY KEY (rowid=?)

sqlite> EXPLAIN QUERY PLAN SELECT * FROM users JOIN orders ON users.id = orders.user_id;
QUERY PLAN
`--SCAN TABLE users
`--SCAN TABLE orders
`--SEARCH SUBQUERY 1 USING INTEGER PRIMARY KEY (rowid=?)
```

### PostgreSQL EXPLAIN

```
EXPLAIN SELECT * FROM users WHERE id = 1;
                       QUERY PLAN
---------------------------------------------------------
 Index Scan using users_pkey on users  (cost=0.43..4.45 rows=1)
   Index Cond: (id = 1)
```

---

## 错误码

| 错误码 | 说明 | 严重程度 |
|--------|------|----------|
| `EXE001` | 执行超时 | Error |
| `EXE002` | 语法执行失败 | Error |
| `EXE003` | 性能风险 | Warning |
| `EXE004` | 权限不足 | Error |
| `EXE005` | 连接失败 | Error |

---

## 验证示例

```java
Connection conn = DriverManager.getConnection("jdbc:sqlite:database.db");
ExecutionValidator validator = new ExecutionValidatorImpl(conn, 5, "sqlite");

ValidationResult result = validator.validate(
    "SELECT * FROM users u, orders o WHERE u.id = o.user_id",
    context
);
// Warnings: ['Potential full table scan on orders']

ExecutionPlan plan = validator.analyzePlan(
    "SELECT * FROM users WHERE id > 1000"
);
// Plan steps: [SEARCH users USING INTEGER PRIMARY KEY]
// Total cost: 4.45
```

---

## 依赖

```xml
<!-- Maven -->
<dependency>
    <groupId>org.antlr</groupId>
    <artifactId>antlr4-runtime</artifactId>
    <version>4.12.0</version>
</dependency>
<!-- JDBC Driver (根据数据库选择) -->
<dependency>
    <groupId>org.xerial</groupId>
    <artifactId>sqlite-jdbc</artifactId>
    <version>3.42.0.0</version>
</dependency>
```
