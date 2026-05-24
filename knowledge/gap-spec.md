# GitAgent Protocol (GAP) — format reference (v0.1.0)

A GAP agent is a public repo with, at minimum, `agent.yaml` + `SOUL.md` at the
root. Reference: https://gitagent.sh

## agent.yaml

```yaml
spec_version: "0.1.0"          # optional
name: my-agent                 # REQUIRED, non-empty, lowercase-hyphen
version: 1.0.0                 # REQUIRED, non-empty (semver)
description: >                 # optional, recommended
  One paragraph on what the agent does.
author: github-username        # optional
license: MIT                   # optional, SPDX id

model:                         # optional
  preferred: anthropic:claude-sonnet-4-6   # <provider>:<model>
  fallback:
    - anthropic:claude-opus-4-7
  constraints:
    temperature: 0.3
    max_tokens: 8192

skills:                        # optional — names map to skills/<name>/SKILL.md
  - skill-one
  - skill-two

runtime:                       # optional
  max_turns: 50                # positive int
  timeout: 600                 # seconds
  budget_usd: 5                # optional

compliance:                    # optional but encouraged
  risk_tier: standard
  supervision:
    human_in_the_loop: none    # ENUM — only: always | destructive | none
    kill_switch: true
  recordkeeping:
    audit_logging: true
  data_governance:
    pii_handling: redact
```

**Hard validation rules** (manifests that break these are rejected by the
harness and registry CI):
- `name` and `version` are required and non-empty.
- `compliance.supervision.human_in_the_loop` must be exactly one of
  `always`, `destructive`, `none`. (A common mistake is `conditional` — invalid.)
- `runtime.max_turns` / `timeout` / `budget_usd` are positive numbers.
- Unknown extra fields are allowed (schema is permissive / passthrough).

## SOUL.md

Plain Markdown holding the agent's persona and instructions — who it is, how it
behaves, its constraints and style. This is the system prompt in standard form.

## Optional, richer structure

`skills/<name>/SKILL.md` (capabilities), `knowledge/` (reference docs),
`templates/`, `RULES.md`, `AGENTS.md`. None required for protocol conformance —
only `agent.yaml` + `SOUL.md` are.

## Adapters

A GAP agent runs on any compatible runtime via an adapter: `claude-code`
(Claude Agent SDK), `gitagent`/`gitclaw`, `deepagents` (LangGraph), and a
generic `system-prompt` adapter. List the ones it supports in the registry entry.
