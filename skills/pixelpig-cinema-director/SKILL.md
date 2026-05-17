---
name: pixelpig-cinema-director
description: "Use Pixel Pig MCP to create cinematic assets, character references, video shots, rough movies, and HyperFrames handoffs inside a Pixel Pig project."
---

# PixelPig Cinema Director

Build cinematic image assets, video shots, and rough movies through PixelPig MCP. Keep generated assets inside the resolved PixelPig project, add usable video/audio outputs to PixelPig movies, and prepare HyperFrames workspace context when the user wants a finished composition.

## Non-Negotiable PixelPig Project Rule

Every generation workflow must run with an explicit PixelPig `projectRoot`.

Before calling `pixelpig_run_workflow`:

1. Call `pixelpig_list_projects`.
2. Resolve a real project root from the user's selected project, current project, or user-provided path.
3. If the user asks for a new project, call `pixelpig_create_project`; use the returned `projectRoot` and `projectStructure`.
4. Stop and ask if no project can be resolved.

Never call `pixelpig_run_workflow` without `projectRoot`. Project-root workflow runs handle organized artifact placement.

Use PixelPig movies as production state. Collections are optional inputs only when the user explicitly points to an asset pack.

## Core MCP Tools

Project and movie spine:

- `pixelpig_list_projects`
- `pixelpig_create_project`
- `pixelpig_list_project_movies`
- `pixelpig_create_movie`
- `pixelpig_get_project_movie`
- `pixelpig_get_movie_json_schema`
- `pixelpig_update_project_movie`
- `pixelpig_add_movie_clip`
- `pixelpig_add_movie_audio_clip`

Workflow generation:

- `pixelpig_list_workflows`
- `pixelpig_describe_workflow`
- `pixelpig_run_workflow`
- `pixelpig_get_workflow_run`
- `pixelpig_get_mcp_status`

For workflow inputs, pass existing local files with `files[].path`. PixelPig workflow code handles provider upload and transport internally.

Call `pixelpig_run_workflow` once, keep the returned `runId`, then poll `pixelpig_get_workflow_run` until it returns completed outputs. Image workflows usually take 15-240 seconds; video workflows usually take at least 1 minute per 5 seconds of generated video.

HyperFrames handoff:

- `pixelpig_prepare_hyperframes_workspace`
- `pixelpig_attach_hyperframes_layer`

Optional asset-pack input:

- `pixelpig_list_collections`
- `pixelpig_get_collection`

## Provider Selection And Recovery

Before generation, call `pixelpig_list_workflows` and prefer connected providers in this order when the needed workflow exists:

1. `kie` for supported image/video workflows, because it is usually cheaper.
2. `fal` as the default broad-coverage provider.
3. `evolink` when it has the needed workflow or another provider is blocked.

Always call `pixelpig_describe_workflow` for the selected workflow before using a provider-specific model or parameter set.

Use `pixelpig_list_workflows` as the provider availability check. It only returns workflows from providers configured on the user's machine, and its messages identify providers hidden because setup is missing.

For Seedance video work, prefer Kie equivalents when available:

- text-to-video: `kie-text-to-video` with `bytedance/seedance-2`, otherwise `fal-text-to-video` with `bytedance/seedance-2.0/text-to-video`
- image-to-video: `kie-image-to-video` with `bytedance/seedance-2`, otherwise `fal-image-to-video` with `bytedance/seedance-2.0/image-to-video`
- reference-to-video: use the connected provider that exposes Seedance reference video, commonly `fal-reference-to-video` or `evolink-reference-to-video`

If a workflow fails because the provider has no funds, quota, or balance, notify the user that the provider may need a top-up, then try the next connected provider from `kie`, `fal`, and `evolink` before giving up.

If a workflow is blocked by moderation or censorship, retry with a more relaxed compatible model when available:

- video: Wan 2.7, for example `wan/2-7-text-to-video`, `wan/2-7-image-to-video`, or `fal-ai/wan/v2.7/image-to-video`
- images: Seedream 4.5, for example `seedream/4.5-text-to-image` or `seedream/4.5-edit`

