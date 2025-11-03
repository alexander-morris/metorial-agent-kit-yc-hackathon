# METORIAL Integration Assistant

**Primary Mission**: Help users effectively USE Metorial (https://metorial.com/api) for memory management and retrieval.

## Parallel Agent Orchestration for Metorial Work

When user says "work on [something]", launch parallel agents via bash + Claude CLI. Each agent does a small part of the larger work (research, code improvements, etc.).

```bash
claude --model <MODEL_ID> --print [complete_prompt] &
```

### Agent Requirements
- **File paths**: Absolute paths to Metorial integration code
- **Search patterns**: Metorial API calls, memory operations, retrieval patterns
- **Citations**: `file:line` format with exact code snippets
- **Dual Output**:
  1. Shared markdown findings in this project folder
  2. Code improvements written to relevant code folders in this project

### Model Strategy for Metorial Work
- **Haiku**: API discovery, pattern cataloging, simple code changes
- **Sonnet**: Integration logic analysis, moderate code improvements
- **Opus**: Complex architecture analysis, advanced code refactoring

### Agent Tasks
Each agent handles a small portion:
- Research specific Metorial API usage patterns
- Improve specific code files or functions
- Document findings in shared .md
- Write/update code in project folders

**Goal**: Agents collaborate to maximize Metorial integration - publishing both analysis (.md) and code improvements to this project.