---
name: "flowchart-doc"
description: "Documents Mermaid flowcharts with node specifications, exception handling, and verification checklist. Invoke when user wants to document flowcharts with detailed specs or verify existing diagrams."
---

# Flowchart Documentation Skill

This skill provides comprehensive documentation for technical flowcharts, ensuring all nodes have detailed specifications, exceptions are handled, and quality standards are met.

## When to Use

Invoke this skill when:
- User wants to create detailed documentation for flowcharts
- User asks to document or verify technical processes
- Creating SKILLs or technical guides with flow diagrams
- Auditing existing flowcharts for completeness

## Node Documentation Template

For each node in the flowchart, document the following:

```markdown
### Node: [Node Name]

**Node Responsibilities**
- 节点职责：该步骤具体做什么
- Clear description of the step's purpose

**Input Parameters**
| Parameter | Type | Format | Required | Description |
|-----------|------|--------|----------|-------------|
| param_name | string/object | JSON | Yes | Description |

**Output Specification**
| Field | Type | Format | Description |
|-------|------|--------|-------------|
| field_name | string | JSON | Description |

**Constraints**
| Type | Requirement | Value |
|------|-------------|-------|
| Performance | Latency | ≤Xms |
| Performance | Throughput | ≥X QPS |
| Environment | Online/Offline | Online |
| Security | Compliance | GDPR/ISO27001 (optional) |

**Key Logic**
- Algorithm or rule details
- Model names if ML-based
- Can reference: [external-doc.md](external-doc.md)

**Dependencies**
| Resource | Type | Description |
|----------|------|-------------|
| service_name | REST API | Endpoint description |
| db_name | Database | Table/collection |
| model.bin | Model File | Path and version |
```

## Exception Handling Section

```markdown
## 4. Exception Handling

### 4.1 Exception Scenarios

| Exception | Category | Trigger Condition | Severity |
|-----------|----------|-------------------|----------|
| 输入图片模糊 | Input | Image sharpness < threshold | HIGH |
| 模型超时 | Service | Response time > 30s | CRITICAL |
| 返回字段缺失 | Result | Required field = null | HIGH |

### 4.2 Handling Strategies

| Exception | Strategy | Action | Fallback |
|-----------|----------|--------|----------|
| 输入图片模糊 | Retry | Re-request with quality notice | Return default avatar |
| 模型超时 | Retry+Alert | Retry 3 times, notify ops | Return cached result |
| 返回字段缺失 | Log+Default | Log error, use default value | Skip processing |

### 4.3 Boundary Conditions

| Parameter | Min | Max | Unit | Handling |
|-----------|-----|-----|------|----------|
| Image Size | 100px | 4096px | pixel | Resize to max |
| File Size | 1KB | 10MB | byte | Reject with error |
| Text Length | 1 | 2000 | char | Truncate + warning |
| Batch Size | 1 | 100 | item | Split into chunks |
```

## Verification Checklist

When completing flowchart documentation, verify:

- [ ] **Mermaid Rendering**: All `flowchart` code renders correctly
- [ ] **Logic Consistency**: Flow matches textual description
- [ ] **Decision Branches**: Each branch has clear condition (use `{}` diamond)
- [ ] **Exception Coverage**:
  - [ ] Input exceptions (invalid format, size, content)
  - [ ] Service exceptions (timeout, model failure, dependency down)
  - [ ] Result exceptions (empty, malformed, partial data)
- [ ] **Boundary Conditions**: At least min/max values for all inputs
- [ ] **Non-Functional Requirements**: Specific numbers (avoid "as fast as possible")
  - Latency: P50/P95/P99 values
  - Throughput: QPS or TPS
  - Availability: percentage or downtime budget
  - Accuracy: percentage for ML models
- [ ] **Business Clarity**: Non-technical stakeholders can understand the flow

## Usage Example

### Input:
User asks to document a Text2SQL flowchart

### Process:
1. Identify all nodes in the flowchart
2. For each node, fill in the documentation template
3. Add exception handling section
4. Verify against checklist
5. Ensure non-functional requirements have specific values

### Output:
Complete markdown documentation with:
- Node specifications
- Exception handling
- Boundary conditions
- Verification checklist marked complete

## Quick Reference

### Decision Node Syntax (Mermaid)
```
    decision{condition?}
    decision -->|yes| next_step
    decision -->|no| error_handler
    style decision fill:#f96,stroke:#333,stroke-width:2px
```

### Node Types
| Shape | Syntax | Use Case |
|-------|--------|----------|
| Rectangle | `[text]` | Process step |
| Diamond | `{text}` | Decision point |
| Rounded | `(text)` | Terminal/start/end |
| Parallelogram | `{/text/}` | Input/Output |
| Hexagon | `{{text}}` | Preparation |

## Quality Standards

| Standard | Requirement |
|----------|-------------|
| Completeness | All nodes documented |
| Precision | No "etc", "and so on" |
| Measurability | All constraints have numbers |
| Traceability | External refs to detailed docs |
| Reviewability | Checklist completed |