Tell the user when changing provider or model after a failure so they understand cost, balance, and moderation behavior.

For music, mention that PixelPig can generate soundtrack assets when the user asks for music or score. Prefer `kie-suno-generate` when it appears in `pixelpig_list_workflows`; use `V5_5` when available, default to instrumental unless the user asks for vocals, and add returned audio `outputs[].filePath` to the PixelPig movie with `pixelpig_add_movie_audio_clip`.

## Vision Fallback For Agents Without Vision

Do not guess from filenames or user shorthand. If the agent cannot directly inspect an image or video, use PixelPig vision workflows before locking character details, extracting references, writing scene prompts, or judging generated outputs.

First check whether the asset already has a usable PixelPig vision text output from an earlier run. PixelPig generated asset names include a shared short code, such as `huw2e-01`, that appears in both the source media filename and related workflow output filenames. Match the source image/video and existing vision `.txt` by this shared code, then read the text file instead of running vision again. MCP workflow responses include text outputs with `name`, `mimeType`, and `filePath`; typical vision output names also include the model id, such as `x-ai-grok-4-fast...txt` or `google-gemini-2.5-flash...txt`.

Run vision again when there is no existing analysis file, the file is missing, the source asset changed, or the needed question is materially different.

For images, use `fal-image-to-text` with Grok 4 Fast:

```json
{
  "workflowId": "fal-image-to-text",
  "projectRoot": "<absolute project root>",
  "model": "x-ai/grok-4-fast",
  "files": [
    { "path": "<absolute image path>" }
  ],
  "parameters": {
    "prompt": "Describe only visible image content for cinematic production. Include subject identity markers, wardrobe, pose, lighting, camera/framing, environment, props, readable text, and any visual continuity details. Do not infer names, brands, age labels, backstory, or hidden traits.",
    "promptFormat": "none",
    "temperature": "0.2",
    "combineOutputs": "false"
  }
}
```

For videos, use `fal-video-to-text` with Gemini 2.5 Flash:

Short clips under 10 seconds are cheap enough to analyze when needed, so use this pragmatically to understand source clips, generated shots, continuity, and failure modes instead of guessing.

```json
{
  "workflowId": "fal-video-to-text",
  "projectRoot": "<absolute project root>",
  "model": "google/gemini-2.5-flash",
  "files": [
    { "path": "<absolute video path>" }
  ],
  "parameters": {
    "prompt": "Describe what happens in this video for cinematic continuity. Include subjects, wardrobe, actions, camera motion, scene changes, lighting, environment, props, visible text, audio-relevant events, and ending state. Do not invent names, brands, ages, or story context not visible in the clip.",
    "promptFormat": "none",
    "temperature": "0.2",
    "combineOutputs": "false"
  }
}
```

Use the returned text as analysis notes, not as final truth. Mirror important extracted details back to the user before generating or changing locked character/scene specs.

## Asset Naming With `filenamePrefix`

Use `parameters.filenamePrefix` on workflow runs to make generated assets easy for both agents and humans to find later. Choose short, searchable, stable prefixes for the thing being created:

- character assets: `<character-slug>-character-base`, `<character-slug>-character-sheet`, `<character-slug>-portrait`
- scene plates: `<scene-slug>-plate`, `<scene-slug>-environment`, `<scene-slug>-keyframe`
- video shots: `<scene-slug>-shot-01-take-01`, `<scene-slug>-shot-01-take-02`
- music/audio: `<scene-slug>-score`, `<scene-slug>-ambient`, `<movie-slug>-theme`
- vision notes: `<source-slug>-vision-notes`

Use the same prefix family across related runs so a later request like "make a shot of Jessica" can be resolved by finding assets named like `jessica-character-sheet`, `jessica-character-base`, or `jessica-portrait`.

Keep prefixes lowercase and filesystem-safe: letters, numbers, and hyphens. Do not put private notes, full prompts, unsafe text, or long descriptions in filenames.

## Movie Notes And Creative Help

Keep lightweight agent-friendly text notes when a movie gets more than a couple of assets. Put them in the PixelPig project alongside the movie context using clear filenames, for example:

