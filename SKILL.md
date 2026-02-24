---
name: agent-readiness-scorecard
description: "Use this skill whenever the user wants to evaluate how 'agent-ready' a product, company, or GitHub repository is — i.e., how easily AI agents can discover, access, and use it autonomously. Triggers include: 'score this product for agents', 'is X agent-friendly', 'audit this repo/company for AI readiness', 'agent-readiness check', 'can agents use this?', or any request to evaluate a URL, GitHub repo, or company name against agent-accessibility criteria. Output is a scored report (0–100) with dimension breakdowns and actionable recommendations."
author: Pranay Tiwari
version: 1.1.0
---

# Agent-Readiness Scorecard

## Overview

The Agent-Readiness Scorecard evaluates how easily an AI agent can **discover**, **access**, **understand**, and **use** a product or service autonomously — without human intervention. Inspired by Andrej Karpathy's framework: *"It's 2026. Build for agents."*

A fully agent-ready product scores 100. Most products today score 10–30. A score above 70 means agents can use the product as a first-class tool in automated pipelines.

---

## Quick Reference

| Dimension | Max Score | Key Signal |
|-----------|-----------|------------|
| Documentation Accessibility | 20 | Markdown/plain-text docs exist and are machine-readable |
| CLI Availability | 20 | Product is operable via terminal commands, with env-var auth and safe sandbox mode |
| API / MCP Access | 20 | Structured programmatic interface with frictionless auth; penalizes browser OAuth |
| Agent Skills / Prompts | 15 | Pre-written instructions for agents to use the product |
| Discoverability | 15 | Agents can find and learn about the product autonomously |
| Output Parsability | 10 | Structured output with semantic error messages, not just happy-path JSON |

**Total: 100 points**

---

## Inputs

The scorecard accepts any of the following as input:

- A **company name** (e.g., `"Stripe"`, `"Linear"`)
- A **product URL** (e.g., `https://stripe.com`)
- A **GitHub repository URL** (e.g., `https://github.com/cli/cli`)
- A **CLI tool name** (e.g., `gh`, `polymarket-cli`)

---

## Scoring Methodology

### Step 1 — Gather Evidence

Before scoring, collect evidence across all six dimensions. Use available tools in this order:

```bash
# 1. Fetch the main URL or GitHub repo
curl -s https://<product-url> | head -200
curl -s https://api.github.com/repos/<owner>/<repo>

# 2. Check for CLI / npm / PyPI packages
npm info <package-name>
pip show <package-name>
which <cli-name>

# 3. Check for MCP server listings
curl -s https://raw.githubusercontent.com/<owner>/<repo>/main/mcp.json
# or search: site:github.com "mcp_servers" OR "model context protocol" <product-name>

# 4. Check docs format
curl -s https://docs.<product-url>/  # look for .md links, /llms.txt, /openapi.json
curl -s https://<product-url>/llms.txt  # emerging standard for agent-readable docs
curl -s https://<product-url>/openapi.json

# 5. Check for Skills/prompts
# Search GitHub for: <product-name> SKILL.md OR system-prompt OR agent-instructions
```

---

### Dimension 1 — Documentation Accessibility (0–20 pts)

Agents need to read and reason over docs. HTML rendered for humans is hard; markdown is easy.

| Evidence | Points |
|----------|--------|
| `/llms.txt` file exists at root (emerging agent-doc standard) | +8 |
| Docs are available as raw Markdown (e.g., GitHub wiki, `/docs/*.md`) | +6 |
| OpenAPI / Swagger spec exists (`/openapi.json`, `/swagger.yaml`) | +4 |
| Docs are PDF-only or purely rendered HTML with no raw export | 0 |
| No public documentation found | −5 (floor: 0) |

**Scoring notes:**
- Even if docs are on a website, check if the underlying source is a GitHub repo with `.md` files — that counts.
- A `/llms.txt` at the domain root is a strong positive signal (similar to `robots.txt` but for LLMs).

---

### Dimension 2 — CLI Availability (0–20 pts)

If a human can do it in a terminal, an agent can do it autonomously. Auth friction and the absence of a safe sandbox are the two most common agent-killers at this layer.

| Evidence | Points |
|----------|--------|
| Official CLI exists and is published (npm, pip, Homebrew, apt) | +10 |
| CLI has `--json` or machine-readable output flag | +4 |
| Auth is via environment variable (e.g., `export API_KEY=...`), no interactive prompt | +2 |
| CLI supports piping and stdin/stdout (composable with other tools) | +2 |
| `--dry-run` flag or sandbox mode available (safe for agentic experimentation) | +2 |
| No CLI exists but a community/unofficial one does | +4 |
| No CLI exists at all | 0 |

*(Max 20 pts — not all criteria are additive; e.g., official and community CLI are mutually exclusive)*

