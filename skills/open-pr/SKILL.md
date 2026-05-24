---
name: open-pr
description: "Open a clean, reviewable pull request against a GitHub repo using the gh CLI — fork if needed, branch, commit only the added files, push, and gh pr create with a concise body. Triggers on: open a PR, raise a pull request, propose changes upstream."
allowed-tools: "Bash, Read, Write"
---

# Open a pull request (gh CLI)

Propose the GAP files to the target repo as a PR. `gh` is authenticated from
`$GH_TOKEN`. Everything here is a proposal — never force-push, never merge.

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
EOF
)"
```

Capture and report the PR URL (the `gh pr create` output is the URL).

## Rules
- Add only the GAP files. Never modify or delete existing files.
- Never `git push --force`; never `gh pr merge`.
- Short, warm PR body. Link gitagent.sh.
- Report the PR URL verbatim.
