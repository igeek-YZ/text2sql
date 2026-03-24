# 语义检测

**层次**：含义级别验证

**核心作用**：验证SQL语句的语义正确性，包括表/列存在性、数据类型兼容性、聚合函数使用规范等。

## 检测内容

| 检测项 | 说明 | 示例 |
|--------|------|------|
| 表存在性 | 引用的表在数据库中存在 | `users` 表存在 |
| 列存在性 | 引用的列在表中存在 | `users.user_id` 列存在 |
| 数据类型 | 操作数类型兼容 | `VARCHAR` 与 `INT` 比较 |
| 聚合规范 | 聚合函数与 GROUP BY 匹配 | 非聚合列需在 GROUP BY 中 |
| 别名冲突 | 别名唯一性检查 | 无重复别名 |
| 空值处理 | NULL 值比较规范 | 使用 IS NULL 而非 = NULL |

## 验证流程

```
AST输入
   │
   ▼
┌──────────────┐
│ 表引用提取   │  从AST提取表名
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ 表存在性验证 │  对比Schema
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ 列引用提取   │  从AST提取列名
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ 列存在性验证 │  对比表结构
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ 类型推断    │  推断表达式类型
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ 类型兼容性   │  检查操作兼容
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ 聚合规范检查 │  验证GROUP BY
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  错误报告    │
└──────────────┘
```

---

## 1. SemanticValidator 接口

**核心作用**：定义SQL语义验证的统一接口，支持引用验证、类型检查和聚合规范检查。

### 1.1 接口方法

| 方法 | 参数 | 返回值 | 说明 |
|------|------|--------|------|
| validate | sql, context | ValidationResult | 执行完整语义验证 |
| validateReferences | ast, schema | ValidationResult | 验证表列引用 |
| validateTypes | ast, schema | ValidationResult | 验证数据类型兼容性 |
| validateAggregations | ast | ValidationResult | 验证聚合函数规范 |

### 1.2 执行步骤

```
【validate 方法】
1. 解析SQL生成AST
         ↓
2. 验证表引用存在性
         ↓
3. 验证列引用存在性
         ↓
4. 推断表达式类型
         ↓
5. 验证类型兼容性
         ↓
6. 检查聚合函数规范
         ↓
7. 检查NULL比较规范
         ↓
8. 返回验证结果

【validateReferences 方法】
1. 从AST提取所有表引用
         ↓
2. 检查每个表是否在Schema中存在
         ↓
3. 从AST提取所有列引用
         ↓
4. 检查列是否在对应表中存在
         ↓
5. 返回引用错误列表
```

---

## 2. DatabaseSchema 结构

**核心作用**：表示数据库的完整结构信息，包含所有表和视图定义。

### 2.1 字段说明

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| name | String | - | 数据库名称 |
| tables | List<Table> | - | 表定义列表 |
| views | List<View> | - | 视图定义列表 |

### 2.2 Table 结构

**核心作用**：表示数据库表的结构定义。

#### 字段说明

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| name | String | - | 表名 |
| columns | List<Column> | - | 列定义列表 |
| indexes | List<Index> | - | 索引定义列表 |
| primaryKey | List<String> | - | 主键列列表 |

### 2.3 Column 结构

**核心作用**：表示数据表列的元信息。

#### 字段说明

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| name | String | - | 列名 |
| type | DataType | - | 数据类型 |
| nullable | boolean | true | 是否可空 |
| primaryKey | boolean | false | 是否主键 |
| foreignKey | ForeignKey | - | 外键引用 |
| defaultValue | String | - | 默认值 |
| comment | String | - | 列注释 |

### 2.4 ForeignKey 结构

**核心作用**：表示外键约束定义。

#### 字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| column | String | 外键列名 |
| referencedTable | String | 引用的表名 |
| referencedColumn | String | 引用的列名 |

### 2.5 Index 结构

**核心作用**：表示索引定义。

#### 字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| name | String | 索引名 |
| columns | List<String> | 索引列列表 |
| unique | boolean | 是否唯一索引 |
| type | String | 索引类型（B-TREE, HASH等） |

---

## 3. DataType 枚举

**核心作用**：定义SQL数据类型分类。

### 枚举值

| 枚举值 | 说明 | 示例 |
|--------|------|------|
| INTEGER | 整数类型 | 1, 2, 3 |
| BIGINT | 长整数类型 | 123456789 |
| VARCHAR | 可变字符串 | 'text' |
| TEXT | 长文本类型 | 文章内容 |
| BOOLEAN | 布尔类型 | TRUE, FALSE |
| DATE | 日期类型 | '2024-01-01' |
| DATETIME | 日期时间类型 | '2024-01-01 12:00:00' |
| TIMESTAMP | 时间戳类型 | 1704067200 |
| DECIMAL | 精确小数 | 123.45 |
| FLOAT | 浮点数 | 3.14 |
| DOUBLE | 双精度浮点 | 3.1415926 |
| JSON | JSON数据类型 | '{"key": "value"}' |
| BLOB | 二进制大对象 | 二进制数据 |

