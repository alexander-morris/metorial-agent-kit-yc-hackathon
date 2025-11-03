# Claude Code Directive: Parallel Agent Orchestration

## Core Capability
Launch unlimited parallel Claude agents using pure bash + Claude CLI for research, analysis, and implementation tasks.

## Execution Pattern
```bash
claude --model <MODEL_ID> --print [complete_prompt]
```

## Models Available
- `claude-haiku-4-5-20251001` (fast/cheap)
- `claude-sonnet-4-20250514` (balanced)
- `claude-opus-4-20250514` (powerful)

## Orchestration Rules

### 1. Agent Information (MANDATORY)
Each agent MUST receive:
- ✅ Complete file paths (absolute)
- ✅ Exact search patterns
- ✅ Full context and success criteria
- ✅ Shared output markdown path

### 2. Source Citations (REQUIRED)
Every finding MUST include:
- ✅ `file:line` format for ALL code references
- ✅ Exact code snippets
- ✅ Context explaining the finding

Example: `/Users/almorris/project/file.ts:45 - User ID extraction: const userId = session.user.id`

### 3. Shared Output Structure
All agents append to single markdown file:
```markdown
## Agent N (model-id) - Focus Area

### FINDINGS
[Detailed discoveries]

### SOURCE CITATIONS (MANDATORY)
- /absolute/path/file.ext:123 - finding with code snippet

### IMPLEMENTATION/CHANGES
[What was done]

### VALIDATION
[How success was verified]

### ISSUES/BLOCKERS
[Problems: FILE NOT FOUND, PATTERN NOT FOUND, etc.]

---
```

## Model Selection Strategy
- **Haiku**: Pattern matching, cataloging, API discovery
- **Sonnet**: Logic analysis, security reviews, moderate complexity
- **Opus**: Architecture analysis, complex vulnerability research

## Best Practices
- Use absolute file paths
- Provide specific search patterns
- Mix models for optimal cost/quality
- Launch 5-10 agents for most tasks
- Auto-retry failed agents (3x with exponential backoff)
- Real-time shared output with file locking

## Key Templates
1. **Research**: 10x Haiku agents sweep codebase
2. **Security**: Mixed models for comprehensive audits
3. **Documentation**: Auto-generate API docs
4. **Architecture**: Opus agents for deep analysis

**Technical Guarantee**: Pure CLI bash execution. No MCP, no Python dependencies. Agents use Read/Grep tools and append to shared markdown output.