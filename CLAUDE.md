# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

This is the **public releases and distribution** repo for Pixel Pig — a closed-source desktop app for AI media workflows (the app source lives elsewhere). There is **no build system, package manager, or test suite** here. The repo holds four kinds of artifacts that ship to users:

1. **The marketing website** — `index.html` plus image/icon assets, `sitemap.xml`, `robots.txt`. Deployed to https://www.pixelpig.net/. `index.html` is a single self-contained file (~1200 lines, inline CSS/JS); edit it directly.
2. **User docs** — `README.md` (GitHub landing page) and `MCP-SETUP.md` (the canonical MCP connection guide).
3. **Community model mappings** — `models/**/*.json` (see below).
4. **Agent skills** — `skills/*/SKILL.md`, installed by users via `npx skills add addr010/pixel-pig-releases --skill <name>`.

Because there's nothing to compile or test, "verifying" a change means re-reading the affected file (validate JSON, check links, view the HTML). Keep edits minimal and faithful to existing style.

## Model mappings (`models/`)

These JSON files let the app expose new provider models without an app update. Directory layout encodes provider and capability — preserve it exactly:

```
models/<provider>/<provider>_<capability>/<model-slug>.json
```

e.g. `models/replicate/replicate_image_to_video/google-veo-3-1-fast.json`. Providers seen here: `fal`, `replicate`, `kie`. Capabilities: `text_to_image`, `image_to_image`, `image_to_video`, etc.

Each file has shape `{ "models": [ { "id", "displayName", "parameters": { ... } } ] }`. Conventions that matter:

- `id` is the provider's real API model id (e.g. `google/veo-3.1-fast`, `fal-ai/hidream-i1-full`).
- Each entry under `parameters` maps an **app-facing key** (e.g. `imageAspect`, `durationSeconds`, `negativePrompt`, `seed`) to the provider's `apiParameter` name. App-facing keys are shared vocabulary across providers — reuse existing names (e.g. `imageAspect`, not `aspectRatio`; a past commit renamed exactly this) so the app's UI binds correctly.
- `apiValueType` (`int`, `bool`, `boolean`, `string`) controls serialization; `default` sets the preselected value; `options[]` lists user choices, where `value`/`display` are app-side and optional `apiValue` is what's sent to the provider.
- `hidden: true` keeps a parameter out of the UI but still sends it (used for `seed`, `negativePrompt`, fixed safety flags, etc.).

Some models are intentionally **not** here — they're built into the app instead (a commit moved Grok built-in "to handle id's for spicy mode correctly"). Don't re-add those as mappings.

## Skills (`skills/`)

Two agent skills, each a single `SKILL.md` with YAML frontmatter (`name`, `description`):

- `pixelpig-cinema-director` — the primary skill; drives the Pixel Pig MCP server to generate cinematic assets, character references, video shots, and rough movies. Enforces a strict project-first rule: every `pixelpig_run_workflow` call must pass an explicit `projectRoot` resolved via `pixelpig_list_projects`/`pixelpig_create_project`.
- `hyperframes-pixelpig` — bridges a Pixel Pig movie into a HyperFrames render and attaches the result back. Depends on separately-installed HyperFrames skills.

Both skills depend on the Pixel Pig MCP server. When editing skills, keep them consistent with `MCP-SETUP.md` — it is the single source of truth for the connection details below.

## Pixel Pig MCP (reference)

Local MCP server bundled with the app; the desktop app starts/stops it automatically.

- HTTP endpoint: `http://127.0.0.1:7361/mcp` (health: `/health`). Port override: `PIXELPIG_MCP_PORT`.
- `stdio` launch (when the app isn't running): `PixelPig.McpServer --stdio` (macOS path: `/Applications/PixelPig.app/Contents/Resources/app/PixelPig.McpServer`).
- Workflow runs are async: `pixelpig_run_workflow` returns a `runId` immediately; poll `pixelpig_get_workflow_run` until outputs complete (images ~15–240s, video ≥1 min per 5s of output).

If you change any of these facts, update `MCP-SETUP.md`, `README.md`, and both `SKILL.md` files together — they restate the same setup steps and must stay in sync.
