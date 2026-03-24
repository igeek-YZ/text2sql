# Flowchart Documentation Skill

Comprehensive skill for documenting technical flowcharts with detailed specifications.

## Structure

```
flowchart-doc/
├── SKILL.md                 # Main skill file
└── docs/
    ├── README.md           # This file
    └── node-spec-example.md # Examples
```

## Usage

### When Invoked
This skill is automatically invoked when:
- User wants to document flowcharts
- User asks to verify technical processes
- Creating documentation for Mermaid diagrams

### How to Use

1. **Identify Nodes**: List all nodes in the flowchart
2. **Fill Template**: For each node, document:
   - Responsibilities
   - Input/Output specifications
   - Constraints (with specific numbers)
   - Key logic
   - Dependencies

3. **Add Exception Handling**:
   - Exception scenarios table
   - Handling strategies table
   - Boundary conditions table

4. **Verify**: Run through the checklist

## Key Templates

### Node Documentation
```markdown
### Node: [Name]

**Node Responsibilities**
- 具体职责描述

**Input Parameters**
| Parameter | Type | Format | Required |
|-----------|------|--------|----------|
| ...       | ... | ...    | Yes/No   |

**Output Specification**
| Field | Type | Format | Description |
|-------|------|--------|-------------|
| ...   | ... | ...    | ...         |

**Constraints**
| Type | Requirement | Value |
|------|-------------|-------|
| Performance | Latency | ≤Xms |
| Performance | Throughput | ≥X QPS |

**Key Logic**
- Algorithm or rule details

**Dependencies**
| Resource | Type | Description |
|----------|------|-------------|
| ...      | ...  | ...         |
```

### Exception Handling
```markdown
## Exception Handling

| Exception | Category | Trigger | Severity |
|-----------|----------|---------|----------|
| ...       | Input/Service/Result | ... | CRITICAL/HIGH/MEDIUM/LOW |

| Exception | Strategy | Action | Fallback |
|-----------|----------|--------|----------|
| ...       | Retry/Block/Log | ... | ... |
```

### Verification Checklist
- [ ] Mermaid Rendering
- [ ] Logic Consistency
- [ ] Decision Branches
- [ ] Exception Coverage (Input/Service/Result)
- [ ] Boundary Conditions
- [ ] Non-Functional Requirements (specific numbers)
- [ ] Business Clarity

## Quality Standards

| Standard | Description |
|----------|-------------|
| Completeness | All nodes documented |
| Precision | No vague descriptions |
| Measurability | All constraints have numbers |
| Traceability | External references provided |
| Reviewability | Checklist completed |
