# Playwright AI Framework — MCP + CLI + Claude Code

An end-to-end test automation template that combines **deterministic
Playwright execution** with **token-light AI assistance**, split across four
cooperating agents — from reading product docs all the way to self-healing
scripts in CI:

| Agent | Interface | When it runs | Token cost |
|---|---|---|---|
| **Test Designer** | Claude Code (reads `requirements/`) | Once per feature, before scripting | Light |
| **Explorer** | Claude Code + Playwright **MCP** | Once, to automate reviewed test cases | Light (AXTree snapshots) |
| **Executor** | Playwright **CLI** | Every test run (local + CI) | Zero — pure code execution |
| **Healer** | Claude Code (Tier-2 fallback + follow-up agent) | Only when a locator breaks | Minimal (micro-DOM snippets), and only as a last resort |

```
0. requirements/*.md   ──▶  1. Test Designer (AI)  ──▶  test-cases/*.md
                                                              │
                                                  ★ HUMAN REVIEW GATE ★
                                                              │
                                                              ▼
                             2. Explorer (MCP, AXTree)  ──▶  tests/*.spec.ts
                                                              │
                                                              ▼
                             3. Executor (CLI, 0 tokens)  ──▶  CI results
                                                              │ locator fails
                                                              ▼
                             4. Tier-1 Fuzzy Heal (0 tokens)
                                                              │ unresolved
                                                              ▼
                             5. Tier-2 AI Heal (minimal tokens) ──▶  Healer promotes fix
```

The whole loop runs inside **Claude Code** (VS Code terminal or panel) —
not claude.ai in the browser — because every stage needs to read/write
files in this repo, run MCP, or execute bash. See
[`docs/07-TEST-CASE-GENERATION.md`](docs/07-TEST-CASE-GENERATION.md) for
the full workflow and [`docs/01-ARCHITECTURE.md`](docs/01-ARCHITECTURE.md)
for the design rationale.

**Process, not just tooling:** [`QA-BIBLE.md`](QA-BIBLE.md) defines the full
QA process this pipeline supports — strategy, roles, review gates,
environments, defect handling, metrics, and release sign-off criteria. Read
it once as a team; refer back to it as the process reference.

---

## What's included

- ✅ **QA Bible** (`QA-BIBLE.md`) — the full QA process: strategy, roles,
  review gates, environments, metrics, release sign-off
- ✅ **Test case generation from docs** — drop a PRD in `requirements/`,
  get structured, prioritized, traceable test cases in `test-cases/`
- ✅ Playwright + TypeScript project, pre-configured
- ✅ HTML, JUnit, and JSON **reports**; automatic **screenshots** and **video**
  on failure; full **trace viewer** support
- ✅ **Parallel execution** across Chromium/Firefox/WebKit + mobile emulation,
  plus CI **sharding**
- ✅ **API testing** template (separate fast project, no browser overhead)
- ✅ **Self-healing locators**: Tier-1 deterministic fuzzy match (0 tokens),
  Tier-2 AI micro-context fallback (minimal tokens)
- ✅ **MCP config** + exploration prompt for one-time, token-light AI test
  generation
- ✅ **3 Claude Code subagents** (Explorer / Executor / Healer) + slash
  commands
- ✅ **CI/CD pipeline** (GitHub Actions) with parallel matrix + sharding +
  artifact upload
- ✅ `.env.example`, `.gitignore`, secrets handling conventions

---

> **New to this project / never used Playwright before?** Start at
> [`docs/00-GETTING-STARTED.md`](docs/00-GETTING-STARTED.md) instead — it
> assumes zero prior knowledge and walks through installing Node, opening
> this in VS Code, and running your first test.

## Quick start

```bash
# 1. Install
npm install
npx playwright install --with-deps

# 2. Configure environment
cp .env.example .env.local
# edit .env.local with your BASE_URL, test user creds, ANTHROPIC_API_KEY, etc.

# 3. Run the example suite
npm test

# 4. View the HTML report
npm run report
```

## Generate test cases from a requirements doc (Test Designer agent)

```bash
cp my-prd.md requirements/checkout.md
claude
# inside the Claude Code session:
/design-tests checkout.md
```

