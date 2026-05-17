---
name: "hyperframes-pixelpig"
description: "Use for HyperFrames renders from Pixel Pig movies that attach back through Pixel Pig MCP."
---

# HyperFrames Pixel Pig

Pixel Pig is the project and movie spine for this workflow. HyperFrames is the
composition and render engine.

Use Pixel Pig MCP for project lookup, movie metadata, generated media placement,
HyperFrames workspace creation, and final render attachment. Use the normal
HyperFrames skills for HTML composition, animation, validation, preview, and
rendering.

## Related Skills

- `hyperframes`: author `index.html`, timing, overlays, captions, transitions,
  and animation
- `hyperframes-cli`: run init, lint, inspect, validate, preview, and render
- `hyperframes-media`: preprocess local audio, captions, TTS, and cutouts
- `pixelpig-cinema-director`: generate cinematic assets or rough movies before
  the HyperFrames finish

## When To Use

Use this skill when the user asks to:

- Create a HyperFrames render from a Pixel Pig movie
- Turn a Pixel Pig rough cut into a finished HyperFrames composition
- Use Pixel Pig project media as HyperFrames source media
- Generate missing media in Pixel Pig, then compose it in HyperFrames
- Attach the final HyperFrames render back to Pixel Pig

Do not use this skill for general Pixel Pig generation unless the user also asks
for a HyperFrames composition.

## Workflow

### 1. Resolve Project And Movie

Use Pixel Pig MCP tools, not raw HTTP calls.

Required tools:

- `pixelpig_list_projects`
- `pixelpig_list_project_movies`
- `pixelpig_get_project_movie`

Resolve a real project and movie before writing files. If the user gives a
`projectRoot`, confirm it exists before using it. If the user asks for a new
movie, use `pixelpig_create_movie`.

Read the movie before authoring. Treat these fields as source of truth:

- Clip paths
- Trim ranges
- Durations
- Audio clips
- Orientation
- Output resolution

Do not invent scene, transcript, caption, or stem metadata that is not present.

### 2. Generate Missing Media

Skip this step if the movie already contains the needed clips and audio.

When generation is needed:

1. Call `pixelpig_list_workflows`.
2. Call `pixelpig_describe_workflow` for unfamiliar workflows or parameters.
3. Call `pixelpig_run_workflow` with the resolved absolute `projectRoot`.
4. Keep the returned `runId`.
5. Poll `pixelpig_get_workflow_run` until completed.
6. Use completed output paths from the run result.

Never run Pixel Pig workflows without `projectRoot`. Generated assets must land
inside the Pixel Pig project.

### 3. Prepare Workspace

Create the HyperFrames run folder through Pixel Pig MCP:

```text
pixelpig_prepare_hyperframes_workspace
  projectRoot: /absolute/project/root
  movieId: <movie id>
  prompt: <optional user brief>
  includeRoughMovie: <optional boolean>
```

Expected return values:

- `runId`
- `runFolder`
- `workFolder`
- `rendersFolder`
- `movieContextPath`
- `promptPath`
- `roughMoviePath`

The run folder is under:

```text
<projectRoot>/.pixelpig/hyperframes/runs/<runId>/
```

### 4. Initialize HyperFrames

Run this from `workFolder`:

```bash
npx hyperframes init . --yes
```

Then use the normal HyperFrames authoring and CLI skills before editing or
running the composition.

### 5. Link Media

Symlink Pixel Pig project media into `workFolder`. Do not copy source media.

Common layout:

```text
<projectRoot>/.pixelpig/hyperframes/runs/<runId>/work/
```

Common links from that folder:

```bash
ln -s ../../../../../footage footage
ln -s ../../../../../audio audio
ln -s ../../../../../images images
```

Adjust the `..` count if the folder depth differs. Verify links with `ls`.

Use relative media paths in HyperFrames files:

```html
<video src="footage/clip.mp4"></video>
<audio src="audio/music.wav"></audio>
```

Never use absolute `/Users/...` media paths in the composition.

### 6. Author And Validate

Read `movie-context.json` before editing `index.html`.

Match:

- `data-width` and `data-height` to the movie output resolution
- Media paths to symlinked project folders
- Media start and trim values to the movie context
- Audio timing and volume to the movie context unless the user asks to change it

Run from `workFolder`:

```bash
npx hyperframes lint
npx hyperframes inspect
npx hyperframes validate
npx hyperframes render --output renders/output.mp4
```

Use `hyperframes-cli` for command details and troubleshooting.

### 7. Attach Render

Attach the rendered output through Pixel Pig MCP:

```text
pixelpig_attach_hyperframes_layer
  projectRoot: /absolute/project/root
  movieId: <movie id>
  runId: <runId>
  renderPath: <absolute path to rendered file>
  name: <display name>
  promptSummary: <short summary>
```

This records a HyperFrames layer for the movie. It must not rewrite the movie
source clips or audio clips.

## Rules

- Always resolve a Pixel Pig project and movie first.
- Always pass `projectRoot` to Pixel Pig workflow runs.
- Always use Pixel Pig MCP tools for Pixel Pig verification.
- Never copy source media into the HyperFrames work folder.
- Never use absolute media paths in HyperFrames composition files.
- Always read `movie-context.json` before authoring.
- Always run lint, inspect, and validate before render.
- Always attach the final render with `pixelpig_attach_hyperframes_layer`.

## Output Checklist

- [ ] Project resolved through Pixel Pig MCP
- [ ] Movie resolved through Pixel Pig MCP
- [ ] Movie context read before authoring
- [ ] Any generated media used `projectRoot`
- [ ] Workspace created with `pixelpig_prepare_hyperframes_workspace`
- [ ] Project media symlinked into `workFolder`
- [ ] Composition uses relative media paths
- [ ] Lint, inspect, and validate completed
- [ ] Render attached with `pixelpig_attach_hyperframes_layer`
