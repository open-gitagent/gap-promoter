---
name: gap-eligibility
description: "MANDATORY gate that runs BEFORE any conversion or PR. Decides whether a target repo is actually a fit for the GitAgent Protocol — checks for existing identity docs anywhere in the tree, whether the repo is a portable agent vs a full system, the contribution process (Discussion-first), and whether a GAP PR already exists. Returns PROCEED / DISCUSS / DECLINE. Triggers on: should I convert this, is this repo a fit, before opening a PR."
allowed-tools: "Bash, Read"
---

# Eligibility gate — run this FIRST, every time

You do **not** open a PR against a repo just because you can. Most repos are not
a fit, and a misjudged PR wastes a maintainer's time and burns the protocol's
goodwill. This gate runs before `gap-convert` and `open-pr`. It returns one of:

- **PROCEED** — clean fit, no existing identity, portable, PRs welcome → convert + PR.
- **DISCUSS** — plausibly interesting but the repo's process wants a conversation
  first, OR it's a borderline fit → open a GitHub **Discussion** (or Issue), not a PR.
- **DECLINE** — not a fit. Report why and stop. Do **not** open anything.

Run all checks. **Any single DECLINE signal → DECLINE.** A DISCUSS signal with no
DECLINE → DISCUSS. Only an all-clear → PROCEED.

```bash
git clone --depth 1 "https://x-access-token:$GITHUB_TOKEN@github.com/<owner>/<repo>.git" /tmp/target 2>&1 | tail -1
cd /tmp/target
```

## Check 1 — Does an existing GAP PR already exist? (broad, not just our branch)

The old check only looked at our own fork's `gitagent-protocol` branch. Broaden it:
look for **any** open or past PR that adds GAP files, from anyone, on any branch.

```bash
# (a) Our own fixed-branch PR (open / closed / merged)
ME=$(gh api user --jq .login)
gh pr list --repo <owner>/<repo> --state all --head "$ME:gitagent-protocol" \
  --json url,state,title --limit 5

# (b) ANY PR (any author, any branch) that touches agent.yaml or SOUL.md
gh pr list --repo <owner>/<repo> --state all --limit 100 \
  --json number,url,state,title,headRefName | \
  python3 -c "import json,sys; [print(p['state'], p['url'], p['title']) for p in json.load(sys.stdin)]" 2>/dev/null

# (c) Search PRs/issues mentioning the protocol by title
gh search prs --repo <owner>/<repo> "GitAgent Protocol" --state all --limit 10 2>/dev/null || true
gh search prs --repo <owner>/<repo> "agent.yaml" --state all --limit 10 2>/dev/null || true
```

