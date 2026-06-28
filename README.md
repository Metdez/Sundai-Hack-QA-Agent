# SpecGuard — Living Specs QA Agent

A reusable, codebase-agnostic system that keeps Living Specifications as the **single source of truth** for your codebase — running in parallel with your coding agent to automatically gather requirements, generate tests, validate security, and produce documentation at every plan phase.

Inspired by [PrintingPress](https://printingpress.dev)'s approach of a slim skill + CLI + MCP. Drop it into any project in under five minutes.

---

## The Problem

Coding agents write code fast. Specifications, tests, security analysis, and documentation lag behind — or never exist at all. When they do exist, they're written once and immediately go stale. The gap between what the code does and what it's supposed to do grows with every PR.

SpecGuard closes that gap by running beside your coding agent. When the agent completes a plan phase, SpecGuard automatically:

- Captures the requirements that were just implemented into structured Living Spec files
- Generates or updates QA tests from those specs
- Flags security issues against OWASP patterns
- Detects spec drift when code changes outpace documented behaviour
- Regenerates user documentation

Every artifact traces back to a single spec file. The spec is the source of truth for code, tests, docs, and security review — not a side artifact.

---

## Architecture

Three surfaces, one core:

```
┌──────────────────────────────────────────────────────────────────┐
│                         SpecGuard Core                           │
│   spec-parser · forward-gen · reverse-gen · validate · security  │
└───────────┬──────────────────┬──────────────────┬───────────────┘
            │                  │                  │
    ┌───────▼──────┐  ┌────────▼───────┐  ┌──────▼────────┐
    │  specguard   │  │ specguard-mcp  │  │   SKILL.md    │
    │     CLI      │  │   MCP Server   │  │ Cursor Skill  │
    │  (CI / shell │  │ (Cursor /      │  │  (parallel    │
    │    agents)   │  │  Claude Desk.) │  │   agent)      │
    └──────────────┘  └────────────────┘  └──────┬────────┘
                                                  │
                                         ┌────────▼────────┐
                                         │  VS Code / Cursor│
                                         │   Extension      │
                                         │  (one-click      │
                                         │   install)       │
                                         └─────────────────┘
```

### CLI (`specguard`)

A lean CLI for shell agents and CI pipelines. Commands map directly to pipelines:

```bash
specguard init                          # scaffold .specguard/ config in any repo
specguard reverse [--app <name>]        # code → specs
specguard generate [--spec <key>]       # specs → tests
specguard validate [--spec <key>] [--url <base>]  # live app + spec → issues
specguard security [--spec <key>]       # spec + code → security test stubs
specguard docs [--spec <key>]           # specs → user documentation
specguard drift                         # detect code changes that outpace specs
specguard status                        # show spec coverage across the codebase
```

### MCP Server (`specguard-mcp`)

Exposes every pipeline as an MCP tool. IDE agents (Cursor, Claude Desktop, Windsurf) call tools directly — no shell required. The MCP server auto-discovers via `.cursor/mcp.json` after install.

Available tools:

| Tool | Description |
|---|---|
| `specguard_reverse` | Generate specs from existing code and tests |
| `specguard_generate` | Generate tests from spec files |
| `specguard_validate` | Validate a live page against its spec |
| `specguard_security` | Generate security test stubs from a spec |
| `specguard_docs` | Generate user documentation from specs |
| `specguard_drift` | Check for spec drift after a code change |
| `specguard_status` | List spec coverage across the project |
| `specguard_read_spec` | Read a spec file by key (e.g. `auth/login`) |
| `specguard_write_spec` | Write or update a spec file |

### Cursor Skill (`SKILL.md`)

The parallel agent companion. Placed in `.cursor/skills/specguard/SKILL.md`, it teaches Cursor when and how to run SpecGuard automatically:

- **After each plan phase**: calls `specguard_reverse` on changed files, then `specguard_generate` for affected specs
- **After security-sensitive changes**: calls `specguard_security`
- **Before a PR**: calls `specguard_drift` and `specguard_validate` to catch regressions
- **On demand**: any of the above triggered manually with natural language

### VS Code / Cursor Extension

One-click installation that:

1. Runs `specguard init` to scaffold `.specguard/config.json` in the workspace
2. Installs the CLI binary (via `npm install -g specguard` or binary download)
3. Registers `specguard-mcp` in `.cursor/mcp.json` and VSCode MCP settings
4. Copies `SKILL.md` into `.cursor/skills/specguard/`
5. Adds a sidebar panel for spec coverage, drift status, and recent validation reports

---

## Pipelines

### Pipeline 1: Reverse Generation — Code → Specs

Reads existing source files (components, routes, API handlers, existing tests) and generates structured Living Spec Markdown files for features that don't yet have a spec, or updates stale specs when code has changed.

**When it runs:** After a coding agent completes a plan phase. After any significant code change. Manually any time.

```bash
specguard reverse --app my-app
specguard reverse --app my-app --force   # overwrite existing specs
specguard reverse --file src/pages/checkout.tsx  # single file
```

**Config-driven discovery** — no hard-coded file lists. The `.specguard/config.json` tells SpecGuard where your routes, pages, API handlers, and existing tests live for each app in your monorepo (or single repo).

**Output:** Structured Markdown files in `specs/<area>/<feature>.md`.

---

### Pipeline 2: Forward Generation — Specs → Tests

Reads Living Spec files and generates test stubs (or full tests) for the configured test framework. Uses Claude agents to explore the live app and produce concrete, selector-verified test code.

**Framework support:** Playwright (default), Jest, Vitest, Cypress, or any custom template.

```bash
specguard generate --spec auth/login
specguard generate --all
specguard generate --spec auth/login --framework vitest
```

**Output:** Test files in the location configured per framework (e.g. `tests/auth/login.spec.ts`).

---

### Pipeline 3: Validation — Live App + Spec → Issue Reports

Navigates to the live application, captures screenshots, HTML, and the ARIA accessibility tree, then sends all of it to an LLM with the spec for structured issue analysis.

```bash
specguard validate --spec auth/login --url http://localhost:3000
specguard validate --all --url http://localhost:3000
specguard validate --all --url https://staging.example.com --out reports/validation.json
```

**Issue types:**

| Type | Description |
|---|---|
| `functional` | Page doesn't match the spec's Visual Expectations |
| `ux` | Violates UX Guidelines (loading states, error specificity, focus) |
| `accessibility` | Fails ARIA, WCAG 2.2 AA, or keyboard navigation requirements |
| `visual` | Layout breaks, missing content, rendering problems |

**Severity:** `critical` / `major` / `minor`. Exits non-zero when critical or major issues are found — safe for CI gates.

---

### Pipeline 4: Security Analysis — Spec + Code → Security Tests

Cross-references spec scenarios and source code against OWASP patterns to generate targeted security test stubs. Covers authentication boundaries, input validation, authorization checks, and data exposure.

```bash
specguard security --spec auth/login
specguard security --all
```

**Output:** Security test stubs annotated with OWASP category and severity, placed alongside your regular test files (or in a dedicated `tests/security/` directory).

**Checks include:**
- Auth bypass scenarios derived from spec `auth:` metadata
- Input validation boundary tests from spec scenarios
- Privilege escalation paths from role-based spec sections
- Sensitive data exposure in API responses
- CSRF and XSS surface area from form-based scenarios

---

### Pipeline 5: Doc Generation — Specs → User Documentation

Strips test-specific content (scenarios, seed references, Zephyr keys) and transforms Living Specs into clean user-facing documentation pages.

```bash
specguard docs --spec auth/login
specguard docs --all --out docs/user
```

**Output:** Markdown files with frontmatter compatible with VitePress, Docusaurus, or any static site generator.

---

### Pipeline 6: Drift Detection — Code Changes → Spec Staleness

Compares recent code changes (git diff) against the specs that describe that code. Flags specs that are likely stale and surfaces which scenarios may no longer be accurate.

```bash
specguard drift                          # check HEAD against last commit
specguard drift --since main             # check branch diff against main
specguard drift --spec auth/login        # check one spec
```

**Output:** A drift report listing stale specs, the code lines that triggered the flag, and suggested spec sections to review.

---

## The Spec Format

Each spec is a structured Markdown file. The format is the same whether hand-written or generated by `reverse`:

```markdown
# Feature Title

<!--
  app: <repo or app name>
  url: <route path, e.g. /checkout>
  auth: <public | user | admin | <custom-scope>>
  framework: <playwright | jest | vitest | cypress>
  security: <enabled | disabled>
  docs: <enabled | disabled>
-->

## Overview
Short description of what this feature does and why it exists.
Cover all sub-states and flows (e.g. "new user" vs "returning user").

## Visual Expectations
- Bullet list of what the user should see
- Include headings, key UI elements, loading states, error states
- Note responsive behaviour where relevant

## Accessibility Requirements
- Specific ARIA roles and labels required
- Focus order expectations
- Keyboard navigation requirements
- WCAG 2.2 AA criteria relevant to this page

## Scenarios

### Scenario 1: <concise name>
**Auth:** public
**Precondition:** (if applicable)

**Steps:**
1. Navigate to /feature
2. Perform action
3. Observe result

**Expected Results:**
- What the user should see
- What state the app should be in

---

### Scenario 2: <concise name>
...

## Security Notes
- Known attack surfaces (e.g. "form accepts user-supplied HTML")
- Auth boundaries that must be enforced
- Data fields that must never be exposed to unauthorised users

## UX Guidelines
- Interaction quality expectations
- Loading state behaviour
- Error message specificity requirements
```

The `<!-- -->` metadata block is the machine-readable layer. All pipelines read it to know which app, URL, auth scope, and framework to use.

---

## Parallel Agent Mode

The core design principle: SpecGuard runs beside your coding agent, not after it.

```
Coding Agent                          SpecGuard Agent
─────────────────────────────         ──────────────────────────────────
1. Receive task
2. Plan phase 1                  →    [triggered] reverse-generate on
   (e.g. "add checkout page")          changed files → create/update
                                        specs/checkout/payment.md
3. Implement                     →    [triggered] generate tests from
                                        new spec → tests/checkout/
                                        payment.spec.ts
4. Plan phase 2                  →    [triggered] security scan →
   (e.g. "add auth middleware")         security/auth-middleware.spec.ts
5. PR ready                      →    [triggered] validate + drift
                                        check → report any regressions
```

### Trigger Points

The Cursor Skill hooks into these natural pause points in the coding agent's workflow:

| Trigger | Pipeline(s) Run |
|---|---|
| Plan phase completed | `reverse`, `generate` |
| Security-sensitive files changed (auth, middleware, API) | `security` |
| PR / branch ready | `drift`, `validate` |
| Spec file manually saved | `generate` (if `--watch` mode) |
| Manual invocation | Any pipeline |

### Manual Invocation

Natural language works via the skill:

```
Run specguard on the checkout feature
Validate the login page against the spec
Generate security tests for the auth middleware
Check for spec drift since main
```

---

## Installation

### Option 1: VS Code / Cursor Extension (recommended)

Install the **SpecGuard** extension from the VS Code marketplace or Cursor extension panel. It handles everything:

1. Installs the `specguard` CLI binary
2. Runs `specguard init` to scaffold `.specguard/config.json`
3. Registers `specguard-mcp` in your IDE's MCP config
4. Adds the Cursor skill to `.cursor/skills/specguard/SKILL.md`

### Option 2: Manual Installation

**Install the CLI:**

```bash
npm install -g specguard
# or
brew install specguard
```

**Scaffold config in your repo:**

```bash
cd your-project
specguard init
```

This creates:

```
your-project/
  .specguard/
    config.json        # app definitions, framework settings, output paths
  specs/
    README.md          # spec directory documentation
  .cursor/
    skills/
      specguard/
        SKILL.md       # parallel agent skill
    mcp.json           # updated with specguard-mcp entry
```

**Register the MCP server** (if not using the extension):

Add to `.cursor/mcp.json`:

```json
{
  "mcpServers": {
    "specguard": {
      "command": "specguard-mcp",
      "args": ["--workspace", "${workspaceFolder}"]
    }
  }
}
```

---

## Configuration

`.specguard/config.json`:

```json
{
  "apps": [
    {
      "name": "my-app",
      "repo": ".",
      "specDir": "specs/my-app",
      "sources": {
        "routes": ["src/App.tsx", "src/router.ts"],
        "pages": ["src/pages/**/*.tsx"],
        "api": ["src/api/**/*.ts", "app/api/**/*.ts"],
        "tests": ["tests/**/*.spec.ts", "e2e/**/*.spec.ts"]
      },
      "framework": "playwright",
      "testOutput": "tests/",
      "baseUrl": "http://localhost:3000",
      "security": true,
      "docs": true
    }
  ],
  "llm": {
    "provider": "anthropic",
    "model": "claude-sonnet-4-6",
    "apiKeyEnv": "ANTHROPIC_API_KEY"
  },
  "triggers": {
    "onPlanPhase": true,
    "onSecurityFiles": ["**/auth/**", "**/middleware/**", "**/api/**"],
    "onPR": true
  }
}
```

### Multi-repo / Monorepo

```json
{
  "apps": [
    {
      "name": "frontend",
      "repo": "packages/web",
      "specDir": "specs/web",
      "sources": { "pages": ["src/pages/**/*.tsx"] },
      "framework": "playwright",
      "baseUrl": "http://localhost:3000"
    },
    {
      "name": "api",
      "repo": "packages/api",
      "specDir": "specs/api",
      "sources": { "api": ["src/routes/**/*.ts"] },
      "framework": "jest",
      "testOutput": "packages/api/tests/"
    }
  ]
}
```

---

## Spec Coverage

```bash
specguard status
```

Output:

```
SpecGuard Status — my-app
──────────────────────────────────────────────────────
  Pages discovered:   24
  Specs written:      18  (75%)
  Tests generated:    16  (89% of specs with tests)
  Security tests:      9  (50% of specs with security)
  Docs generated:     14  (78% of specs with docs)

  Stale specs (drift detected):
    ⚠  auth/login          — src/pages/login.tsx changed 2d ago
    ⚠  checkout/payment    — src/api/stripe.ts changed 4h ago

  Missing specs:
    ✗  admin/reports
    ✗  settings/billing
    ✗  onboarding/invite
──────────────────────────────────────────────────────
  Run `specguard reverse --app my-app` to fill gaps.
```

---

## CI Integration

Add to your CI pipeline to gate PRs on spec health:

```yaml
# .github/workflows/specguard.yml
name: SpecGuard
on: [pull_request]

jobs:
  specguard:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm install -g specguard
      - name: Check for spec drift
        run: specguard drift --since origin/main
      - name: Validate against staging
        run: specguard validate --all --url ${{ vars.STAGING_URL }}
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

Exit codes follow the [PrintingPress typed-exit convention](https://github.com/mvanhorn/cli-printing-press):

| Code | Meaning |
|---|---|
| 0 | All checks passed |
| 1 | Internal error |
| 2 | Validation failed (critical/major issues found) |
| 3 | Drift detected (specs are stale) |
| 4 | Missing specs (uncovered features) |
| 5 | Security issues found |

---

## Relationship to Practera Test Suite

SpecGuard generalises the approach proven in [practera-test-suite](https://github.com/your-org/practera-test-suite):

| Practera Implementation | SpecGuard |
|---|---|
| `packages/spec-tools/src/spec-parser.ts` | Core spec parser (same format) |
| `packages/spec-tools/src/validate.ts` | `specguard validate` |
| `packages/spec-tools/src/reverse-generate.ts` | `specguard reverse` |
| `packages/spec-tools/src/doc-generate.ts` | `specguard docs` |
| `.claude/agents/playwright-test-planner.md` | `specguard generate` (Playwright mode) |
| `.claude/agents/playwright-test-generator.md` | `specguard generate` (code output) |
| Hard-coded `APP_CONFIGS` in reverse-generate.ts | `.specguard/config.json` |
| Practera-specific `SPECS_DIR` paths | Config-driven `specDir` per app |
| No security pipeline | `specguard security` |
| No drift detection | `specguard drift` |
| Manual CLI only | Parallel agent via Cursor Skill |

The spec format is **fully compatible**. Any `specs/` directory from the Practera test suite works with SpecGuard out of the box.

---

## Roadmap

- [ ] CLI binary (TypeScript, ships as npm package)
- [ ] MCP server (`specguard-mcp`)
- [ ] Cursor Skill (`SKILL.md`)
- [ ] `specguard init` scaffolding
- [ ] Pipeline 1: Reverse Generation
- [ ] Pipeline 2: Forward Generation (Playwright)
- [ ] Pipeline 3: Validation
- [ ] Pipeline 4: Security Analysis
- [ ] Pipeline 5: Doc Generation
- [ ] Pipeline 6: Drift Detection
- [ ] `specguard status` coverage report
- [ ] VS Code / Cursor Extension
- [ ] Forward Generation: Jest / Vitest support
- [ ] Forward Generation: Cypress support
- [ ] GitHub Actions integration
- [ ] Web dashboard for spec coverage (port of practera-test-suite web package)
- [ ] Zephyr Scale / Jira integration for traceability

---

## Contributing

This project is in active development. The core pipelines are extracted from production use in the Practera test suite and are being generalised here. PRs welcome — especially framework adapters for test generation beyond Playwright.

---

## License

MIT
