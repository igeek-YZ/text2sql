# 句法检测

**层次**：结构级别验证

**核心作用**：验证SQL语句的语法结构是否完整正确，包括子句完整性、括号匹配、关键字顺序等。

## 检测内容

| 检测项 | 说明 | 示例 |
|--------|------|------|
| 子句完整性 | 必需子句是否存在 | SELECT必须有表达式 |
| 括号匹配 | 左右括号数量一致 | `(...)` 配对 |
| 关键字顺序 | 子句顺序是否符合规范 | WHERE在FROM之后 |
| 表达式结构 | 表达式语法正确性 | `COUNT(*)` 正确 |
| 别名语法 | AS子句格式正确 | `AS alias` |

## 验证流程

```
SQL输入
   │
   ▼
┌──────────────┐
│  SQL规范化   │  预处理格式
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ 括号匹配检查 │  统计()[]{}
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ 子句解析    │  识别SELECT/FROM等
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ AST构建     │  生成语法树
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ 结构验证    │  验证节点类型
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  错误报告    │
└──────────────┘
```

---

## 1. SyntacticValidator 接口

**核心作用**：定义SQL语法验证的统一接口，支持括号匹配、子句完整性和关键字顺序验证。

### 1.1 接口方法

| 方法 | 参数 | 返回值 | 说明 |
|------|------|--------|------|
| parse | sql | ParseTree | 解析SQL生成语法树 |
| validate | sql | ValidationResult | 执行完整语法验证 |
| validateBracketMatch | sql | ValidationResult | 验证括号匹配 |
| validateClauseOrder | ast | ValidationResult | 验证子句顺序 |
| validateStructure | ast | ValidationResult | 验证结构完整性 |

### 1.2 执行步骤

```
【validate 方法】
1. 调用 validateBracketMatch 检查括号
         ↓
2. 解析SQL生成语法树（ParseTree）
         ↓
3. 验证子句完整性
         ↓
4. 验证关键字顺序
         ↓
5. 检查表达式结构
         ↓
6. 返回验证结果

【validateBracketMatch 方法】
1. 遍历SQL字符
         ↓
2. 遇到左括号入栈
         ↓
3. 遇到右括号出栈并检查匹配
         ↓
4. 检查栈是否为空
         ↓
5. 返回括号错误
```

---

## 2. ParseTree 结构

**核心作用**：表示SQL语句的语法分析树，是语法验证的基础。

### 字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| type | ParseTreeType | 语法树节点类型 |
| text | String | 节点文本内容 |
| children | List<ParseTree> | 子节点列表 |
| parent | ParseTree | 父节点 |

### ParseTreeType 枚举

| 枚举值 | 说明 |
|--------|------|
| SelectStmt | SELECT语句 |
| InsertStmt | INSERT语句 |
| UpdateStmt | UPDATE语句 |
| DeleteStmt | DELETE语句 |
| CreateTableStmt | CREATE TABLE语句 |
| SelectList | SELECT列列表 |
| FromClause | FROM子句 |
| WhereClause | WHERE子句 |
| GroupByClause | GROUP BY子句 |
| HavingClause | HAVING子句 |
| OrderByClause | ORDER BY子句 |
| LimitClause | LIMIT子句 |
| JoinClause | JOIN子句 |
| Expression | 表达式 |

---

## 3. AST 结构

**核心作用**：表示SQL语句的抽象语法树，包含语句类型和组件信息。

### 3.1 SelectStatement 结构

**核心作用**：表示SELECT查询语句的完整结构。

#### 字段说明

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| type | String | - | 语句类型（select/insert/update/delete） |
| distinct | boolean | - | 是否使用DISTINCT |
| expressions | List<Expression> | - | SELECT表达式列表 |
| from | FromClause | - | FROM子句 |
| where | WhereClause | - | WHERE子句 |
| groupBy | GroupByClause | - | GROUP BY子句 |
| having | HavingClause | - | HAVING子句 |
| orderBy | OrderByClause | - | ORDER BY子句 |
| limit | LimitClause | - | LIMIT子句 |

### 3.2 FromClause 结构

#### 字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| tableSource | TableSource | 表来源 |
| joinClauses | List<JoinClause> | JOIN子句列表 |
| aliases | Map<String, String> | 表别名映射 |

### 3.3 WhereClause 结构

#### 字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| condition | Expression | WHERE条件表达式 |

### 3.4 GroupByClause 结构

#### 字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| columns | List<String> | GROUP BY列列表 |
| having | HavingClause | HAVING条件 |

### 3.5 Expression 结构

**核心作用**：表示SQL表达式的基本结构。

#### 字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| type | ExpressionType | 表达式类型 |
| operator | String | 操作符 |
| left | Expression | 左操作数 |
| right | Expression | 右操作数 |
| value | Object | 字面量值 |

---

## 4. 括号匹配规则

**核心作用**：验证SQL中括号的开闭是否正确配对。

### 4.1 支持的括号类型

| 左括号 | 右括号 | 说明 |
|--------|--------|------|
| ( | ) | 圆括号 |
| [ | ] | 方括号 |
| { | } | 大括号 |

