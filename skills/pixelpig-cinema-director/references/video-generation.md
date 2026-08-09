# Video Generation

## Choose The Shot Path

Use image-to-video or reference-to-video after still/reference assets are approved. Use text-to-video only when the user explicitly wants to skip still development. Prefer connected Kie equivalents where supported, then Fal, after describing the exact workflow and model.

Common Seedance shapes include:

- `kie-text-to-video` or `fal-text-to-video` for text-only shots.
- `kie-image-to-video` or `fal-image-to-video` for a base plate plus motion prompt.
- `fal-reference-to-video` or `evolink-reference-to-video` for several continuity references.

Do not copy remembered model IDs or parameters into a run without checking the descriptor.

## Preserve Reference Semantics

For Seedance reference-to-video, put the selected base/scene image in `files[0]` and reusable reference images in exact string parameters such as `referenceImage1`, `referenceImage2`, and `referenceImage3` when the descriptor exposes them.

- `files[0]` maps to `@base` or `@scene`.
- `referenceImage1` maps to `@ref1`, and so on.
- Do not move rejected `referenceImageN` values into `files[]` as a blind retry.
- Keep reference ordering stable across retakes.

## Write Visible Motion Prompts

Describe the visible shot rather than production reasoning. Include:

- Subject appearance and environment continuity.
- Dynamic action across the full duration.
- Camera movement, framing, lens feel, and focus behavior.
- Per-beat timing for multi-action shots.
- Diegetic audio only when audio generation is enabled.
- Total described time matching `durationSeconds`.

Unless requested, end with `no subtitles, no captions, no music`. Do not include score language, lyrics, or artist names in a video prompt; generate music separately.

Use the image reference's narrative, studio, action, crowd, or atmospheric mode consistently. Prefer fast model variants only when the user prioritizes speed or cost over quality.

## Review And Retake

Inspect the completed video or use a video-to-text workflow when vision is unavailable. Check identity continuity, wardrobe, action, camera behavior, artifacts, first/last frames, duration, audio, and whether the shot hands off cleanly to its neighbors.

Recover interrupted outputs before rerunning. For a single failed item from a text-prompt batch, use `pixelpig_retake_workflow_output` where compatible so take numbering remains traceable.

## Use Adjacent Video Workflows When They Fit

- Lipsync image/video: use an approved portrait or clip plus an approved dialogue file.
- Audio dubbing: use only when direct voice conversion sounds suitable; otherwise generate approved sentence-level TTS and edit the movie audio.
- Remaster: refresh a low-quality shot while preserving its extracted start/end continuity.
- Subtitle video: burn captions only when requested.
- Video upscale: apply after the cut is approved, not to every draft.

Confirm all input roles with `pixelpig_describe_workflow`. Do not infer that a video file can stand in for a dedicated audio or reference input unless the descriptor says so.
