---
name: pixelpig-cinema-director
description: "Use Pixel Pig MCP for cinematic image and video generation, character references, scene plates, dialogue and TTS, voice cloning, dubbing, speech-to-text, lipsync, movie audio repair, timeline edits, rough cuts, renders, retakes, recovery, and HyperFrames handoffs inside PixelPig projects."
---

# PixelPig Cinema Director

Use PixelPig MCP as the production interface. Keep generated assets inside the resolved PixelPig project, preserve approved work, and treat the project movie as production state.

## Route The Task

Read only the references needed for the request:

- Read [references/image-generation.md](references/image-generation.md) for character bases, character sheets, scene plates, portraits, image edits, virtual try-on, reframing, and image upscale.
- Read [references/video-generation.md](references/video-generation.md) for Seedance shots, reference ordering, prompt grammar, lipsync video, remastering, subtitles, and video upscale.
- Read [references/dialogue-audio.md](references/dialogue-audio.md) for TTS, voice collections, retakes, dubbing, transcription/timestamps, duration fitting, loudness, music, sound effects, and dialogue review.
- Read [references/movie-editing.md](references/movie-editing.md) before any movie mutation, selected-media cut, audio swap, render, or HyperFrames handoff.

For a task spanning several domains, read each relevant reference before acting. Movie editing has the strictest state-safety rules.

## Connect And Verify

Use the setup guide at <https://github.com/addr010/pixel-pig-releases/blob/main/MCP-SETUP.md> when the tools are unavailable. Configure the client directly when permitted; do not make the user hand-edit config.

- Connect to `http://127.0.0.1:7361/mcp` while PixelPig is open.
- For stdio on macOS, launch `/Applications/PixelPig.app/Contents/Resources/app/PixelPig.McpServer --stdio`.
- For stdio on Windows, launch the bundled `PixelPig.McpServer.exe --stdio`.
- Verify with a real MCP call such as `pixelpig_get_mcp_status` or `pixelpig_list_projects`; raw HTTP health checks do not verify MCP exposure.

Inspect the connected tool schemas before movie work. Pixel Pig v0.32.0 lacks `pixelpig_backup_project_movie`, live-editor routing, and revision fields such as `expectedUpdatedUtc`; use the legacy closed-app path in `references/movie-editing.md` on that version. Use the current live-editor path only when its tools and fields are actually exposed.

Ask only when the installation cannot be found, a required client restart cannot be performed, or the environment blocks configuration.

## Resolve The Project

Pass an explicit `projectRoot` to every generation and movie tool.

1. Call `pixelpig_list_projects`.
2. Resolve the user's named project, selected project, current project, or supplied path.
3. Call `pixelpig_create_project` if the user asks for a new project.
4. Stop and ask if no real project root can be resolved.

Use collections only when the user explicitly points to an asset pack. Use `pixelpig_list_collections` and `pixelpig_get_collection` to resolve it.

## Discover Then Run

Before every workflow run:

1. Call `pixelpig_list_workflows`; only connected providers appear.
2. Call `pixelpig_describe_workflow` for the selected workflow. Include the model ID only when the connected tool schema exposes its optional `model` input.
3. Treat the descriptor as authoritative for exposed inputs. Do not invent parameters or hidden overrides.
4. Follow any parameter `contract` validation rules, format examples, mode requirements, and processing semantics.
5. If an advanced UI input is missing from the selected model's descriptor, report that contract gap instead of guessing.
6. Pass local inputs as `files[].path`; PixelPig handles provider upload and transport.
7. Include a short, stable, lowercase `parameters.filenamePrefix` where supported.
8. Call `pixelpig_run_workflow` once, keep its `runId`, and poll `pixelpig_get_workflow_run` to completion.

Use prefix families such as `<character>-character-base`, `<character>-character-sheet`, `<scene>-plate`, `<scene>-shot-01-take-01`, `<movie>-dialogue-02`, and `<movie>-score`. Do not put prompts or private notes in filenames.

## Confirm Paid Or Long Runs

Before a new paid or long generation, provide a concise preflight and wait for approval unless the user explicitly approved the batch:

```text
Pre-run check:
- Project: <name/root>
- Movie: <name or none>
- Asset: <type and quantity>
- Workflow: <workflowId> / <model>
- Inputs: <brief summary>
- Runtime/aspect: <when applicable>

Run it?
```

Skip a new confirmation only for a minor iteration already approved in the same task. For dialogue generation, approval to generate clips is not approval to place them in a movie.

## Recover Before Paying Again

Use these tools before rerunning an interrupted or partial workflow:

- `pixelpig_find_workflow_for_file` to recover provenance from an output path.
- `pixelpig_list_provider_tasks` and `pixelpig_recover_workflow_output` to reconcile provider tasks and download finished results.
- `pixelpig_retake_workflow_output` for one item from a prior text-prompt batch so take numbering stays traceable.

If a provider lacks funds or quota, tell the user and try another connected provider. Prefer `kie` where it supports the requested model, then `fal`, then `evolink`. If moderation blocks a compatible request, explain the model change and try a more permissive connected alternative such as Wan for video or Seedream for images.

## Find A Batch Or Project Fast

PixelPig filenames often include a five-character batch ID, for example `7d0sm`. Search workflow history for that exact ID before scanning the home directory.

1. Search `workflow-history` under the PixelPig app-data root: `~/Library/Application Support/PixelPig` on macOS or `%APPDATA%\PixelPig` on Windows; honor `PIXELPIG_APPDATA_ROOT`.
2. Read the matching NDJSON record's `summary.fileResults`, `inputFiles`, and project/output paths.
3. Report the project, source, and output paths.
4. Search project folders or backups only when history has no match.

On macOS:

```bash
rg -n -F '7d0sm' "${PIXELPIG_APPDATA_ROOT:-${HOME}/Library/Application Support/PixelPig}/workflow-history" 2>/dev/null
```

## Read Live Selection Exactly

When the user refers to files selected in the open PixelPig media browser, read the ordered selection from:

```bash
curl --fail --silent --show-error http://127.0.0.1:7362/selected
```

Treat the JSON array as exact. An empty array means nothing is selected; do not substitute the selected movie or guess from recent files. Use the bridge only for selection discovery, then use MCP for project resolution, workflows, movie edits, and renders.

## Use Vision Instead Of Guessing

If direct visual inspection is unavailable, find an existing vision text output sharing the source filename's short code before running another paid analysis. Otherwise use `fal-image-to-text` with a connected vision model for images or `fal-video-to-text` for video. Ask for visible subject, wardrobe, action, framing, camera motion, lighting, environment, props, readable text, audio-relevant events, and ending state. Treat output as notes, not unquestionable truth.

Do not infer identity, brands, age labels, backstory, or hidden traits. Mirror continuity-critical observations to the user before locking them into generation prompts.

## Keep Production Notes Compact

For movies with several assets, maintain factual project notes such as `<movie>-characters.md`, `<movie>-shots.md`, or `<movie>-continuity.md`. Record asset prefixes, selected movie IDs, takes, approvals, rejects, and continuity facts. Do not include chain-of-thought or private reasoning.

Use connected text-to-text workflows for creative ideation when helpful, but treat their output as suggestions and confirm changes to locked direction.

## Stop Conditions

- Stop on failed or partial workflows unless the user explicitly accepts partial output.
- Stop before generation when `projectRoot`, required inputs, or an authoritative workflow contract cannot be resolved.
- Stop before a movie mutation until the applicable movie-safety sequence is complete.
- Stop before placing generated dialogue until the user approves that take.