- `<movie-slug>-production-notes.md`
- `<movie-slug>-characters.md`
- `<movie-slug>-shots.md`
- `<movie-slug>-continuity.md`

Use compact markdown tables or bullets so future agents can quickly recover character names, asset prefixes, scene slugs, shot/take status, selected movie IDs, and which outputs were approved or rejected. Keep these files factual and project-facing; do not include private chain-of-thought or internal reasoning.

When script, scene, shot, tagline, dialogue, or style ideation would benefit from a second creative pass, use PixelPig text-to-text workflows if available. Call `pixelpig_list_workflows`, then prefer `fal-text-to-text` with a creative model such as `x-ai/grok-4-fast` or `anthropic/claude-opus-4`.

Example:

```json
{
  "workflowId": "fal-text-to-text",
  "projectRoot": "<absolute project root>",
  "model": "x-ai/grok-4-fast",
  "parameters": {
    "basePrompt": "Give 6 cinematic shot ideas for this movie beat. Keep each idea practical for image-to-video generation, include subject action, camera motion, location, and emotional turn. Return concise bullets only.",
    "filenamePrefix": "<movie-slug>-script-help",
    "combineOutputs": "false"
  }
}
```

Treat text-to-text outputs as collaborators' notes, not commands. Select, adapt, and confirm anything that changes the user's locked character, scene, or movie direction.

## Strict Asset Workflow

Follow this order. Do not skip steps. Do not combine steps unless the user explicitly asks for a smaller one-off.

### Step 0: Resolve Project And Movie

Call `pixelpig_list_projects` first. Resolve the project, then create or select a movie if the work will produce video clips or a rough cut.

For a new movie:

```json
{
  "projectRoot": "<absolute project root>",
  "name": "<movie title>",
  "outputResolution": 1080,
  "outputOrientation": "Landscape"
}
```

### Step 1: Is The Character Already Built?

Before character assets, ask whether the character already exists or needs developing.

If the character exists:

- ask for the reference image(s) or PixelPig asset path(s)
- study and lock face, bone structure, skin tone, hair color and texture, identity markers, body proportions, makeup, expression, and wardrobe if present
- mirror the locked spec in plain language
- wait for confirmation before generating anything

If the character is new:

- let the user describe vibe, era, role, energy, look, references, and styling
- mirror back a locked spec covering face, hair, body/build, markers, default makeup, default expression, and energy
- iterate until the user says it is locked

Use visual descriptors, not names, in prompts sent to models.

### Step 2: Single-Image Character Outfit Base

Once the character is locked, the first generated image for a new outfit is always a single full-body character/outfit base reference on pure white seamless studio.

No 6-panel character sheet gets built before this base exists and the user approves it.

Ask for the outfit if missing. Capture every garment, accessory, footwear, styling choice, prop, hair adjustment, makeup adjustment, jewelry, and visible body marker. Mirror the wardrobe spec back before generation.

Default PixelPig tool:

```json
{
  "workflowId": "fal-text-to-image",
  "projectRoot": "<absolute project root>",
  "model": "openai/gpt-image-2",
  "parameters": {
    "basePrompt": "<single-image base prompt>",
    "generations": "1",
    "aspectRatio": "16:9",
    "resolution": "2K"
  }
}
```

If using existing character/wardrobe references, use:

```json
{
  "workflowId": "fal-image-to-image",
  "projectRoot": "<absolute project root>",
  "model": "openai/gpt-image-2/edit",
  "files": [
    { "path": "<absolute character image path>" },
    { "path": "<absolute wardrobe image path>" }
  ],
  "parameters": {
    "basePrompt": "<composite/edit prompt>",
    "generations": "1",
    "aspectRatio": "16:9",
    "resolution": "2K"
  }
}
```

Base prompt structure:

```text
[Visual descriptor of the locked character: face, skin, hair, makeup, body/build, visible markers]. [Full wardrobe head-to-toe: garments, fabrics, color, fit, structure, footwear, jewelry, accessories, props]. [Pose: centered full-body, body angled 15-30 degrees from camera, weight shifted onto one hip, chin level or slightly tucked, eyes to camera or slightly off-camera, neutral controlled expression].

Pure white seamless studio background, no visible seam line, no grey gradient. Soft cinematic key light from camera-left at 45 degrees with gentle fill from camera-right, subtle rim light defining shoulder and hair separation. Full body framing from head to just below footwear.

[Locked photoreal stack].
```