**Example check:**
```bash
gh --help                   # GitHub CLI — env var auth (GITHUB_TOKEN), --json flag
stripe --help               # Stripe CLI — env var auth, sandbox mode built-in
STRIPE_API_KEY=sk_test_...  # env var pattern = agent-friendly
stripe charges list --json  # structured output
```

---

### Dimension 3 — API / MCP Access (0–20 pts)

Structured programmatic access is the foundation of agent composability. Authentication friction is the single biggest silent killer of agent pipelines — an OAuth popup breaks the autonomous loop entirely.

| Evidence | Points |
|----------|--------|
| MCP (Model Context Protocol) server exists and is published | +10 |
| Official, well-typed SDKs available (Python and/or TypeScript) | +4 |
| REST API with simple auth (e.g., env-var API key, Bearer token) | +4 |
| API has rate limits documented and machine-readable | +2 |
| Requires interactive browser-based OAuth or CAPTCHA to authenticate | −5 |
| No API exists | 0 |

*(Max 20 pts before penalties — floor is 0)*

**MCP check:**
```bash
# Look for mcp.json, .cursor/mcp.json, or MCP listings at:
# https://github.com/modelcontextprotocol/servers
# https://mcp.so
curl -s https://raw.githubusercontent.com/<owner>/<repo>/main/.cursor/mcp.json
```

**Auth check:**
```bash
# Green flag — agent can inject key without human interaction:
export OPENAI_API_KEY=sk-...
curl -H "Authorization: Bearer $OPENAI_API_KEY" https://api.openai.com/v1/models

# Red flag — requires browser redirect or interactive login flow
# These completely break autonomous pipelines in Claude Code, Cursor, etc.
```

**Scoring notes:**
- MCP is the gold standard: it means the product was explicitly designed for agent use.
- SDKs with rich docstrings reward code-writing agents specifically — an agent can `pip install` and read the source to understand the interface without separate documentation.
- The browser OAuth penalty applies even if a token-based option *also* exists but isn't the default or isn't documented clearly.

---

### Dimension 4 — Agent Skills / Prompts (0–15 pts)

Skills are pre-written instructions that teach agents *how* to use a product. Without them, agents have to figure it out by trial and error.

| Evidence | Points |
|----------|--------|
| Official `SKILL.md` or equivalent agent instruction file exists in repo | +8 |
| System prompt / agent instructions published in docs | +5 |
| Example agent workflows or LangChain/Claude tool definitions published | +4 |
| Community-contributed prompts or agent templates exist | +3 |
| No agent-specific guidance exists | 0 |

**Check:**
```bash
# Look for these files in the repo root or /docs:
find . -name "SKILL.md" -o -name "agent-instructions.md" -o -name "system-prompt.txt"

# Or search GitHub:
# site:github.com <product-name> "SKILL.md"
```

---

### Dimension 5 — Discoverability (0–15 pts)

Can an agent *find* the product and understand what it does without human hand-holding?

| Evidence | Points |
|----------|--------|
| Listed in a public agent/MCP registry (mcp.so, npm, pip, Homebrew) | +5 |
| GitHub repo has clear README with installation instructions in < 3 steps | +4 |
| Product has a `/robots.txt` that allows crawling (not blocking agents) | +2 |
| README or docs include example agent prompts or use-case descriptions | +4 |
| Product is behind a login wall with no public discovery path | −5 (floor: 0) |

---

### Dimension 6 — Output Parsability (0–10 pts)

Agent pipelines need to parse and pass data downstream. This includes the *error path* — an agent that receives a silent failure or a generic "Bad Request" gets stuck with no self-correction path. Human-friendly ≠ machine-friendly.

| Evidence | Points |
|----------|--------|
| Default CLI/API output is structured JSON or YAML | +4 |
| Output schema is documented (OpenAPI spec, JSON Schema, or equivalent) | +3 |
| Error responses are semantic JSON with field-level detail (e.g., `{"error": "missing_param", "field": "user_id"}`) | +3 |
| Output is plain text (parseable but requires regex) | +1 |
| Output is HTML, rich terminal formatting, or proprietary format only | 0 |
| Output format is undocumented and inconsistent | 0 |

**Error handling check:**
```bash
# Good — agent can self-correct from this:
# {"error": "invalid_param", "field": "amount", "message": "must be > 0"}

# Bad — agent is stuck:
# HTTP 400 Bad Request (HTML body)
# "Something went wrong. Please try again."
# Silent 200 with empty response body
```

---

## Step 2 — Compile the Score

After gathering evidence, complete this scorecard:

