# Autonomous Epic-to-Production Pipeline for example-mcp-app-security

## Overview

Adapt treadmill's autonomous development pipeline (proven on elastic/kibana) to the `elastic/example-mcp-app-security` repo — a TypeScript MCP App with React views, Elasticsearch integration, and GitHub Actions CI.

**Key difference from Kibana**: This is a standalone npm project (~1200 LOC TypeScript, ~180 LOC CSS) with simple CI (lint → typecheck → build → test), no Buildkite, no monorepo complexity. The challenge is MCP App-specific: tools and views co-evolve, views are self-contained HTML bundles, and tool results must satisfy both LLM consumption and React rendering.

---

## Phase 1: Project Adapter (Treadmill Core Changes)

**Goal**: Make treadmill's dispatch pipeline project-agnostic.

### 1.1 detect-checks.ts — Generic npm Project Detection

Current `detect-checks.ts` assumes Kibana scripts. Add a `detectNpmProject()` path:

```typescript
// If package.json exists and has standard scripts, use them directly
if (existsSync(join(worktree, 'package.json'))) {
  const pkg = JSON.parse(readFileSync(join(worktree, 'package.json'), 'utf8'));
  return {
    typecheck: pkg.scripts?.typecheck ? 'npm run typecheck' : null,
    lint: pkg.scripts?.lint ? 'npm run lint' : null,
    test: pkg.scripts?.['test:run'] ?? pkg.scripts?.test ? 'npm run test:run' : null,
    build: pkg.scripts?.build ? 'npm run build' : null,
  };
}
```

### 1.2 buildScopedCommand — Project-Aware Path Resolution

Fix pitfall #26: `buildScopedCommand` must check `existsSync()` before using Kibana script paths. For non-Kibana projects, fall back to `baseCommand` from detect-checks.

### 1.3 Pre-PR Script Generation — npm-Aware

`treadmill_pre_pr_script` should generate:
```bash
#!/bin/bash
set -euo pipefail

# Stack health (project-specific)
npm run typecheck || { echo "FAIL: typecheck"; exit 1; }
npm run lint || { echo "FAIL: lint"; exit 1; }
npm run build || { echo "FAIL: build"; exit 1; }
npm run test:run || { echo "FAIL: tests"; exit 1; }
```

### 1.4 GitHub Actions CI Integration

Replace Buildkite-specific CI diagnosis with GitHub Actions:
- Use `gh run list --repo elastic/example-mcp-app-security --branch <branch>` to check CI status
- Parse failure logs via `gh run view <run-id> --log-failed`
- Surface CI failures in the auto-pilot loop

### 1.5 Smoke Test Injection — MCP App Variant

For MCP App projects, inject a smoke test that verifies the server starts and responds to tool calls:

```bash
# Start MCP server in background
ELASTICSEARCH_URL=mock ES_API_KEY=mock node dist/main.js --stdio &
MCP_PID=$!
sleep 2

# Verify process is alive
kill -0 $MCP_PID 2>/dev/null || { echo "FAIL: MCP server crashed on startup"; exit 1; }

# Verify build artifacts exist (self-contained HTML views)
for view in alert-triage attack-discovery case-management detection-rules threat-hunt sample-data; do
  [ -f "dist/views/$view/mcp-app.html" ] || { echo "FAIL: missing view $view"; exit 1; }
done

kill $MCP_PID
```

---

## Phase 2: Quality Gates

### Gate 1: Static Analysis (inherited from CI)
- `npm run typecheck` — TypeScript strict mode
- `npm run lint` — ESLint with custom license rule
- `npm run build` — Full build (server + views)

### Gate 2: Unit Tests
- `npm run test:run` — Vitest suite
- Existing tests: tool handlers, client mocks, analytics, shared utilities
- **Coverage gate**: Track line coverage, fail if coverage drops (currently ~60% estimated)

### Gate 3: View Build Integrity
- All 6 views produce valid self-contained HTML files in `dist/views/`
- Each HTML file is < 5MB (self-contained includes all JS/CSS)
- HTML passes basic validation (has `<html>`, `<head>`, `<body>`)

### Gate 4: Tool Response Schema Validation (new — from idea capture)
- Zod schemas for all model-facing tool responses
- Contract tests asserting response shapes match view expectations
- Runtime validation in tool handlers (dev mode)

### Gate 5: MCP Protocol Compliance
- Server starts without errors in stdio mode
- All model-facing tools are registered with correct names
- Tool descriptions are non-empty
- No duplicate tool names

