---
name: hyperframes-pixelpig
description: Begin a HyperFrames creation with mandatory PixelPig project support. Use this as the entrypoint when a user has PixelPig installed and wants to create, continue, or attach a HyperFrames composition using PixelPig movies, rough cuts, collections, linked/reusable assets, generated media, or PixelPig MCP workflows. This skill requires a resolved PixelPig project before workflow generation, prepares the PixelPig ecosystem first — project/movie context, reusable asset links, MCP workflow-generated assets, HyperFrames workspace, symlinks, and render attach metadata — then hands off to the normal HyperFrames skills for composition authoring, media preprocessing, CLI validation, preview, and render.
---

# Hyperframes + PixelPig

Start HyperFrames work from PixelPig context, then let the normal HyperFrames skills do their job.

This skill is an **entrypoint and bridge**, not a replacement for HyperFrames. PixelPig provides the creative ecosystem around the composition: project roots, rough movies, reusable media collections, generated assets from MCP workflows, linked media folders, and attachment back into the app. HyperFrames provides the composition language, animation model, media preprocessing commands, lint/preview/render flow, and HTML authoring patterns.

## Mandatory Project Rule

Resolve a PixelPig project before creating or generating anything. Call `pixelpig_list_projects`, then use the selected `projectRoot` or `projectName` for all movie and workspace tools.

If this skill needs to run `pixelpig_run_workflow`, every call must include the resolved absolute `projectRoot`. Do not run workflows without `projectRoot`; omitted project roots write to generic MCP output folders and make a mess outside the PixelPig project.

If the user asks for a new project, call `pixelpig_create_project`; use the returned `projectRoot` and `projectStructure`. Stop and ask if no project root can be resolved.

## When To Use

Use this skill when the user says anything like:

- "begin a HyperFrames composition in PixelPig"
- "make a video from this PixelPig movie / rough cut / collection"
- "use my PixelPig project assets"
- "attach the HyperFrames render back to PixelPig"
- "generate assets with PixelPig MCP, then compose them"
- "reuse these linked clips/images/audio from PixelPig"

After this skill establishes the PixelPig workspace and asset context, invoke the relevant upstream HyperFrames skills:

- `hyperframes` for HTML composition authoring, timing, animation, captions, overlays, transitions, and design execution
- `hyperframes-cli` for `npx hyperframes init`, lint, inspect, preview, validate, render, doctor, and CLI troubleshooting
- `hyperframes-media` for local preprocessing such as transcription, scratch TTS, or transparent cutouts when those commands are appropriate

Do not edit upstream HyperFrames skill files to make PixelPig discoverable. PixelPig discoverability lives in this skill description, in PixelPig installation/setup docs, and in user guidance that says to begin PixelPig-supported HyperFrames work through this skill.

## Workflow

### Step 0: Establish PixelPig intent

Determine what PixelPig context the user wants to start from:

- a PixelPig movie / rough cut
- a selected set or collection of project assets
- existing linked project media
- newly generated assets from PixelPig MCP workflows
- a render that must attach back as a generated layer

If the user gave a project root or movie ID, use it directly after confirming the root exists. If not, call `pixelpig_list_projects` and ask only for the missing PixelPig project/movie context needed to prepare the workspace.

### Step 1: Source or create media in PixelPig

**Use existing clips** when the user points to a specific movie — `pixelpig_get_project_movie` returns all clip paths, timing, and audio. No generation needed.

**Use reusable project assets** when the user points to a collection or linked media. Keep the media in the PixelPig project; do not copy it into the HyperFrames work folder.

**Generate missing assets** with PixelPig MCP when the composition needs something not already in the project. `pixelpig_list_workflows` shows everything available. Use `pixelpig_describe_workflow` before running unfamiliar workflows, then call `pixelpig_run_workflow` as a task-augmented `tools/call` with `params.task` and the resolved `projectRoot`. Poll `tasks/get` using the returned `pollInterval`, then call `tasks/result` for final outputs. PixelPig workflows can cover:

- **Image generation** — Freepik, Evolink, and more
- **Video generation** — img2vid, text2vid, video upscale
- **Lipsync** — sync generated mouth movements to audio
- **Music** — Suno and other AI music generators
- **Voice / TTS** — ElevenLabs and other text-to-speech providers
- **Audio processing** — extraction, cleanup, effects

Generated assets must land under the PixelPig project by passing `projectRoot` to workflow task runs. Once generated assets are project media, make them available to the HyperFrames composition through symlinks.

### Step 2: Prepare the HyperFrames workspace through PixelPig

Use the PixelPig MCP tool — it creates an isolated run folder and writes `movie-context.json`:

