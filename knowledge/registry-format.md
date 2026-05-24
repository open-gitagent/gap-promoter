# Open GAP registry — submission format

Repo: https://github.com/open-gitagent/registry · Site: https://registry.gitagent.sh

To list an agent, open a PR adding a folder under `agents/`.

## Folder

```
agents/<author>__<name>/
```

- `<author>` = the agent repo owner's GitHub username.
- `<name>` = the agent name.
- Separator is a **double underscore** `__`. CI fails if the folder name doesn't
  match `<author>__<name>` from metadata.json.

## Required files

### metadata.json

```json
{
  "name": "my-agent",
  "author": "github-username",
  "description": "What the agent does (<= 200 chars)",
  "repository": "https://github.com/owner/repo",
  "path": "",
  "version": "1.0.0",
  "category": "developer-tools",
  "tags": ["tag1", "tag2"],
  "license": "MIT",
  "model": "claude-sonnet-4-5-20250929",
  "adapters": ["claude-code", "system-prompt"],
  "icon": false,
  "banner": false
}
```

| Field | Req | Notes |
|---|---|---|
| name | yes | lowercase, hyphens |
| author | yes | repo owner's GitHub username |
| description | yes | <= 200 chars |
| repository | yes | public GitHub URL |
| path | no | subdir within the repo (default root) |
| version | yes | semver |
| category | yes | one allowed value (below) |
| tags | yes | array, max 10 |
| license | yes | SPDX id |
| model | yes | preferred model id |
| adapters | yes | array, e.g. claude-code, openai, lyzr, system-prompt |
| icon | no | true if icon.png (256x256) included |
| banner | no | true if banner.png (1200x630) included |

**Categories:** developer-tools, data-engineering, devops, compliance, security,
documentation, testing, research, productivity, finance, customer-support,
creative, education, other.

### README.md

Non-empty Markdown describing the agent (shown on its detail page): what it does,
key capabilities, example usage, optional screenshots.

## CI checks (on the PR)

- metadata.json validates against `schema/metadata.schema.json`.
- Folder name matches `<author>__<name>`.
- README.md exists and is non-empty.
- The `repository` is cloned and checked for `agent.yaml` + `SOUL.md` (at `path`).

So the target repo must be **public** and already have those two files on its
default branch before the registry PR will pass CI.

## After merge

The agent appears on registry.gitagent.sh within minutes (the index is rebuilt
from all `agents/*/metadata.json`). To update an entry, open a new PR bumping
`version` and editing the folder.