### 数据类型分类

| 分类 | 类型 | 说明 |
|------|------|------|
| 数值型 | INTEGER, BIGINT, FLOAT, DOUBLE, DECIMAL | 支持算术运算 |
| 字符串型 | VARCHAR, TEXT | 支持字符串操作 |
| 日期时间型 | DATE, DATETIME, TIMESTAMP | 支持日期函数 |
| 布尔型 | BOOLEAN | TRUE/FALSE |
| 二进制型 | BLOB, JSON | 非结构化数据 |

### 类型兼容性规则

| 左侧类型 | 兼容的右侧类型 |
|----------|----------------|
| INTEGER | INTEGER, BIGINT, FLOAT, DOUBLE, DECIMAL |
| VARCHAR | VARCHAR, TEXT |
| DATE | DATE, DATETIME, TIMESTAMP |
| BOOLEAN | BOOLEAN |

---

## 4. ColumnRef 结构

**核心作用**：表示SQL语句中对列的引用。

### 字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| columnName | String | 列名 |
| tableRef | String | 表引用（别名或表名） |
| position | Position | 位置信息 |

### Position 结构

| 字段 | 类型 | 说明 |
|------|------|------|
| line | int | 行号 |
| column | int | 列号 |

---

## 5. 验证规则详解

### 5.1 表存在性验证

**核心作用**：检查SQL中引用的表是否在数据库中存在。

```
验证逻辑：
1. 从SQL AST提取所有表引用
         ↓
2. 获取数据库Schema中的表名列表
         ↓
3. 逐个对比，检查是否存在
         ↓
4. 返回不存在的表列表
```

### 5.2 列存在性验证

**核心作用**：检查SQL中引用的列是否在对应表中存在。

```
验证逻辑：
1. 从SQL AST提取所有列引用
         ↓
2. 获取表别名到实际表名的映射
         ↓
3. 对每个列引用，检查是否在对应表中
         ↓
4. 处理模糊列引用（无表前缀）
         ↓
5. 返回不存在的列列表
```

### 5.3 类型兼容性验证

**核心作用**：检查比较运算和表达式中操作数类型的兼容性。

```
验证逻辑：
1. 提取所有比较表达式
         ↓
2. 推断左侧和右侧操作数类型
         ↓
3. 检查类型是否兼容
         ↓
4. 返回类型不兼容的错误
```

### 5.4 聚合规范验证

**核心作用**：检查SELECT列表中的聚合函数使用是否符合规范。

```
验证规则：
1. 如果使用聚合函数（COUNT, SUM等）
         ↓
2. 且没有GROUP BY子句
         ↓
3. 则SELECT列表中只能有聚合表达式
         ↓
4. 如果有GROUP BY子句
         ↓
5. 则非聚合列必须在GROUP BY中

示例（错误）：
SELECT name, COUNT(*) FROM users
-- name 既不是聚合函数也不是GROUP BY列

示例（正确）：
SELECT name, COUNT(*) FROM users GROUP BY name
```

### 5.5 NULL比较规范

**核心作用**：检查NULL值的比较是否使用正确的语法。

```
验证规则：
1. 检测 WHERE 子句中的比较表达式
         ↓
2. 如果使用 = NULL 或 <> NULL
         ↓
3. 建议改为 IS NULL 或 IS NOT NULL

示例（不规范）：
WHERE name = NULL
WHERE age <> NULL

示例（规范）：
WHERE name IS NULL
WHERE age IS NOT NULL
```

---

## 错误码

| 错误码 | 说明 | 严重程度 |
|--------|------|----------|
| `SEM001` | 表不存在 | Error |
| `SEM002` | 列不存在 | Error |
| `SEM003` | 类型不兼容 | Error |
| `SEM004` | 聚合与 GROUP BY 不匹配 | Warning |
| `SEM005` | NULL 比较不规范 | Warning |
| `SEM006` | 模糊列引用 | Warning |

---

## 验证示例

```java
DatabaseSchema schema = loadSchema("schema.json");
SemanticValidator validator = new SemanticValidatorImpl(schema);

ValidationResult result = validator.validate(
    "SELECT name, COUNT(*) FROM users",
    context
);
// Warning: Column 'name' not in GROUP BY

result = validator.validate(
    "SELECT * FROM usrs WHERE age = '25'",
    context
);
// Error: Table 'usrs' does not exist
// Error: Type mismatch: comparing INTEGER with VARCHAR

result = validator.validate(
    "SELECT * FROM users WHERE name = NULL",
    context
);
// Warning: Use IS NULL instead of = NULL
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
```
