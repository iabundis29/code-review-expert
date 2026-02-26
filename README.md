# Code Review Expert

A comprehensive code review skill for AI agents. Performs structured reviews with a senior engineer lens, covering architecture, security, performance, and code quality.

## Installation

### Quick Start
Install to your current project and all detected agents:

```bash
npx skills add iabundis29/code-review-expert
```

### Installation Options

**For GitHub Copilot Agent**:
```bash
npx skills add iabundis29/code-review-expert -a github-copilot
```

**Install to specific agents** (GitHub Copilot, Cursor, Claude Code, etc.):
```bash
npx skills add iabundis29/code-review-expert -a github-copilot -a cursor -a claude-code
```

**Install globally** (available across all projects):
```bash
npx skills add iabundis29/code-review-expert -g
```

**List available skills before installing**:
```bash
npx skills add iabundis29/code-review-expert --list
```

**Non-interactive installation** (CI/CD friendly):
```bash
npx skills add iabundis29/code-review-expert -a github-copilot -y
```

**Install to all agents without prompts**:
```bash
npx skills add iabundis29/code-review-expert --all
```

### Installation Methods

The installer supports two modes:
- **Symlink** (recommended) - Single source of truth, easy updates
- **Copy** - Independent copies for each agent (use `--copy` flag if needed)

### Verify Installation

List all installed skills:
```bash
npx skills list
```

or for specific agents:
```bash
npx skills list -a claude-code -a cursor
```

## Features

- **SOLID Principles** - Detect SRP, OCP, LSP, ISP, DIP violations
- **Security Scan** - XSS, injection, SSRF, race conditions, auth gaps, secrets leakage
- **Performance** - N+1 queries, CPU hotspots, missing cache, memory issues
- **Error Handling** - Swallowed exceptions, async errors, missing boundaries
- **Boundary Conditions** - Null handling, empty collections, off-by-one, numeric limits
- **Removal Planning** - Identify dead code with safe deletion plans

## How to Use

### With GitHub Copilot Agent

Once installed in GitHub Copilot, invoke it with:

```
@code-review-expert
```

or

```
/code-review-expert
```

The skill will analyze your current git changes and provide a structured code review.

### With Other Agents

Invocation syntax varies by agent platform—check your agent's documentation for the correct syntax.

### What Happens

The skill performs a **structured, multi-phase code review** of your current git changes:

1. **Analyzes your changes** - Examines `git diff` to scope modifications
2. **Checks SOLID principles** - Identifies architecture and design pattern violations
3. **Scans for removal candidates** - Finds dead code and unused features
4. **Security audit** - Detects XSS, injection, race conditions, secrets leakage
5. **Code quality review** - Flags error handling, performance, boundary condition issues
6. **Generates findings** - Categorizes issues by severity (P0–P3)
7. **Requests confirmation** - Asks before implementing any auto-fixes

### Example Workflow

```
You:  @code-review-expert
Agent: Analyzing your git changes...
       - Found 2 files modified
       - Scanning for SOLID violations...
       - Running security checks...
       - Results:
         • P0: Hardcoded API key in config.ts
         • P1: Race condition in user.service.ts
         • P2: Missing error handling in fetch
       
       Would you like me to fix these issues?
```

### Review Without Implementation

The skill defaults to **review-only mode**. It will not modify your code unless you explicitly request it.

If you want only findings without auto-fixes, the skill respects that preference throughout.

## Internal Workflow

Behind the scenes, the skill follows this structured process:

| Phase | What It Does |
|-------|------|
| **1. Preflight** | Runs `git diff --stat` and `git diff` to scope changes |
| **2. SOLID Analysis** | Checks design principles (SRP, OCP, LSP, ISP, DIP) |
| **3. Removal Scan** | Identifies dead code, unused features, deprecations |
| **4. Security Audit** | Detects injection, XSS, auth gaps, race conditions, secrets |
| **5. Code Quality** | Reviews error handling, performance, boundary conditions |
| **6. Output** | Groups findings by severity (P0 = critical, P3 = optional) |
| **7. Confirmation** | Asks user before implementing fixes |

## Managing the Skill

### Update to Latest Version
Check if updates are available:
```bash
npx skills check
```

Update all installed skills:
```bash
npx skills update
```

### Remove the Skill
Remove interactively:
```bash
npx skills remove
```

Remove specific skill by name:
```bash
npx skills remove code-review-expert
```

Remove from global scope:
```bash
npx skills remove code-review-expert -g
```

Remove from specific agents only:
```bash
npx skills remove code-review-expert -a claude-code -a cursor
```

## Supported Agents

This skill works with **GitHub Copilot** and 40+ other agent platforms that implement the [Agent Skills specification](https://agentskills.io/):

### Primary Support
- **GitHub Copilot** (Agent) - Recommended agent for this skill
- Cursor
- Continue
- Claude Code
- Cline

### Other Supported Agents
- GitHub Copilot Chat
- OpenCode
- Windsurf
- OpenHands
- And 30+ more agents

**Note**: Invocation syntax varies by agent. For GitHub Copilot Agent, use `/code-review-expert` or `@code-review-expert` depending on your version.

## Tips for Best Results

1. **Run on uncommitted changes** - The skill reviews `git diff`, so make sure your changes are in the working directory or staged
2. **Small, focused PRs** - Reviews are most effective on changes < 500 lines; for larger diffs, the skill summarizes by file
3. **Follow suggestions** - The skill defaults to review-only; implement fixes at your discretion
4. **Check git status** - If no changes are detected, verify with `git status` and `git diff`
5. **Use for learning** - The skill explains *why* issues matter, not just what to fix

## Severity Levels

| Level | Name | Action |
|-------|------|--------|
| P0 | Critical | Must block merge |
| P1 | High | Should fix before merge |
| P2 | Medium | Fix or create follow-up |
| P3 | Low | Optional improvement |

## Structure

```
code-review-expert/
├── SKILL.md                 # Main skill definition
├── agents/
│   └── agent.yaml           # Agent interface config
└── references/
    ├── solid-checklist.md   # SOLID smell prompts
    ├── security-checklist.md    # Security & reliability
    ├── code-quality-checklist.md # Error, perf, boundaries
    └── removal-plan.md      # Deletion planning template
```

## References

Each checklist provides detailed prompts and anti-patterns:

- **solid-checklist.md** - SOLID violations + common code smells
- **security-checklist.md** - OWASP risks, race conditions, crypto, supply chain
- **code-quality-checklist.md** - Error handling, caching, N+1, null safety
- **removal-plan.md** - Safe vs deferred deletion with rollback plans

## License

MIT