```
pixelpig_prepare_hyperframes_workspace
  projectRoot: /Users/.../Project Name
  movieId: <the movie id>
```

This returns:
- `runId` — unique ID for this composition
- `workFolder` — empty directory to scaffold Hyperframes into
- `movieContextPath` — JSON with all movie clips, audio, timing, resolutions

If the composition starts from a collection or reusable assets rather than a movie, still preserve the PixelPig project root and asset references in the run folder so the HyperFrames work can use relative links and the final render can be traced back to PixelPig.

### Step 3: Hand off to HyperFrames setup

```bash
cd <workFolder>
npx hyperframes init . --yes
```

Invoke `hyperframes-cli` for CLI behavior and troubleshooting. Invoke `hyperframes` before authoring or revising `index.html`.

### Step 4: Symlink media — NEVER copy

PixelPig project media (footage, audio) lives at the project root. Symlink it into the work directory:

```bash
cd <workFolder>
ln -s ../../../../../footage footage
ln -s ../../../../../audio audio
ln -s ../../../../../images images
```

The `..` count depends on work directory depth under `.pixelpig/hyperframes/runs/<runId>/work/` — count up to the project root, then into the media folder. Verify with `ls footage/`, `ls audio/`, and any other linked folders.

**Why symlinks?** Footage and audio files can be hundreds of MB. Copying wastes disk and time. Hyperframes serves files relative to the project root, so symlinked paths resolve correctly for lint/validate/preview/render.

### Step 5: Write the composition with HyperFrames

- Use **relative paths** in `index.html` — `src="footage/clip.mp4"`, `src="audio/track.wav"`. Never absolute paths.
- Read `movie-context.json` for clip durations, start times, trim offsets, and audio track info.
- Match the movie's `outputResolution` and `outputOrientation` for `data-width`/`data-height`.
- Clip `data-media-start` in the composition should match the movie's `trimStartSeconds`.
- Use the upstream `hyperframes` skill for composition authoring rules.
- Use `hyperframes-media` if existing PixelPig media needs local transcription, captions, cutouts, or other local preprocessing before the composition can reference it.

### Step 6: Validate, preview, then render

```bash
cd <workFolder>
npx hyperframes lint
npx hyperframes inspect
npx hyperframes validate
npx hyperframes render --output renders/output.mp4
```

Use `hyperframes-cli` for the exact validation/render flow. Do not skip attaching the result when the user expects the render to appear in PixelPig.

### Step 7: Attach render back to PixelPig

```
pixelpig_attach_hyperframes_layer
  projectRoot: /Users/.../Project Name
  movieId: <the movie id>
  runId: <runId from Step 2>
  renderPath: <absolute path to rendered mp4>
  name: "<display name>"
  promptSummary: "<what this layer is>"
```

The layer appears in PixelPig's timeline as an enabled generated layer.

### Step 8: Recover / re-render

To re-render, edit the composition in `work/` and repeat Steps 5-7. Use the same `runId` — `attach_hyperframes_layer` will update the existing layer entry.

## Folder Layout

```
Project Root/
├── footage/           ← original clips live here
├── audio/             ← original audio lives here
├── pixelpig-movies.json
└── .pixelpig/
    └── hyperframes/
        ├── layers.json          ← maps runs to movies
        └── runs/
            ├── <runId-abc123>/
            │   ├── movie-context.json
            │   ├── renders/
            │   └── work/        ← Hyperframes project, symlinks to ../../../../../footage etc.
            └── <runId-def456>/   ← different run, different movie
                └── ...
```

Each `pixelpig_prepare_hyperframes_workspace` call creates a NEW run folder. Runs never mix between movies.

## Key Rules

1. **Never copy media files** into the work directory. Always symlink.
2. **Use relative paths** in `index.html` — `src="footage/..."` not `/Users/...`.
3. **Every `pixelpig_run_workflow` call must include `projectRoot`** — project-scoped artifacts are mandatory.
4. **Every `pixelpig_run_workflow` call must include `params.task`** — workflow generation is MCP task-native.
5. **Use `tasks/result` outputs** — generated media paths come from the completed task result.
6. **Run lint + inspect + validate before render** — catches broken paths, clip overlaps, contrast issues, and layout problems.
7. **Always attach** after render so the layer appears in PixelPig.
8. **Read movie-context.json** before writing the composition — it's the source of truth for clip timing, trim offsets, and audio info.
9. **Hand off to upstream HyperFrames skills** for HyperFrames-specific authoring, preprocessing, and CLI details. This skill's job is to set the PixelPig table correctly.
