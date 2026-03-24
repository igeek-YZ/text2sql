# 词法检测

**层次**：Token级别验证

**核心作用**：对SQL进行分词处理，验证关键字、标识符的正确性，是语法检测的基础。

## 检测内容

| 检测项 | 说明 | 示例 |
|--------|------|------|
| 关键字 | SQL保留字正确性 | `SELECT`, `FROM`, `WHERE` |
| 标识符 | 表名、列名、别名格式 | `users`, `user_name` |
| 运算符 | 算术、比较、逻辑运算符 | `+`, `=`, `AND`, `OR` |
| 标点符号 | 括号、引号、分号 | `(`, `)`, `'`, `,` |
| 字面量 | 数值、字符串常量 | `'John'`, `123` |

## 验证流程

```
SQL输入
   │
   ▼
┌──────────────┐
│  Tokenization │  分词处理
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ 关键字识别   │  匹配保留字
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ 标识符提取   │  提取表名、列名
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  错误报告    │  返回Token错误
└──────────────┘
```

---

## 1. LexicalValidator 接口

**核心作用**：定义SQL词法验证的统一接口，支持分词和Token验证。

### 1.1 接口方法

| 方法 | 参数 | 返回值 | 说明 |
|------|------|--------|------|
| tokenize | sql | List<Token> | 分词处理 |
| validate | sql | ValidationResult | 验证SQL词法正确性 |
| validateTokens | tokens | ValidationResult | 验证Token列表 |

### 1.2 执行步骤

```
【tokenize 方法】
1. 接收SQL语句
         ↓
2. 按分隔符进行初步分词
         ↓
3. 对每个Token进行类型分类
         ↓
4. 记录Token位置信息
         ↓
5. 返回Token列表

【validate 方法】
1. 调用 tokenize 进行分词
         ↓
2. 检查无效Token
         ↓
3. 检测未知关键字
         ↓
4. 返回验证结果
```

---

## 2. Token 结构

**核心作用**：表示SQL语句中的最小语义单元。

### 字段说明

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| type | TokenType | - | Token类型枚举 |
| text | String | - | Token文本内容 |
| position | Position | - | 位置信息 |
| length | int | - | Token长度 |

### Position 结构

| 字段 | 类型 | 说明 |
|------|------|------|
| line | int | 行号 |
| column | int | 列号 |

---

## 3. TokenType 枚举

**核心作用**：定义SQL Token的类型分类。

### 枚举值

| 枚举值 | 说明 | 示例 |
|--------|------|------|
| KEYWORD | SQL保留关键字 | SELECT, FROM, WHERE |
| IDENTIFIER | 标识符 | users, user_name |
| OPERATOR | 运算符 | +, =, AND, OR |
| PUNCTUATION | 标点符号 | (, ), ', ', ; |
| STRING | 字符串常量 | 'John', "name" |
| NUMBER | 数值常量 | 123, 45.67 |
| COMMENT | 注释 | --, /* */ |
| WHITESPACE | 空白字符 | 空格, 换行, 制表符 |
| INVALID | 无效Token | 非法字符 |

### Token分类规则

| 类型 | 识别规则 | 优先级 |
|------|----------|--------|
| KEYWORD | 在保留字集合中 | 高 |
| STRING | 以引号包围 | 高 |
| NUMBER | 可解析为数字 | 高 |
| OPERATOR | 匹配运算符正则 | 中 |
| PUNCTUATION | 匹配标点正则 | 中 |
| IDENTIFIER | 匹配标识符正则 | 低 |
| INVALID | 其他情况 | - |

---

## 4. SQL保留关键字

**核心作用**：定义SQL语言的标准保留字。

### 关键字分类表