This reads the doc and writes structured, prioritized test cases to
`test-cases/checkout.md` (plus updating `test-cases/_traceability-matrix.md`).
**Review this file before automating it** — see the review gate in
[`QA-BIBLE.md`](QA-BIBLE.md#5-review-gates-non-negotiable-checkpoints).

## Generate a new test with AI (Explorer agent)

Requires [Claude Code](https://docs.claude.com/en/docs/claude-code) installed
and the bundled `.mcp.json` (already configured for Playwright MCP).

Two ways to drive this:

```bash
# Preferred: automate reviewed test cases (traceable, recommended)
/generate-scripts CHK-001,CHK-003,CHK-004 checkout.md

# Free-form: no test case yet, quick prototyping
/explore checkout /checkout "Add an item to cart and complete guest checkout"
```

Either drives the Explorer subagent through the flow using accessibility-tree
snapshots only, then writes `tests/ui/checkout.spec.ts` using semantic
locators, and disconnects MCP. See
[`docs/02-MCP-EXPLORATION.md`](docs/02-MCP-EXPLORATION.md) and
[`docs/07-TEST-CASE-GENERATION.md`](docs/07-TEST-CASE-GENERATION.md).

## Run tests (Executor agent / plain CLI)

You don't need AI for this step at all — it's just Playwright:

```bash
npm test                    # full suite, all browsers, parallel
npm run test:ui             # UI tests only
npm run test:api            # API tests only
npm run test:chromium       # single browser
npm run test:parallel       # explicit 4 workers
npm run test:shard1         # CI-style sharding (1 of 2)
```

Or ask Claude Code: `/run-tests checkout` — same commands, agent just
summarizes the output for you. See
[`docs/04-CLI-USAGE.md`](docs/04-CLI-USAGE.md).

## When a locator breaks (Healer)

Nothing to do manually — it happens automatically inside the run. Tier-1
fuzzy matching (0 tokens) tries first; only if that fails does Tier-2 send a
small HTML snippet to a cheap model (`claude-haiku-5-5`) for a one-line fix.
After a run, check what got healed and promote it into your source file:

```bash
npm run healed:review
# then, in Claude Code:
/heal-apply
```

Full details: [`docs/03-SELF-HEALING.md`](docs/03-SELF-HEALING.md).

## API testing

Separate, browser-less project (`--project=api`) for fast, cheap API
coverage. Template at `tests/api/example-users.api.spec.ts`, client wrapper
at `utils/api-client.ts`. See [`docs/06-API-TESTING.md`](docs/06-API-TESTING.md).

## CI/CD

`.github/workflows/playwright.yml` runs UI tests across 3 browsers × 2 shards
in parallel, plus a separate API job, uploads HTML reports and failure
artifacts (video/screenshot/trace), and surfaces any self-healing that
occurred. Adapt the secrets block for your environment (GitLab CI / Jenkins /
CircleCI equivalents follow the same shape — see
[`docs/04-CLI-USAGE.md`](docs/04-CLI-USAGE.md#cicd-notes)).

---

## Project structure

```
.
├── QA-BIBLE.md               # ★ full QA process definition — read this
├── .claude/
│   ├── agents/               # 4 subagents: test-designer, explorer, executor, healer
│   └── commands/              # /design-tests, /generate-scripts, /explore, /run-tests, /heal-apply
├── .github/workflows/        # CI/CD pipeline
├── docs/                     # Full guide set (start at 00-GETTING-STARTED.md)
├── requirements/              # INPUT: PRDs, user stories, API specs (you write these)
├── test-cases/                # OUTPUT: structured test cases + traceability matrix (AI-drafted, human-reviewed)
├── fixtures/
│   └── healing-fixture.ts    # injects heal() into `test`
├── mcp/
│   ├── mcp-config.json       # Playwright MCP server config
│   └── exploration-prompt.md
├── scripts/                  # healed-locators review/clear CLI helpers
├── tests/
│   ├── ui/                   # example-login.spec.ts
│   └── api/                  # example-users.api.spec.ts
├── utils/
│   ├── healing/              # fuzzy-match.ts, healer.ts, healed-locators.json
│   ├── api-client.ts
│   └── logger.ts
├── .env.example
├── .gitignore
├── .mcp.json                  # auto-loaded by Claude Code
├── playwright.config.ts
└── package.json
```

## Docs index

0. [Getting started — zero prior knowledge assumed](docs/00-GETTING-STARTED.md)
1. [Architecture & token-optimization rationale](docs/01-ARCHITECTURE.md)
2. [MCP exploration in depth](docs/02-MCP-EXPLORATION.md)
3. [Self-healing (Tier-1 / Tier-2)](docs/03-SELF-HEALING.md)
4. [CLI usage, parallelism, sharding, CI/CD notes](docs/04-CLI-USAGE.md)
5. [Claude Code agents & slash commands](docs/05-CLAUDE-CODE-AGENTS.md)
6. [API testing guide](docs/06-API-TESTING.md)
7. [Requirements → test cases → scripts workflow](docs/07-TEST-CASE-GENERATION.md)

And [`QA-BIBLE.md`](QA-BIBLE.md) for the process (not tooling) side of all
of this — strategy, roles, review gates, metrics, release sign-off.
