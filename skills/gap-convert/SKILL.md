---
name: gap-convert
description: "Convert any AI-agent repo to the GitAgent Protocol format — author a valid agent.yaml + SOUL.md from the repo's existing prompt/README/model. Triggers on: convert to gitagent, make this GAP-compliant, add agent.yaml, onboard this repo to the protocol."
allowed-tools: "Bash, Read, Write"
---

# Convert a repo to the GitAgent Protocol

Turn an existing agent repo into a GAP-conformant one by adding two files at the
repo root: `agent.yaml` and `SOUL.md`. See `knowledge/gap-spec.md` for the full
field reference.

> **Pre-req: run `gap-eligibility` first.** Do not convert a repo until the gate
> returns **PROCEED**. If it returned DISCUSS or DECLINE, there's nothing to
> convert — open a Discussion or stop. In particular: if the repo already manages
> its agent identity somewhere (e.g. `src/<proj>/identity/SOUL.md` loaded at
> runtime), a root `SOUL.md` would create drift — that's a DECLINE, not a
> conversion.

## 1. Understand the repo

Clone and read it. Identify:
- **Purpose** — what the agent does (from README / prompt files).
- **Persona / instructions** — the system prompt (look for `SOUL.md`, `CLAUDE.md`,
  `system_prompt`, `prompt.md`, `.cursorrules`, README "you are…" text).
- **Model** — any model mentioned (default to `anthropic:claude-sonnet-4-6`).
- **Skills/tools** — distinct capabilities (optional to model as `skills:`).

```bash
git clone https://x-access-token:$GITHUB_TOKEN@github.com/<owner>/<repo>.git /tmp/target
ls -la /tmp/target; cat /tmp/target/README.md 2>/dev/null | head -80
```

## 2. Write `agent.yaml` (repo root)

Minimum valid manifest is `name` + `version`. Add what you can infer:

```yaml
spec_version: "0.1.0"
name: <repo-slug>                 # lowercase, hyphens
version: 1.0.0
description: >
  <one-paragraph summary of what the agent does>
license: MIT                      # match the repo's actual license
model:
  preferred: anthropic:claude-sonnet-4-6
runtime:
  max_turns: 50
compliance:                       # optional, but nice to include
  risk_tier: standard
  supervision:
    human_in_the_loop: none       # MUST be one of: always | destructive | none
```

**Validity rules (the registry/harness reject invalid manifests):**
- `name` and `version` are required, non-empty.
- `compliance.supervision.human_in_the_loop` ∈ `always | destructive | none` only.
- Keep model ids in `<provider>:<model>` form (e.g. `anthropic:claude-sonnet-4-6`).
- Extra fields are allowed (the schema passes them through) — don't omit useful ones.

## 3. Write `SOUL.md` (repo root)

Distill the repo's existing persona/instructions into a clean soul file: who the
agent is, how it behaves, its constraints. If the repo already has a strong
system prompt, adapt it; don't rewrite its intent. Keep it focused.

## 4. Verify

```bash
test -f /tmp/target/agent.yaml && test -f /tmp/target/SOUL.md && echo "GAP files present"
python3 -c "import yaml,sys; m=yaml.safe_load(open('/tmp/target/agent.yaml')); assert m.get('name') and m.get('version'); print('agent.yaml valid:', m['name'], m['version'])"
```

Then hand off to the `open-pr` skill to propose these files to the repo.

## Rules
- Faithful, not creative — represent what the agent actually does.
- Never modify existing files; only ADD `agent.yaml` + `SOUL.md`.
- Use valid enum values everywhere.
- If the repo already has both files and they're valid, skip — go straight to registry submission.