### 4.2 括号匹配算法

```
算法：栈匹配算法
1. 初始化空栈
         ↓
2. 遍历SQL每个字符
         ↓
3. 遇到左括号 → 入栈
         ↓
4. 遇到右括号 → 检查栈顶是否匹配
         ↓
   - 匹配 → 出栈
         ↓
   - 不匹配 → 报错
         ↓
5. 遍历结束
         ↓
6. 检查栈是否为空
         ↓
   - 非空 → 有未闭合括号
```

### 4.3 错误类型

| 错误类型 | 说明 | 示例 |
|----------|------|------|
| 未闭合括号 | 左括号没有匹配 | `SELECT * FROM users WHERE (age > 18` |
| 多余闭合括号 | 右括号没有匹配 | `SELECT * FROM users) WHERE age > 18` |
| 括号不匹配 | 左右括号类型不配对 | `SELECT * FROM users WHERE (age > 18]` |

---

## 5. 子句完整性规则

**核心作用**：验证SQL语句必需子句的存在性和正确性。

### 5.1 SELECT语句子句规则

| 子句 | 必需性 | 规则 |
|------|--------|------|
| SELECT | 必须 | 必须有至少一个表达式 |
| FROM | 可选 | 有列引用时必须 |
| WHERE | 可选 | 根据查询需求 |
| GROUP BY | 条件必需 | 有聚合函数且无HAVING时可选，有HAVING时必须 |
| HAVING | 条件必需 | 有GROUP BY时可选，必须配合GROUP BY使用 |
| ORDER BY | 可选 | 排序需求 |
| LIMIT | 可选 | 结果限定 |

### 5.2 子句顺序规则

**核心作用**：验证SQL子句出现的顺序是否符合规范。

#### 标准顺序

```
SELECT → FROM → JOIN → WHERE → GROUP BY → HAVING → ORDER BY → LIMIT
```

#### 顺序要求表

| 位置 | 子句 | 前置要求 |
|------|------|----------|
| 1 | SELECT | 必须首位 |
| 2 | FROM | - |
| 3 | JOIN | FROM之后 |
| 4 | WHERE | FROM/JOIN之后 |
| 5 | GROUP BY | WHERE之后 |
| 6 | HAVING | GROUP BY之后 |
| 7 | ORDER BY | HAVING之后 |
| 8 | LIMIT | ORDER BY之后 |

### 5.3 依赖关系规则

| 规则 | 说明 |
|------|------|
| HAVING依赖GROUP BY | HAVING必须在GROUP BY之后 |
| 聚合列规则 | SELECT中的非聚合列必须在GROUP BY中 |
| 别名引用 | ORDER BY中可引用SELECT别名 |

---

## 6. 表达式结构规则

**核心作用**：验证SQL表达式语法是否正确。

### 6.1 表达式类型

| 类型 | 示例 | 验证规则 |
|------|------|----------|
| 列引用 | `column_name`, `table.column` | 标识符格式 |
| 字面量 | `'text'`, `123`, `TRUE` | 类型正确 |
| 函数调用 | `COUNT(*)`, `SUM(amount)` | 函数名+括号 |
| 运算符 | `a + b`, `x > y` | 操作数+运算符 |
| 逻辑表达式 | `a AND b`, `NOT c` | 布尔表达式 |

### 6.2 聚合函数列表

| 函数 | 说明 | 示例 |
|------|------|------|
| COUNT | 计数 | COUNT(*), COUNT(column) |
| SUM | 求和 | SUM(amount) |
| AVG | 平均值 | AVG(price) |
| MIN | 最小值 | MIN(score) |
| MAX | 最大值 | MAX(value) |
| GROUP_CONCAT | 字符串连接 | GROUP_CONCAT(name) |

---

## 7. AST 构建流程

**核心作用**：将SQL文本解析为抽象语法树结构。

```
构建流程：
1. SQL输入
         ↓
2. 词法分析（Tokenization）
   - 分词处理
   - 关键字识别
   - 标识符提取
         ↓
3. 语法分析（Parsing）
   - 构建Token流
   - 匹配语法规则
   - 生成语法树节点
         ↓
4. AST优化
   - 别名解析
   - 表引用解析
   - 类型推断
         ↓
5. 返回完整AST
```

---

## 错误码

| 错误码 | 说明 | 严重程度 |
|--------|------|----------|
| `SYN001` | 子句缺失 | Error |
| `SYN002` | 括号不匹配 | Error |
| `SYN003` | 关键字顺序错误 | Error |
| `SYN004` | 表达式结构异常 | Warning |

---

## 验证示例

```java
SyntacticValidator validator = new SyntacticValidatorImpl("mysql");

ValidationResult result = validator.validate("SELECT * FROM users WHERE (age > 18");
// Error: Unclosed bracket at position 31

result = validator.validate("HAVING COUNT(*) > 5");
// Error: HAVING clause requires GROUP BY

result = validator.validate("SELECT * FORM users");
// Error: Clause order error or parse error
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