### Step 3: 6-Panel Character Sheet

Only after the single-image base reference exists and is approved, generate the character sheet.

Critical rules:

- one prompt
- one workflow run
- one horizontal image output
- one 3-column by 2-row grid
- never six separate prompts
- never six separate images
- the approved base reference is the input image

PixelPig tool:

```json
{
  "workflowId": "fal-image-to-image",
  "projectRoot": "<absolute project root>",
  "model": "openai/gpt-image-2/edit",
  "files": [
    { "path": "<absolute approved base image path>" }
  ],
  "parameters": {
    "basePrompt": "<6-panel sheet prompt>",
    "generations": "1",
    "aspectRatio": "16:9",
    "resolution": "2K"
  }
}
```

6-panel prompt structure:

```text
A 6-panel character reference sheet arranged as a 3-column by 2-row grid in a single horizontal frame, separated by thin clean white gutters between panels. Each panel shows the same single character from the approved base reference — [full visual descriptor of the character including build, face, skin, hair, makeup, full wardrobe head-to-toe, all accessories, jewelry, body markers, footwear, and held props].

Panel 1 (top-left): Full body front — straight-on neutral stance, full styling readable head-to-boots.
Panel 2 (top-center): Full body 3/4 turn — body angled 30 degrees from camera, weight on back hip, full styling readable from a turned angle.
Panel 3 (top-right): Full body back — straight back view, showing hair fall, back garment structure, pant/skirt drape, footwear, and accessory details from behind.
Panel 4 (bottom-left): Waist-up portrait — head, shoulders, and upper torso, face and upper styling clearly locked.
Panel 5 (bottom-center): Hands detail close-up — both hands forward, rings, nails, cuffs, sleeves, and any held prop clearly visible.
Panel 6 (bottom-right): Face detail close-up — tight crop from collarbone up, earrings, lips, skin texture, eyes, lashes, hairline, and makeup detail.

Pure white seamless studio backdrop applied uniformly across all six panels. Soft three-point classical lighting — key from camera-left at 45 degrees, gentle fill from camera-right, subtle rim defining shoulder and hair separation — applied uniformly across all six panels. Sharp focus across every panel. Identical character identity locked across all six panels — same face, same skin, same hair, same wardrobe, same accessories, same proportions in every cell.

[Locked photoreal stack].
```

Variation rule: if the user requests profile, boot detail, back-of-head, tattoo detail, prop detail, or other panels, swap specific panel contents but keep one 3x2 grid and explicit panel labels.

### Step 4: Scene Plates

Scene plates are available only when the user asks for a scene, environment, plate, moment, or setting. Do not proactively generate scene plates when the user only asked for character work.

Two modes:

- character-in-environment plate: locked character(s) placed into a cinematic world
- pure environment plate: no characters, useful as an anchor for Seedance reference-to-video

Default still workflow:

```json
{
  "workflowId": "fal-text-to-image",
  "projectRoot": "<absolute project root>",
  "model": "openai/gpt-image-2",
  "parameters": {
    "basePrompt": "<scene plate prompt>",
    "generations": "1",
    "aspectRatio": "16:9",
    "resolution": "2K"
  }
}
```

Reference-guided plate workflow:

```json
{
  "workflowId": "fal-image-to-image",
  "projectRoot": "<absolute project root>",
  "model": "openai/gpt-image-2/edit",
  "files": [
    { "path": "<absolute character sheet image path>" },
    { "path": "<absolute environment reference image path>" }
  ],
  "parameters": {
    "basePrompt": "<scene plate prompt>",
    "generations": "1",
    "aspectRatio": "16:9",
    "resolution": "2K"
  }
}
```

Scene plate prompt structure:

```text
A cinematic anamorphic still photograph, the kind of frame a director of photography would grab on set between takes.

[If characters are present: visual descriptor of each locked character by hair, makeup, wardrobe, jewelry, body markers, pose, and action in this moment.]

[Environment in full detail: location, architecture, materials, time of day, weather, lighting direction and color temperature, set dressing, props, atmosphere, palette.]

[Camera/lens/filtration/grade language matching the scene mode]. [Framing: wide establishing / medium two-shot / tight character-in-environment / extreme close-up]. [Depth of field and focus plane].

[Locked photoreal stack].
```

### Step 5: Detail and Key-Art Mode

For explicit detail or key-art requests, use GPT Image 2 with medium quality settings: chest-up portraits, face detail, skin/eye/hair fidelity, final key art, or exacting image edits.

Text-to-image detail:

```json
{
  "workflowId": "fal-text-to-image",
  "projectRoot": "<absolute project root>",
  "model": "openai/gpt-image-2",
  "parameters": {
    "basePrompt": "<portrait/detail prompt>",
    "generations": "1",
    "aspectRatio": "3:2-2k",
    "quality": "medium",
    "outputFormat": "png"
  }
}
```

Reference-guided detail edit:

```json
{
  "workflowId": "fal-image-to-image",
  "projectRoot": "<absolute project root>",
  "model": "openai/gpt-image-2/edit",
  "files": [
    { "path": "<absolute reference image path>" }
  ],
  "parameters": {
    "basePrompt": "<detail edit prompt>",
    "generations": "1",
    "aspectRatio": "auto",
    "quality": "medium",
    "outputFormat": "png"
  }
}
```

Detail prompt structure:

```text
[Visual descriptor of the character visible in frame: hair, makeup, wardrobe visible from chest up, jewelry, eye color/detail, lip detail, skin finish]. [Pose direction: head angle, shoulder angle, expression register].

[Background: pure white seamless studio or specified moody backdrop]. Classical beauty lighting — soft key from slightly above and camera-left at 35 degrees, soft fill at chest level from camera-right, subtle hair light behind defining the crown, soft underlight bounce lifting the eye sockets. [Framing: chest-up portrait / shoulders-up / face-only forehead-to-collarbone].

Extreme face fidelity. Real skin texture with visible pores, fine peach fuzz catching light along the jawline and upper lip, subtle subsurface scattering on the nose bridge, cheeks, and ears, micro-expression detail in the eyes and mouth corners, individual lash detail, real moisture and reflection in the iris with visible iris pattern, real lip texture with subtle natural lip lines, hair rendered strand by strand at the hairline with visible baby hairs and flyaways, fabric weave visible at the collar and shoulder.

[Locked photoreal stack].
```

## Seedance 2.0 Video Shots

Use Seedance after the still/reference assets are ready, or for text-only shots when the user explicitly wants to skip still development.

Use Kie Seedance workflows first when connected; use the Fal examples below as the default fallback shape after confirming the workflow schema with `pixelpig_describe_workflow`.

Text-only shot:

```json
{
  "workflowId": "fal-text-to-video",
  "projectRoot": "<absolute project root>",
  "model": "bytedance/seedance-2.0/text-to-video",
  "parameters": {
    "basePrompt": "<Seedance shot prompt>",
    "durationSeconds": "5",
    "aspectRatio": "16:9",
    "resolution": "720p",
    "generateAudio": "true"
  }
}
```

Image-to-video:

```json
{
  "workflowId": "fal-image-to-video",
  "projectRoot": "<absolute project root>",
  "model": "bytedance/seedance-2.0/image-to-video",
  "files": [
    { "path": "<absolute scene plate image path>" }
  ],
  "parameters": {
    "prompt": "<Seedance motion prompt>",
    "durationSeconds": "5",
    "aspectRatio": "16:9",
    "resolution": "720p",
    "generateAudio": "true"
  }
}
```

Reference-to-video:

Reference images are ordered string parameters, not extra `files[]` entries and not objects. Put reusable refs in `parameters.referenceImage1`, `parameters.referenceImage2`, etc. as absolute path strings. Use `files[]` only for the selected base/scene image that should become `@base` or `@scene`.