| 分类 | 关键字列表 |
|------|------------|
| 查询 | SELECT, DISTINCT, ALL |
| 表操作 | FROM, WHERE, JOIN, LEFT, RIGHT, INNER, OUTER, ON |
| 分组 | GROUP, BY, HAVING |
| 排序 | ORDER, ASC, DESC |
| 限定 | LIMIT, OFFSET |
| 集合 | UNION, INTERSECT, EXCEPT |
| 投影 | AS |
| 条件 | AND, OR, NOT, IN, EXISTS, BETWEEN, IS, NULL |
| 插入 | INSERT, INTO, VALUES |
| 更新 | UPDATE, SET |
| 删除 | DELETE |
| 创建 | CREATE, TABLE, VIEW, INDEX, DATABASE, SCHEMA |
| 修改 | ALTER, DROP |
| 约束 | PRIMARY, KEY, FOREIGN, REFERENCES, CONSTRAINT, DEFAULT, CHECK, UNIQUE |
| 级联 | CASCADE, RESTRICT |
| 流程 | CASE, WHEN, THEN, ELSE, END |
| 函数 | CAST, COALESCE, NULLIF, IF |

---

## 5. 标识符规则

**核心作用**：定义SQL标识符（表名、列名、别名）的命名规范。

### 命名规范

| 规则类型 | 正则表达式 | 说明 |
|----------|------------|------|
| 格式要求 | `^[a-zA-Z_][a-zA-Z0-9_]*$` | 必须以字母或下划线开头 |
| 长度限制 | 1-64字符 | 建议不超过64字符 |
| 区分大小写 | 是 | 取决于数据库配置 |
| 保留字冲突 | 需转义 | 避免与关键字同名 |

### 标识符类型

| 类型 | 说明 | 示例 |
|------|------|------|
| 表名 | 数据库表名称 | users, orders |
| 列名 | 表列名称 | user_id, user_name |
| 别名 | 查询结果别名 | u, user_info |
| 索引名 | 索引对象名称 | idx_user_id |
| 视图名 | 视图对象名称 | v_user_active |

---

## 6. 运算符规则

**核心作用**：定义SQL支持的运算符类型。

### 运算符分类

| 分类 | 运算符 | 说明 |
|------|--------|------|
| 算术 | +, -, *, / | 加减乘除 |
| 比较 | =, <>, !=, <, >, <=, >= | 等于、不等于、大小比较 |
| 逻辑 | AND, OR, NOT | 布尔逻辑运算 |
| 模式 | LIKE, IN, BETWEEN | 模式匹配运算 |
| 空值 | IS NULL, IS NOT NULL | NULL值判断 |

### 运算符正则表达式

```
^[+\-*/=<>!]+$
```

---

## 7. 字面量规则

**核心作用**：定义SQL中常量的表示方式。

### 字面量类型

| 类型 | 格式 | 示例 |
|------|------|------|
| 整数 | 十进制数字 | 123, 0, -456 |
| 浮点数 | 十进制小数 | 3.14, -0.5 |
| 字符串 | 单引号或双引号 | 'hello', "world" |
| 日期 | 数据库特定格式 | '2024-01-01' |
| 时间戳 | 数据库特定格式 | '2024-01-01 12:00:00' |
| 布尔 | TRUE/FALSE | TRUE, FALSE |
| NULL | NULL关键字 | NULL |

### 字符串字面量规则

| 规则 | 说明 |
|------|------|
| 定界符 | 单引号(')或双引号(") |
| 转义 | '' 或 \" 表示引号本身 |
| 区分大小写 | 取决于数据库配置 |

---

## 错误码

| 错误码 | 说明 | 严重程度 |
|--------|------|----------|
| `LEX001` | 无效Token | Error |
| `LEX002` | 未知关键字 | Warning |
| `LEX003` | 未闭合引号 | Error |
| `LEX004` | 无效标识符 | Error |
| `LEX005` | 非法字符 | Error |

---

## 验证示例

```java
LexicalValidator validator = new LexicalValidatorImpl("mysql");

ValidationResult result = validator.validate("SELEC * FORM users");
// Token: SELEC (unknown keyword)
// Token: FORM (unknown keyword)
// Result: valid=false, errors=[LEX001, LEX002]

List<Token> tokens = validator.tokenize("SELECT id, name FROM users WHERE age > 18");
// tokens: [SELECT, id, ,, name, FROM, users, WHERE, age, >, 18]
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