**Decision:**
- An OPEN GAP PR (ours or anyone's) → if it's ours and our files differ, push to
  the same branch to update it; otherwise **DECLINE** (someone's already on it).
- A CLOSED-not-merged GAP PR (ours or anyone's) → **DECLINE.** The maintainer
  already saw this proposal and passed. Never re-pitch.
- A MERGED GAP PR → files are upstream; skip conversion, go straight to registry.
- Nothing found → continue to the next checks.

## Check 2 — Does the repo already have agent identity / persona docs?

Don't grep only the repo root. Identity docs commonly live in `src/`, `.claude/`,
`docs/`, `prompts/`, `config/`, etc. If the repo already manages its agent's
identity somewhere, a repo-root `SOUL.md` would create **drift** (the repo reads
one file at runtime, shows another externally). That's a DECLINE.

```bash
# Identity / persona files anywhere in the tree (not just root)
find . -path ./.git -prune -o -type f \
  \( -iname "SOUL.md" -o -iname "CLAUDE.md" -o -iname "identity*.md" \
     -o -iname "persona*.md" -o -iname "system*prompt*" -o -iname ".cursorrules" \
     -o -iname "AGENTS.md" -o -iname "agent.yaml" -o -iname "agent.yml" \) -print 2>/dev/null

# Runtime references to a soul/identity file (proves it's loaded at runtime)
grep -rinE "soul\.md|identity/|system_prompt|persona" --include=*.{ts,js,py,go,rs,md} . 2>/dev/null | head -20
```

**Decision:**
- A root-level `agent.yaml` + `SOUL.md` already present & valid → skip conversion,
  go straight to registry (this is the existing "already GAP" path).
- Identity docs exist **elsewhere** (e.g. `src/<proj>/identity/SOUL.md`) and are
  referenced at runtime → **DECLINE.** Adding a root `SOUL.md` would duplicate /
  drift from their real source of truth. Say exactly that in your report.
- No identity management at all → continue.

## Check 3 — Is this a PORTABLE agent, or a full system?

GAP is for **portable** agents you can drop into a runtime. It is NOT for
applications that happen to use an LLM but need dedicated infrastructure
(databases, daemons, message bridges, deploy pipelines). Adding an `agent.yaml`
to those implies a plug-and-play portability that doesn't exist — and maintainers
(rightly) reject it.

Signals it is a **full system, not a portable agent** (any strong hit → DECLINE):

```bash
# Infra / deployment footprint
ls -d docker-compose* Dockerfile* k8s/ helm/ terraform/ systemd/ .github/workflows/ 2>/dev/null
grep -rilE "qdrant|pinecone|weaviate|postgres|sqlite|redis|kafka|rabbitmq|systemd|supervisord|telegram|slack bridge|webhook server" \
  --include=*.{md,yml,yaml,toml,py,ts} . 2>/dev/null | head -20
# Long-running services / daemons
grep -rilE "while true|app\.listen|uvicorn|gunicorn|fastapi|express\(\)|createServer|\.serve\(\)" \
  --include=*.{py,ts,js,go} . 2>/dev/null | head -10
```

**Decision:**
- Strong infra footprint (its own DB + daemon + bridges + deploy config, i.e. it
  needs dedicated infrastructure to run) → **DECLINE.** It's a system, not a
  droppable agent. A manifest would misrepresent it.
- A small prompt/skill repo, a single-file agent, a Claude Code project, a
  LangChain agent script → portable → continue.

## Check 4 — What does the repo's contribution process require?

```bash
for f in CONTRIBUTING.md CONTRIBUTING .github/CONTRIBUTING.md docs/CONTRIBUTING.md; do
  [ -f "$f" ] && { echo "=== $f ==="; cat "$f"; }
done 2>/dev/null
# Does the project use Discussions?
gh api repos/<owner>/<repo> --jq '.has_discussions' 2>/dev/null
```

**Decision:**
- CONTRIBUTING says "open an Issue/Discussion before a PR" (for features / new
  standards / dependencies) → **DISCUSS.** Open a Discussion (or Issue if
  Discussions are off) proposing GAP adoption and asking if it's of interest.
  Do **not** open a PR. Respect the stated process — this is the single most
  common reason a well-meaning PR gets closed.
- No such requirement / PRs explicitly welcome → continue.

## Check 5 — Is the repo even an agent, and is it active?

```bash
# Archived / abandoned?
gh api repos/<owner>/<repo> --jq '{archived, disabled, pushed_at, stargazers_count}' 2>/dev/null
# Is there actually an agent persona/prompt to convert?
grep -rilE "you are|system prompt|persona|assistant|agent" --include=*.md . 2>/dev/null | head -5
```

**Decision:**
- Archived / disabled → **DECLINE.**
- No discernible agent prompt/persona (it's a library, a dataset, a CLI tool with
  no LLM persona) → **DECLINE.** There's nothing to put in a SOUL.
- Otherwise → continue.

## Output of the gate

End with an explicit verdict line and the reason, e.g.:

```
ELIGIBILITY: DECLINE — Genesis manages its runtime identity at
src/genesis/identity/SOUL.md (loaded into sessions); a root SOUL.md would drift.
It's also a full system (Qdrant + SQLite + systemd + Telegram bridge), not a
portable agent, and CONTRIBUTING.md requires a Discussion before feature PRs.
No PR opened.
```

or

```
ELIGIBILITY: DISCUSS — CONTRIBUTING.md asks for a Discussion before standards-
adoption PRs. Opening a Discussion proposing GAP rather than a PR.
```

or

```
ELIGIBILITY: PROCEED — portable single-prompt agent, no existing identity docs,
no prior GAP PR, PRs welcome. Continuing to gap-convert + open-pr.
```

## Rules
- This gate is **mandatory** and runs **before** `gap-convert` / `open-pr`.
- **Any DECLINE signal wins.** Don't rationalize around a clear no.
- When in doubt between PROCEED and DISCUSS, choose **DISCUSS** — a Discussion is
  cheap and respectful; a rejected PR is not.
- Never open a PR against a repo that returned DECLINE or DISCUSS.
- Report the verdict + reasons to the user verbatim, even on PROCEED.