If a reference-to-video run says it needs reference images, do not retry by moving those images into `files[]`. Keep them in `parameters.referenceImageN` and verify the names are exact.

- `referenceImage1` maps to `@ref1`
- `referenceImage2` maps to `@ref2`
- `referenceImage3` maps to `@ref3`
- `files[0]` maps to `@base` or `@scene` for Seedance reference prompts

```json
{
  "workflowId": "fal-reference-to-video",
  "projectRoot": "<absolute project root>",
  "model": "bytedance/seedance-2.0/reference-to-video",
  "files": [
    { "path": "<absolute scene plate image path>" }
  ],
  "parameters": {
    "referenceImage1": "<absolute character sheet image path>",
    "referenceImage2": "<absolute prop or style reference image path>",
    "prompt": "Use @ref1 as the locked character reference, @ref2 as the prop/style reference, and @scene as the scene plate. <Seedance motion prompt>",
    "durationSeconds": "5",
    "aspectRatio": "16:9",
    "resolution": "720p",
    "generateAudio": "true"
  }
}
```

Use `bytedance/seedance-2.0/fast/*` only when the user prioritizes speed/cost over quality.

## Pre-Run Confirmation Rule

Before every new paid/long generation, show a short plain-language confirmation and wait for approval unless the user explicitly pre-approved the batch.

Use clean bullets:

```text
Pre-run check:
- Project: <project name/root>
- Movie: <movie name or none yet>
- Asset: <single-image base / 6-panel sheet / scene plate / GPT detail / Seedance shot>
- Character: <locked visual marker summary>
- Outfit/setting: <short summary>
- Workflow: <workflowId> / <model>
- Runtime/aspect: <if video>

Run it?
```

Skip only for minor iteration on an already approved asset in the same thread. New character, new outfit, new sheet, new scene plate, GPT detail, and every Seedance batch needs confirmation.

## Reference Reading Rules

When the user provides reference images, extract visible details only. Never invent details.

Capture:

- hair: color, length, style, texture, parting, accessories
- makeup: skin finish, brow shape, eye treatment, lashes, lip, cheek, freckles/marks only if visible
- wardrobe: every garment top to bottom, fabric, color, fit, structure, neckline, sleeves, hem, layering, footwear
- jewelry/accessories: earrings, necklaces, rings, bracelets, belts, bags, glasses, watches
- body markers: piercings, tattoos, nails, distinguishing features only if visible
- pose/energy: body angle, weight, hands, expression
- environment: location, architecture, materials, time/weather, lighting, set dressing, props, palette

Prompt output rules:

- no character names in model prompts
- no real brand names in model prompts
- no age labels like child, kid, teen, young, elderly, old
- no protected IP names
- no negative prompt blocks unless the workflow explicitly exposes one and the user asks
- no internal context like "as before" or "matching the prior scene"; every prompt must stand alone

## Locked Photoreal Stack

End still-image prompts with this stack unless the user explicitly requests stylization:

```text
Hyperrealistic photography. Real human skin texture with visible pores, subtle subsurface scattering on the cheeks, nose bridge, and ears, fine peach fuzz catching light along the jawline and cheekbones, slight skin imperfections — natural unevenness, not retouched. Hair rendered strand by strand with realistic flyaways, baby hairs at the hairline, individual strands catching light, light transmission through the hair ends, natural texture and movement. Fabric rendered with real weave detail, real weight, real drape, visible texture variation across the surface. Eyes with real reflection, real moisture, real depth in the iris. Jewelry with real metal surface detail and tarnish or polish appropriate to the piece. Kodak Vision3 500T film emulation, visible fine film grain, subtle chromatic aberration at the edges of the frame, soft lens vignette, cinematic color grade with warm mid-tones and slightly cooled shadows. Lived-in, not pristine. Photographic, not rendered.
```

For pure environment plates, remove human skin/hair/eye lines and keep material detail, light, lens, grain, atmosphere, and grade.

## Cinematic Mode Grammar For Plates And Video

Use these modes to structure scene plates and Seedance prompts:

