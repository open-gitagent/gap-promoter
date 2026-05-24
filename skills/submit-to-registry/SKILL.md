---
name: submit-to-registry
description: "Submit a GAP agent to the Open GAP registry (github.com/open-gitagent/registry) via a PR — add agents/<owner>__<name>/metadata.json + README.md per the schema, fork, branch, push, gh pr create. Triggers on: list in the registry, submit to registry, publish my agent, register the agent."
allowed-tools: "Bash, Read, Write"
---

# Submit an agent to the Open GAP registry

List a GAP agent on https://registry.gitagent.sh by opening a PR against
`open-gitagent/registry`. See `knowledge/registry-format.md` for the full schema.

**Precondition:** the target repo must be **public** and already have a valid
`agent.yaml` + `SOUL.md` on its default branch (the registry CI clones it and
checks). So run `gap-convert` + `open-pr` first, and only submit to the registry
once those files are merged (or already present) on the target's default branch.

## 1. Fork + clone the registry

```bash
gh repo fork open-gitagent/registry --clone=false 2>&1 || true
ME=$(gh api user --jq .login)
git clone "https://x-access-token:$GITHUB_TOKEN@github.com/$ME/registry.git" /tmp/registry
cd /tmp/registry && git checkout -b add-<owner>-<repo>
```

## 2. Create the agent folder + metadata.json

Folder name uses a **double underscore**: `agents/<repo-owner-github-username>__<agent-name>/`.

```bash
DIR="agents/<owner>__<name>"
mkdir -p "$DIR"
cat > "$DIR/metadata.json" <<'EOF'
{
  "name": "<agent-name>",
  "author": "<repo-owner-github-username>",
  "description": "<= 200 chars, what the agent does",
  "repository": "https://github.com/<owner>/<repo>",
  "version": "1.0.0",
  "category": "<one allowed category>",
  "tags": ["<up to 10 tags>"],
  "license": "MIT",
  "model": "claude-sonnet-4-5-20250929",
  "adapters": ["claude-code", "system-prompt"],
  "icon": false
}
EOF
```

**Allowed `category`:** `developer-tools`, `data-engineering`, `devops`,
`compliance`, `security`, `documentation`, `testing`, `research`, `productivity`,
`finance`, `customer-support`, `creative`, `education`, `other`.

**`adapters`:** the runtimes it supports, e.g. `claude-code`, `openai`, `lyzr`,
`system-prompt`.

## 3. README.md (required, non-empty)

```bash
cat > "$DIR/README.md" <<'EOF'
# <Agent Name>

<What it does, key capabilities, example usage.>
EOF
```

## 4. Validate locally if tooling is present, then PR

```bash
cd /tmp/registry
npm ci 2>/dev/null && npx tsx scripts/validate.ts 2>&1 | tail -20 || echo "(skipping local validate; CI will run it)"
git add "$DIR"
git -c user.email="bot@gitagent.sh" -c user.name="GAP Promoter" commit -m "Add <owner>/<name> to the registry"
git push -u origin "add-<owner>-<repo>"
gh pr create --repo open-gitagent/registry --head "$ME:add-<owner>-<repo>" \
  --title "Add <owner>/<name>" \
  --body "Adds **<name>** by <owner> to the registry. Repo: https://github.com/<owner>/<repo> — has a valid agent.yaml + SOUL.md. Category: <category>."
```

Report the registry PR URL.

## Rules
- Folder name MUST match `<author>__<name>` exactly (double underscore), or CI fails.
- `metadata.json` must validate against the schema (required fields, valid category, <=10 tags, <=200-char description).
- README.md must exist and be non-empty.
- Target repo must be public with `agent.yaml` + `SOUL.md` on its default branch.
- Open a PR; never merge it. Report the URL.
