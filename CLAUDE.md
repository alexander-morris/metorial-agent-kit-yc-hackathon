# METORIAL Integration Assistant

**Primary Mission**: Help users effectively USE Metorial (https://metorial.com/api) for memory management and retrieval.

## Parallel Agent Orchestration for Metorial Work

**CRITICAL**: When user says "work on [something]", ALWAYS launch parallel agents via bash + Claude CLI. Each agent does a small part of the larger work (research, code improvements, etc.).

```bash
claude --model claude-haiku-4-5 --print "[agent_role] [complete_task] Write all findings to /Users/almorris/yc-hack-metorial/[output_filename].md" &
```

### Agent Requirements
- **Model**: ALWAYS use `claude-haiku-4-5` for agent dispatch
- **Output File**: EVERY agent MUST write findings to a markdown file in `/Users/almorris/yc-hack-metorial/`
- **File paths**: Absolute paths to Metorial integration code
- **Search patterns**: Metorial API calls, memory operations, retrieval patterns
- **Citations**: `file:line` format with exact code snippets
- **Dual Output**:
  1. Shared markdown findings in this project folder (REQUIRED for every agent)
  2. Code improvements written to relevant code folders in this project

### Model Strategy for Metorial Work
- **Haiku 4.5**: API discovery, pattern cataloging, documentation, simple code changes (PRIMARY for agent dispatch)
- **Sonnet**: Integration logic analysis, moderate code improvements
- **Opus**: Complex architecture analysis, advanced code refactoring

### Agent Tasks
Each agent handles a small portion:
- Research specific Metorial API usage patterns
- Improve specific code files or functions
- Document findings in shared .md
- Write/update code in project folders

**Goal**: Agents collaborate to maximize Metorial integration - publishing both analysis (.md) and code improvements to this project.