### Gate 6: Dependency Audit
- `npm audit --audit-level=high` — no high/critical vulnerabilities
- Package-lock.json is up to date (`npm ci` succeeds)

### Gate 7: Git Hygiene
- No `.env` file committed
- No hardcoded API keys in source
- License headers present (enforced by custom ESLint rule)

---

## Phase 3: Autonomous Pipeline State Machine

Adapts the Kibana 11-state epic pipeline to the MCP app's simpler lifecycle:

```
             ┌──────────────────────────────────────┐
             │                                      │
  ┌──────┐   │  ┌──────────┐   ┌──────────┐        │
  │ idea │───┼─►│ shaping  │──►│ slicing  │        │
  └──────┘   │  └──────────┘   └──────────┘        │
             │       │              │               │
             │       ▼              ▼               │
             │  ┌──────────┐   ┌──────────┐        │
             │  │waiting_pm│   │in_progress│        │
             │  └──────────┘   └──────────┘        │
             │       │              │               │
             │       ▼              ▼               │
             │  ┌──────────┐   ┌──────────┐        │
             │  │ assessed │   │ pr_open  │         │
             │  └──────────┘   └──────────┘        │
             │                      │               │
             │                      ▼               │
             │                 ┌──────────┐         │
             │                 │ ci_green │          │
             │                 └──────────┘         │
             │                      │               │
             │                      ▼               │
             │                 ┌──────────┐         │
             │                 │ reviewed │          │
             │                 └──────────┘         │
             │                      │               │
             │                      ▼               │
             │                 ┌──────────┐         │
             │                 │  merged  │          │
             │                 └──────────┘         │
             │                      │               │
             │                      ▼               │
             │                 ┌──────────┐         │
             │                 │ released │          │
             │                 └──────────┘         │
             └──────────────────────────────────────┘
```

### State Transitions

| From → To | Trigger | Quality Gate |
|-----------|---------|-------------|
| idea → shaping | `treadmill_epic_intake` | Shape quality score ≥ 4/6 |
| shaping → slicing | Shape passes, overlap scan clean | No blocking overlaps |
| slicing → in_progress | Plan generated, feature created, plan activated | Plan has tasks |
| in_progress → pr_open | All tasks completed, `treadmill_promote_plan_to_pr` | Gates 1-7 pass |
| pr_open → ci_green | GitHub Actions CI passes | All checks green |
| ci_green → reviewed | Code review requested | Auto-review via `treadmill_review_pr` |
| reviewed → merged | Review approved, no blocking findings | Confidence > 0.7 |
| merged → released | Post-merge CI green, tag cut | No reverts detected |

### Key Differences from Kibana Pipeline

| Aspect | Kibana | MCP App |
|--------|--------|---------|
| CI | Buildkite (complex, multi-step) | GitHub Actions (single job) |
| Build | `yarn kbn bootstrap` + plugin build | `npm ci && npm run build` |
| Tests | Jest + Scout + Cypress | Vitest only |
| Smoke test | ES Docker + Kibana source + browser | MCP server startup + view file check |
| Port allocation | Dynamic ES/Kibana ports (9201-9299, 5610-5699) | Not needed (stdio mode) |
| Pre-PR | tsc + jest + eslint + i18n + check_changes + grep gates | npm run typecheck + lint + build + test:run |
| PR labels | `release-note:enhancement`, `deploy:staging` | None (no staging deployment) |
| CODEOWNERS | Complex multi-team | Single team |
| Feature flags | Required for new features | Not used |

---

## Phase 4: Worktree Strategy

```
~/Projects/example-mcp-app-security                    # main checkout (never work here)
~/Projects/example-mcp-app-security.worktrees/
  ao-feat-<feature-slug>/                              # feature worktree
    .ao/
      stack.json                                        # not needed (no ES Docker)
      agent-stream-*.jsonl                              # agent session logs
      status.json                                       # job completion status
    openspec/changes/<plan-slug>/                       # OpenSpec artifacts
      proposal.md
      research.md
      design.md
      specs.md
      tasks.md
```

### Git Remote Setup

The repo uses SSH remote. Fork to `patrykkopycinski/example-mcp-app-security` for PR creation:

```bash
cd ~/Projects/example-mcp-app-security
git remote add fork git@github.com:patrykkopycinski/example-mcp-app-security.git
```

---

## Phase 5: Agent Prompt Injection

### Project Context (injected into every agent dispatch)

