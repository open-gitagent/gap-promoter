---
name: open-pr
description: "Open a clean, reviewable pull request against a GitHub repo using the gh CLI — fork if needed, branch, commit only the added files, push, and gh pr create with a concise body. Triggers on: open a PR, raise a pull request, propose changes upstream."
allowed-tools: "Bash, Read, Write"
---

# Open a pull request (gh CLI)

Propose the GAP files to the target repo as a PR. `gh` is authenticated from
`$GH_TOKEN`. Everything here is a proposal — never force-push, never merge.

## 0. Pre-flight: the eligibility gate must have returned PROCEED

If you reached this skill without running `gap-eligibility` and getting a
**PROCEED**, stop and run the gate now. If it returned DISCUSS or DECLINE, do
NOT open a PR. This skill assumes the gate already cleared the repo.

## 0b. Check if a GAP PR already exists (idempotency — do NOT re-open)

Check for an existing GAP PR from **anyone**, not just this bot's fork. The old
check only looked at our own `gitagent-protocol` branch and missed PRs opened by
other contributors (or our own under a different branch). Check all three:

```bash
ME=$(gh api user --jq .login)

# (a) Our fixed-branch PR
gh pr list --repo <owner>/<repo> --state all \
  --head "$ME:gitagent-protocol" --json url,state,title,mergedAt --limit 5

# (b) ANY PR (any author / branch) whose title smells like a GAP adoption PR
gh pr list --repo <owner>/<repo> --state all --limit 100 \
  --json url,state,title,headRefName | python3 -c "import json,sys; \
[print(p['state'], p['url'], p['title']) for p in json.load(sys.stdin) \
 if any(k in (p['title'] or '').lower() for k in ['gitagent','agent.yaml','soul.md','gap support','gap manifest'])]" 2>/dev/null

# (c) Search PRs by title across all states
gh search prs --repo <owner>/<repo> "GitAgent Protocol" --state all --limit 10 2>/dev/null || true
```

Decide based on state (a hit in (b) or (c) counts the same as our own PR):
- **open** → A GAP PR is already waiting on the maintainer. **Do NOT create a
  new one.** If it's ours and our locally-generated files differ, push the
  updates to the same `gitagent-protocol` branch on our fork — GitHub
  auto-updates the open PR. If it's someone else's, report it and stop.
- **merged** → GAP files are already upstream. Skip this skill and go straight
  to the `submit-to-registry` step. Report the merged PR URL for context.
- **closed (not merged)** → The maintainer declined or moved on (ours or
  anyone's). **Do NOT reopen or open a fresh one.** Report the closed URL and
  stop; the user can re-engage manually if they want.
- **none** (no existing GAP PR anywhere) → proceed with the steps below.

This check is mandatory — one target repo, one GAP PR, ever, from anyone.

## 1. Confirm auth

```bash
gh auth status 2>&1 | head -3   # should show the token's account
```

## 2. Fork (if you don't have write access) and clone your fork

```bash
gh repo fork <owner>/<repo> --clone=false --remote=false 2>&1 || true
ME=$(gh api user --jq .login)
git clone "https://x-access-token:$GITHUB_TOKEN@github.com/$ME/<repo>.git" /tmp/fork
cd /tmp/fork
git remote add upstream "https://github.com/<owner>/<repo>.git" 2>/dev/null || true
```

If you DO have write access to the upstream, skip the fork and branch directly on it.

## 3. Branch, add ONLY the new files, commit

```bash
cd /tmp/fork
git checkout -b gitagent-protocol
cp /tmp/target/agent.yaml ./agent.yaml
cp /tmp/target/SOUL.md ./SOUL.md
git add agent.yaml SOUL.md
git -c user.email="bot@gitagent.sh" -c user.name="GAP Promoter" commit -m "Add GitAgent Protocol manifest (agent.yaml + SOUL.md)"
git push -u origin gitagent-protocol
```

## 4. Open the PR

```bash
gh pr create --repo <owner>/<repo> \
  --head "$ME:gitagent-protocol" \
  --title "Add GitAgent Protocol support (agent.yaml + SOUL.md)" \
  --body "$(cat <<'EOF'
Hi! This PR adds **GitAgent Protocol** support to your agent — a small, open
standard for portable AI agents (https://gitagent.sh).

**What this adds (nothing else changes):**
- `agent.yaml` — a standard manifest (name, version, model, runtime)
- `SOUL.md` — your agent's persona in the standard format

With these two files, your agent can run on any GAP-compatible runtime (Claude
Code, GitClaw, and others) and be listed in the open registry. Totally optional
to accept — feel free to tweak or close. Thanks for building in the open! 🦀

---

**What is GAP?** An open, vendor-neutral standard for portable AI agents — one
manifest, run anywhere. If this looks useful, the project lives at
**⭐ https://github.com/open-gitagent/opengap** — a star helps more maintainers
discover it.
EOF
)"
```

Capture and report the PR URL (the `gh pr create` output is the URL).

## Rules
- **Idempotency: one target repo → one GAP PR, ever.** Always run Step 0
  before anything else. Open / closed / merged — never duplicate. If the
  existing branch needs an update, push to the same `gitagent-protocol`
  branch on the fork (GitHub auto-updates the open PR).
- Branch name is **always** `gitagent-protocol` (never randomise / suffix).
- Add only the GAP files. Never modify or delete existing files.
- Never `git push --force`; never `gh pr merge`.
- Short, warm PR body with a footer linking to **https://github.com/open-gitagent/opengap**
  asking for a star (one line, not pushy). Link gitagent.sh too.
- Report the PR URL verbatim.