```
AGENT-READINESS SCORECARD
=========================
Target: <product name or URL>
Evaluated: <date>

DIMENSION SCORES
----------------
Documentation Accessibility:  __ / 20
CLI Availability:              __ / 20
API / MCP Access:              __ / 20
Agent Skills / Prompts:        __ / 15
Discoverability:               __ / 15
Output Parsability:            __ / 10
                               --------
TOTAL:                         __ / 100

RATING
------
90–100  Agent-Native     ✦✦✦✦✦  Built for agents from day one
70–89   Agent-Ready      ✦✦✦✦   Agents can use this productively
50–69   Agent-Friendly   ✦✦✦    Usable with some effort
30–49   Agent-Awkward    ✦✦     Agents struggle; workarounds needed
0–29    Agent-Hostile    ✦      Effectively inaccessible to agents

EVIDENCE SUMMARY
----------------
[List the specific URLs, commands, and findings that support each score]

TOP 3 RECOMMENDATIONS
---------------------
1. [Highest-impact improvement with estimated point gain]
2. [Second priority]
3. [Third priority]
```

---

## Step 3 — Generate Recommendations

Always end the scorecard with actionable, prioritized recommendations. Use this framework:

**High impact (10+ points available):**
- Publish a CLI if none exists
- Add an MCP server
- Create a `/llms.txt` at the domain root
- Switch from browser OAuth to env-var API key auth (also removes the −5 penalty)

**Medium impact (5–9 points):**
- Export docs to Markdown or add a GitHub wiki
- Add `--json` flag and semantic JSON error messages to existing CLI
- Publish official Python/TypeScript SDKs with docstrings
- Write and publish a `SKILL.md`

**Low impact (1–4 points):**
- Add `--dry-run` flag to CLI for safe agentic experimentation
- Add example agent prompts to README
- List product in MCP registries
- Ensure `robots.txt` allows crawling

---

## Example Outputs

### GitHub CLI (`gh`) — Expected Score: ~80/100

```
Documentation Accessibility:  16 / 20  (markdown docs, OpenAPI partial)
CLI Availability:              19 / 20  (official CLI, --json, piping, GITHUB_TOKEN env var)
API / MCP Access:              10 / 20  (REST API with token auth, documented rate limits, no MCP, no SDK)
Agent Skills / Prompts:         4 / 15  (community prompts only)
Discoverability:               14 / 15  (Homebrew, clear README)
Output Parsability:             9 / 10  (structured JSON, documented schema, decent error messages)
                               --------
TOTAL:                         72 / 100  ✦✦✦✦ Agent-Ready
```

### Stripe — Expected Score: ~78/100

```
Documentation Accessibility:  18 / 20  (excellent markdown docs, full OpenAPI spec)
CLI Availability:              19 / 20  (official CLI, --json, env var auth, sandbox mode)
API / MCP Access:              18 / 20  (REST + MCP server, official Python/TS SDKs, rate limits; OAuth for some flows: -2)
Agent Skills / Prompts:         5 / 15  (some agent guidance, no SKILL.md)
Discoverability:               10 / 15  (good public docs, not in MCP registries)
Output Parsability:             9 / 10  (structured JSON, documented schema, semantic errors)
                               --------
TOTAL:                         79 / 100  ✦✦✦✦ Agent-Ready
```

### Hypothetical Legacy Enterprise SaaS — Expected Score: ~13/100

```
Documentation Accessibility:   4 / 20  (PDF docs behind login)
CLI Availability:               0 / 20  (no CLI)
API / MCP Access:               3 / 20  (REST API exists but requires browser SSO OAuth: +8 raw, -5 penalty)
Agent Skills / Prompts:         0 / 15  (none)
Discoverability:                5 / 15  (public website, no agent guidance)
Output Parsability:             1 / 10  (JSON responses but undocumented schema, generic error messages)
                               --------
TOTAL:                         13 / 100  ✦ Agent-Hostile
```

---

## Critical Rules

- **Always gather evidence before scoring** — never estimate without checking URLs, GitHub repos, npm/pip registries.
- **Be conservative** — only award points for evidence you can verify. Partial implementations get partial credit.
- **MCP > API > CLI > Docs** — this is the rough hierarchy of agent-readiness value. An MCP server alone can make an otherwise poor product agent-usable.
- **Auth friction is a dimension multiplier** — a browser OAuth requirement in Dimension 3 can negate an otherwise good API score. Always check the auth path explicitly.
- **Score the current state, not the roadmap** — announced but unshipped features score 0.
- **Test the error path, not just the happy path** — many products have clean JSON responses for success but return HTML error pages or empty bodies on failure. Check both.
- **Always include the Evidence Summary** — the score is only credible with citations.
- **Always end with 3 concrete recommendations** — the scorecard should be actionable, not just diagnostic.

---

## Dependencies

- `curl` — HTTP requests to check URLs, APIs, and raw files
- `gh` — GitHub CLI for repo inspection (optional but helpful)
- `npm` / `pip` — package registry checks
- Web search — for finding MCP registries, community tools, and docs