```markdown
## Project: elastic/example-mcp-app-security

TypeScript MCP App with 6 interactive React views for SOC operations.
Architecture: model-facing tools return text + HTML views; app-only tools
provide interactivity hidden from LLM.

### Build & Test
npm ci
npm run typecheck
npm run lint
npm run build          # builds server + all 6 views
npm run test:run       # vitest suite

### Key Directories
src/tools/          — tool handlers (model-facing + app-only)
src/elastic/client/ — Elasticsearch API clients
src/views/          — React view apps (one per tool)
skills/             — Claude Skills (SKILL.md files)
docs/               — architecture, setup, permissions

### Quality Requirements
- All views must be self-contained HTML (vite-plugin-singlefile)
- Tool responses: compact text for LLM (<5KB), full data for views
- ES API compatibility: elastic-api-version 2023-10-31 headers
- License: Elastic-2.0 header required on all source files
- No .env committed, no hardcoded credentials
```

### Smoke Test (injected into Kibana-equivalent position)

```markdown
## SMOKE TEST (MANDATORY)

Before claiming task completion:
1. Run `npm run build` — server + all views compile
2. Verify all 6 view HTML files exist in dist/views/
3. Run `npm run test:run` — all tests pass
4. If you modified a tool response shape, verify the corresponding view
   still receives the expected data structure
```

---

## Phase 6: Release Automation

### Versioning
The repo uses semver in package.json + manifest.json (synced via `npm version` hook).

### Release Workflow
`.github/workflows/release.yml` creates GitHub releases with `.mcpb` bundles.

### Post-Merge Monitoring
- Watch for CI failures on main after merge
- Watch for new issues filed within 48h of merge (regression indicator)
- If `.mcpb` bundle size increases >20%, flag for review

---

## Phase 7: Research Backlog (Pre-requisites)

Before activating the autonomous pipeline, complete these research items (captured as treadmill ideas):

1. **MCP Apps Specification** — understand what's stable vs changing in the app API surface
2. **Quality Gates for Tool+View Co-evolution** — define how to detect view breakage from tool changes
3. **Treadmill Project Adapter** — implement the non-Kibana detect-checks + pre-PR script changes
4. **Streamable-HTTP Transport** — evaluate for remote deployment scenarios

### Research Priority Order
```
[3] Project Adapter (blocks pipeline activation)
  → [1] MCP Apps Spec (informs gate design)
    → [2] Tool+View Quality Gates (defines gate 4)
      → [4] HTTP Transport (nice-to-have, not blocking)
```

---

## Implementation Roadmap

### Sprint 1: Foundation (Week 1)
- [ ] Fork repo, set up worktree infrastructure
- [ ] Implement detect-checks.ts npm project detection in treadmill
- [ ] Implement buildScopedCommand non-Kibana fallback
- [ ] Generate first pre-PR script for the repo
- [ ] Verify gates 1-3, 5-7 pass on current main

### Sprint 2: Quality Gates (Week 2)
- [ ] Add Zod schema contract tests (Gate 4)
- [ ] Add MCP protocol compliance check (Gate 5)
- [ ] Add dependency audit step (Gate 6)
- [ ] Wire GitHub Actions CI status into treadmill

### Sprint 3: Pipeline Integration (Week 3)
- [ ] Configure treadmill to target example-mcp-app-security as projectRoot
- [ ] Test plan generation → task dispatch → verify → promote flow
- [ ] Validate PR creation via fork remote
- [ ] Test end-to-end: idea → plan → PR → CI → review

### Sprint 4: Hardening (Week 4)
- [ ] Add post-merge monitoring (CI health, issue regression)
- [ ] Tune retry budgets for smaller repo (faster cycles)
- [ ] Document the project adapter pattern for future non-Kibana repos
- [ ] Backport learnings into treadmill-operations skill

---

## Risk Assessment

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|------------|
| Kibana-specific code paths in treadmill break non-Kibana dispatch | High | Blocks pipeline | Phase 1 adapter work + integration tests |
| MCP Apps spec changes break view rendering | Medium | Breaks Gate 3 | Pin @modelcontextprotocol/ext-apps version, monitor spec |
| Agent doesn't understand tool+view co-evolution | Medium | Poor code quality | Rich project context injection, contract test feedback |
| GitHub Actions rate limits slow CI feedback loop | Low | Slow iteration | Cache npm deps, parallelize gate checks |
| Small repo = agents finish too fast, less verification needed | Low | Over-engineering | Tune turn caps and retry budgets down |
