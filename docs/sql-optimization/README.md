# SQL优化文档索引

本目录包含 SQL 优化的各策略详细实现文档。

## 文档列表

| 文档 | 策略 | 说明 |
|------|------|------|
| [prompt-optimization.md](prompt-optimization.md) | 提示词嵌入 | 在Prompt中嵌入性能约束 |
| [sql-rewriter.md](sql-rewriter.md) | SQL重写 | 自动重写和优化SQL |
| [execution-plan-analyzer.md](execution-plan-analyzer.md) | 执行计划分析 | 分析EXPLAIN结果 |

## 方法选择指南

| 场景 | 推荐策略 | 原因 |
|------|----------|------|
| 生成阶段优化 | 提示词嵌入 | 预防胜于治疗 |
| 生成后优化 | SQL重写 | 自动修复常见问题 |
| 验证阶段分析 | 执行计划分析 | 准确评估性能 |

## 组合策略

```mermaid
flowchart LR
    A[SQL生成] --> B[提示词嵌入]
    B --> C[LLM生成]
    C --> D[SQL重写]
    D --> E[执行计划分析]
    E --> F[性能评分]
    
    style F fill:#90EE90,stroke:#333,stroke-width:2px
```

推荐流程：
1. **提示词嵌入** = 生成阶段预防
2. **SQL重写** = 生成后自动优化
3. **执行计划分析** = 验证阶段评估
