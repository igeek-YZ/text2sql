# Flowchart Node Specification Examples

This document provides examples of how to document flowchart nodes according to the flowchart-doc skill.

## Example 1: Text2SQL Node

### Node: Schema Linking

**Node Responsibilities**
- 从数据库Schema中智能选择与查询相关的表和列
- 减少LLM处理的干扰信息
- 智能识别用户意图对应的数据库对象

**Input Parameters**
| Parameter | Type | Format | Required | Description |
|-----------|------|--------|----------|-------------|
| user_query | string | Text | Yes | 用户的自然语言查询 |
| db_schema | object | JSON | Yes | 数据库Schema信息 |
| metadata | object | JSON | No | 表/列注释、关系 |

**Output Specification**
| Field | Type | Format | Description |
|-------|------|--------|-------------|
| tables | array | JSON | 相关的表名列表 |
| columns | array | JSON | 相关的列名及所属表 |
| confidence | float | JSON | 匹配置信度 0-1 |
| relationships | array | JSON | 表之间的外键关系 |

**Constraints**
| Type | Requirement | Value |
|------|-------------|-------|
| Performance | Latency | ≤500ms |
| Performance | Throughput | ≥100 QPS |
| Environment | Online | Yes |
| Accuracy | Match Rate | ≥90% |

**Key Logic**
- 字符串匹配：Levenshtein距离 > 0.8
- 语义匹配：Sentence-BERT余弦相似度 > 0.75
- 优先级：精确匹配 > 语义匹配 > 关系匹配
- 参考：[schema-linking.md](../schema-linking.md)

**Dependencies**
| Resource | Type | Description |
|----------|------|-------------|
| Database | PostgreSQL | information_schema表 |
| Cache | Redis | Schema缓存，TTL=5min |
| Embedding Model | ONNX | sentence-transformers |

---

### Node: LLM Function Call

**Node Responsibilities**
- 调用LLM生成SQL语句
- 处理Function Calling请求
- 管理上下文和重试逻辑

**Input Parameters**
| Parameter | Type | Format | Required | Description |
|-----------|------|--------|----------|-------------|
| prompt | string | Text | Yes | 构造好的Prompt |
| function_def | array | JSON | Yes | Function定义 |
| temperature | float | JSON | No | 温度参数 0.0-0.5 |

**Output Specification**
| Field | Type | Format | Description |
|-------|------|--------|-------------|
| sql | string | Text | 生成的SQL语句 |
| confidence | float | JSON | 生成置信度 |
| tokens_used | int | JSON | 消耗的token数 |

**Constraints**
| Type | Requirement | Value |
|------|-------------|-------|
| Performance | Latency P95 | ≤2s |
| Performance | Latency P99 | ≤5s |
| Environment | Online | Yes |
| Rate Limit | RPM | 60 |

**Key Logic**
- 模型选择：根据复杂度选择单Agent/Multi-Agent
- Few-shot：添加3-5个示例提升准确性
- CoT：复杂查询启用思维链
- 参考：[llm-config.md](../llm-config.md)

**Dependencies**
| Resource | Type | Description |
|----------|------|-------------|
| MiniMax API | REST | LLM服务 |
| Prompt Cache | Redis | 相同查询缓存 |
| Token Counter | Service | 用量统计 |

---

## Decision Node Example

### Node: 验证通过?

**Decision Type**: Diamond (菱形判断)

**Condition**: `validation_result == true`

**Branches**:
| Branch | Condition | Target Node | Description |
|--------|-----------|-------------|-------------|
| Yes | result.valid == true | 执行SQL | 验证通过 |
| No | result.valid == false | 修复SQL | 需要修复，最多3次 |

**Styling**:
```mermaid
    G --> I{验证通过?}
    style I fill:#f96,stroke:#333,stroke-width:2px
```

---

## Exception Handling Example

### 4. Exception Handling

### 4.1 Exception Scenarios

| Exception | Category | Trigger Condition | Severity |
|-----------|----------|-------------------|----------|
| Schema获取失败 | Service | metadata = null | CRITICAL |
| LLM响应超时 | Service | latency > 30s | HIGH |
| SQL验证失败 | Result | syntax_error = true | HIGH |
| 空查询结果 | Result | rows.length = 0 | MEDIUM |
| 输入查询过长 | Input | length > 2000 | HIGH |
| 非法SQL注入 | Input | 包含DROP/DELETE | CRITICAL |

### 4.2 Handling Strategies

| Exception | Strategy | Action | Fallback |
|-----------|----------|--------|----------|
| Schema获取失败 | Retry | 重试3次 | 返回错误提示 |
| LLM响应超时 | Retry+Alert | 重试3次，通知运维 | 返回缓存结果 |
| SQL验证失败 | Retry | 最多重试3次 | 返回友好错误 |
| 空查询结果 | Log | 记录日志 | 返回空结果提示 |
| 输入查询过长 | Reject | 拒绝并提示 | 建议拆分查询 |
| 非法SQL注入 | Block | 直接拒绝 | 记录安全日志 |

### 4.3 Boundary Conditions

| Parameter | Min | Max | Unit | Handling |
|-----------|-----|-----|------|----------|
| 查询长度 | 1 | 2000 | char | 超出提示拆分 |
| 表数量 | 1 | 50 | 个 | 超出提示简化 |
| 列数量 | 1 | 200 | 个 | 超出提示选择 |
| 嵌套深度 | 1 | 3 | 层 | 超出降级单表 |
| 结果行数 | 0 | 10000 | 行 | 超出分页 |
| 执行超时 | 1 | 30 | 秒 | 超时终止 |

---

## Verification Checklist (Completed)

- [x] **Mermaid Rendering**: 4个flowchart全部可渲染
- [x] **Logic Consistency**: 流程与文本描述一致
- [x] **Decision Branches**: 使用 `{}` 菱形 + 明确条件
- [x] **Exception Coverage**:
  - [x] Input exceptions: 输入过长、SQL注入
  - [x] Service exceptions: Schema失败、超时
  - [x] Result exceptions: 空结果、验证失败
- [x] **Boundary Conditions**: 7项具体限制值
- [x] **Non-Functional Requirements**:
  - Latency: P95≤1.5秒, P99≤3秒
  - Throughput: ≥50 QPS
  - Accuracy: EX≥85%
- [x] **Business Clarity**: 核心作用 + 职责分工表