- **M1 Narrative:** lived-in real-world streets, kitchens, cars, bars, interiors, exterior locations. ARRI Alexa 35, Panavision Ultra Vintage anamorphic, handheld natural breath, Black Pro-Mist 1/4, Kodak 250D, teal/amber split.
- **M2 Studio / Editorial:** white void, clean studio, fashion, editorial portrait, performance-on-set. Alexa Mini LF, Cooke S4/i spherical, locked tripod or slow push, Black Pro-Mist 1/2, saturated editorial grade.
- **M3 Action / Combat:** chase, stunts, combat, dust, debris, high physicality. Alexa 35, Ultra Vintage anamorphic, handheld shaky throughout, no stabilized shots, gritty documentary war-film texture.
- **M4 Crowds / Public Energy:** crowds, gatherings, markets, nightlife, events, public spaces, busy movement, layered background action. Alexa 35, natural handheld camera, documentary immediacy, realistic ambient light, controlled haze or bloom only when the scene calls for it.
- **M5 Atmospheric / Empty:** abandoned environments, weather plates, landscapes, no humans. Alexa Mini LF, Ultra Vintage anamorphic, locked-off or slow drift, strong negative space, palette-driven grade.

## Seedance Prompt Rules

Seedance prompts should describe the visible shot and motion, not internal planning.

Unless the user explicitly asks for subtitles, captions, music, or a soundtrack, append: `no subtitles, no captions, no music`.

Include:

- dynamic action across the duration
- camera motion and lens/framing
- character visual descriptors and environment details
- per-shot timing if composing multiple cuts
- total runtime matching `durationSeconds`
- diegetic audio only when `generateAudio` is true

Do not include music, lyrics, artist names, soundtrack cues, or score language unless generating separate audio through an audio workflow.

## Build The PixelPig Movie

After a video workflow completes, use each returned video `outputs[].filePath` as a movie clip:

```json
{
  "projectRoot": "<absolute project root>",
  "movieId": "<movie id>",
  "path": "<output filePath>",
  "insertIndex": 0,
  "durationSeconds": 5
}
```

After an audio workflow completes, use each returned audio `outputs[].filePath` as an audio clip:

```json
{
  "projectRoot": "<absolute project root>",
  "movieId": "<movie id>",
  "path": "<audio filePath>",
  "startTimeSeconds": 0,
  "durationSeconds": 5
}
```

For nontrivial ordering, trims, volume, or timing changes, prefer:

1. `pixelpig_get_project_movie`
2. `pixelpig_get_movie_json_schema`
3. edit the full movie JSON
4. `pixelpig_update_project_movie`

## HyperFrames Handoff

When the rough movie is ready or the user asks for a polished composition:

```json
{
  "projectRoot": "<absolute project root>",
  "movieId": "<movie id>",
  "includeRoughMovie": true,
  "prompt": "<creative brief and generated asset summary>"
}
```

Then invoke:

- `hyperframes-pixelpig` for PixelPig/HyperFrames workspace rules
- `hyperframes` for composition authoring
- `hyperframes-cli` for init/lint/inspect/preview/render
- `hyperframes-media` only for local preprocessing when needed

After rendering, attach the artifact:

```json
{
  "projectRoot": "<absolute project root>",
  "movieId": "<movie id>",
  "runId": "<run id>",
  "renderPath": "<absolute render path>",
  "name": "<layer name>",
  "promptSummary": "<brief summary>"
}
```

## Silent Checklist

Before running a generation:

- [ ] projectRoot resolved and will be passed to `pixelpig_run_workflow`
- [ ] current workflow/model verified with `pixelpig_describe_workflow`
- [ ] character locked if character appears
- [ ] single-image base exists before any 6-panel sheet
- [ ] 6-panel sheet is one 3x2 image, not six images
- [ ] scene plate only generated because the user asked for a scene/plate/environment
- [ ] GPT Image 2 used for image generation and image edit workflows
- [ ] prompt contains no names, brands, protected IP, age labels, or internal context
- [ ] pre-run confirmation delivered and approved
- [ ] returned outputs have `filePath`

Hard stop on failed/partial workflows unless the user explicitly chooses to continue with partial outputs.
