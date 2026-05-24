# gap-promoter

A [GitAgent Protocol](https://gitagent.sh) agent that **promotes the protocol** —
it brings existing AI-agent repos into the Open GAP ecosystem.

Point it at any public agent repo and it will:

1. **Convert** the repo to the GitAgent Protocol format (`agent.yaml` + `SOUL.md`),
   faithfully distilled from the repo's existing prompt, README, and model.
2. **Open a PR** against that repo adding the GAP files — a clean, optional,
   reviewable proposal (never a force-push or self-merge).
3. **Submit it to the registry** ([open-gitagent/registry](https://github.com/open-gitagent/registry))
   with a second PR adding `agents/<owner>__<repo>/metadata.json` + `README.md`.

It reports both PR URLs when done.

## Layout

- `agent.yaml` — GAP manifest (claude-code adapter)
- `SOUL.md` — the promoter's persona + workflow
- `skills/gap-convert` — convert a repo to GAP format
- `skills/open-pr` — open a reviewable PR via the `gh` CLI
- `skills/submit-to-registry` — add the agent to the Open GAP registry
- `knowledge/gap-spec.md` — the GAP format reference
- `knowledge/registry-format.md` — the registry submission schema

## Requirements

A GitHub token in the environment (`GITHUB_TOKEN` / `GH_TOKEN`) with `repo`
scope and permission to fork + open PRs. Everything the agent does is a
proposal — it never merges or force-pushes.
