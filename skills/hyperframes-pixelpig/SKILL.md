---
name: hyperframes-pixelpig
description: "Bridge Pixel Pig projects and movies into verified HyperFrames compositions and attach finished renders through Pixel Pig MCP. Use when a user wants to create, continue, render, deliver, or attach a HyperFrames composition using a Pixel Pig movie, rough cut, project media, collection, or newly generated asset."
---

# HyperFrames Pixel Pig

Use Pixel Pig as the project and movie spine. Use HyperFrames as the composition and render engine.

This skill prepares Pixel Pig context and preserves the final attachment path. It does not replace the normal HyperFrames entrypoint or its authoring, media, validation, preview, and render skills.

## Connect And Verify

Use the setup guide at <https://github.com/addr010/pixel-pig-releases/blob/main/MCP-SETUP.md> when Pixel Pig tools are unavailable. Configure the client directly when permitted.

- Connect to `http://127.0.0.1:7361/mcp` while Pixel Pig is open.
- For stdio on macOS, launch `/Applications/PixelPig.app/Contents/Resources/app/PixelPig.McpServer --stdio`.
- For stdio on Windows, launch the bundled `PixelPig.McpServer.exe --stdio`.
- Verify with `pixelpig_get_mcp_status` or `pixelpig_list_projects`, not a raw HTTP probe.

## Load The Owning Skills

Use the mandatory `hyperframes` entrypoint for routing and project lifecycle. Load its domain skills only when the task needs them:

- `hyperframes-core` for composition structure, timing, media ownership, and deterministic rendering.
- `hyperframes-animation` for motion, transitions, and runtime adapters.
- `hyperframes-creative` for design, typography, narration, and beat planning.
- `media-use` for images, icons, audio, captions, grades, cutouts, and other media preparation.
- `hyperframes-cli` for scaffold, lint, check, preview, render, upgrade, and diagnostics.
- `pixelpig-cinema-director` when cinematic assets or a rough Pixel Pig movie must be created first.

Do not use the obsolete `hyperframes-media` name. Do not use deprecated `inspect` or `validate` commands in new work; `hyperframes check` is the final gate.

## Resolve The Pixel Pig Context

Resolve a real project before creating or generating anything:

1. Call `pixelpig_list_projects`.
2. Resolve the user's selected, named, current, or supplied absolute `projectRoot`.
3. Call `pixelpig_create_project` only when the user asks for a new project.
4. Call `pixelpig_list_project_movies` and `pixelpig_get_project_movie` when starting from a movie.
5. Stop and ask if the required project or movie cannot be resolved.

Treat the returned movie as source of truth for clip paths, trims, playback rates, audio, duration, orientation, output resolution, and `updatedUtc`. Do not invent transcript, caption, stem, scene, or timing metadata.

## Generate Only Missing Media

Skip generation when the project already contains the required media. When generation is needed:

1. Call `pixelpig_list_workflows`.
2. Call `pixelpig_describe_workflow` for the selected workflow and model.
3. Pass the resolved absolute `projectRoot` and local inputs as `files[].path`.
4. Call `pixelpig_run_workflow` once.
5. Keep its `runId` and poll `pixelpig_get_workflow_run` until terminal state.
6. Use only completed output `filePath` values.

Use `pixelpig-cinema-director` for paid-run approval, recovery, retakes, reference locking, dialogue approval, and rough-movie safety. Never run a Pixel Pig workflow without `projectRoot`.

## Prepare The Workspace

Call:

```text
pixelpig_prepare_hyperframes_workspace
  projectRoot: /absolute/project/root
  movieId: <movie id>
  prompt: <concise creative brief>
  includeRoughMovie: true
```

Set `includeRoughMovie` only when the composition needs a reference cut. It is composition context, not a finished export.

Record the returned:

- `runId`
- `runFolder`
- `workFolder`
- `rendersFolder`
- `movieContextPath`
- `promptPath`
- `roughMoviePath`
- `messages`

Read `movie-context.json` before authoring. It includes the exact `projectStructure.mediaFolders`, movie settings, clip timing, trim ranges, audio automation, and source existence checks.

## Link Media Without Copying

Keep Pixel Pig media in its configured project folders. From `workFolder`, create relative symlinks to the exact folders returned in `movie-context.json`.

Typical links look like:

```bash
ln -s ../../../../../footage videos
ln -s ../../../../../audio audio
ln -s ../../../../../images images
```

Folder names are user-configurable, so derive every target from `projectStructure.mediaFolders`; do not assume the examples exist. Verify each link resolves before authoring.

Use relative composition paths such as `videos/clip.mp4` and `audio/music.wav`. Never copy large media into `workFolder`, and never write absolute user paths into composition HTML.

## Author Through HyperFrames

Treat `workFolder` as the HyperFrames project root and invoke the `hyperframes` entrypoint. Let it route fresh work, resume existing state, and choose the correct scaffold or workflow. Do not reconstruct current HyperFrames setup commands from memory.

During authoring:

- Match composition dimensions to the Pixel Pig movie output.
- Map source media starts and trims from `movie-context.json`.
- Preserve audio timing and gain unless the user requests a mix change.
- Keep generated or processed user-facing media in the configured Pixel Pig media folders.
- Keep HyperFrames run state and the canonical render under the returned run folder.

## Check, Preview, Approve, And Render

Use `hyperframes-cli` for exact command contracts.

During iteration, run:

```bash
npx hyperframes lint
```

For the final gate, preview, and delivery render:

```bash
npx hyperframes check
npx hyperframes preview
# Wait for the user's final preview approval.
npx hyperframes render --quality high --output ../renders/output.mp4
test -s ../renders/output.mp4
ffprobe -v error -show_format ../renders/output.mp4
```

Do not render merely because checks pass. Wait for final preview approval. Verify file existence, dimensions, duration, video codec, and audio stream.

Keep the canonical attached artifact in `rendersFolder`. Copy the verified final non-destructively into `projectStructure.mediaFolders.videos` with a clear delivery name when the user expects a normal project export. Never overwrite an existing delivery without explicit approval.

## Attach The Verified Render

Attach the canonical run render through Pixel Pig MCP:

```text
pixelpig_attach_hyperframes_layer
  projectRoot: /absolute/project/root
  movieId: <movie id>
  runId: <runId from workspace preparation>
  renderPath: <absolute canonical path under rendersFolder>
  name: <display name>
  promptSummary: <short factual summary>
  enabled: true
```

Confirm the returned layer ID, movie ID, run ID, enabled state, render path, and layers metadata path. Attaching a layer must not rewrite the movie's source clips or audio.

For a revision, keep the same run folder, update the composition, repeat check/preview/approval/render, and attach the newly verified artifact. Do not overwrite a prior user-facing delivery unless requested.

## Hard Rules

- Resolve the Pixel Pig project and movie before authoring.
- Pass `projectRoot` to every generation call.
- Use Pixel Pig MCP for Pixel Pig operations.
- Read `movie-context.json` before writing the composition.
- Symlink project media; never copy it into `workFolder`.
- Use relative media paths in HyperFrames files.
- Use current HyperFrames skills and `check`; avoid obsolete names and deprecated commands.
- Wait for final preview approval before rendering.
- Verify the canonical render before attachment.
- Preserve a separate non-destructive user-facing delivery when requested.
