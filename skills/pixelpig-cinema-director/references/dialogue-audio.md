# Dialogue And Audio

## Separate Generation From Placement

Generate and review dialogue before changing a movie. Approval to create voices does not authorize placing them on the timeline. Do not update a movie until the user approves the specific audio take.

Identify every line by stable line number, exact text, source start/end time, source duration, target duration, generated filename prefix, take number, model, voice, speed, and approval state. Present the exact text when the user asks to revise a line.

## Discover Real Inputs

Call `pixelpig_list_workflows` and `pixelpig_describe_workflow` before using TTS, voice conversion, dubbing, or speech-to-text. Resolve the requested voice from its named PixelPig collection with `pixelpig_get_collection` and pass the actual selected voice file through the workflow input defined by the descriptor.

Treat the selected model's MCP descriptor as authoritative for exposed parameters and follow any structured parameter contract it returns. If the UI shows an advanced input that MCP omits, report the contract gap. Never invent a `scriptOverride`, timing field, speed control, or other parameter.

Use `files[].path` for local media. PixelPig performs provider upload and passes provider references internally; do not base64-embed large media unless the descriptor explicitly requires it.

## Choose Retake Versus New TTS Correctly

- If the line text is unchanged, retake the existing workflow output where supported so provenance and take numbering remain intact.
- If the text changes, run direct TTS with the corrected text. Recover the approved voice sample, model, provider, and speed from the prior run or notes; do not claim a retake can override text when it cannot.
- If the requested model has no speed control, generate normally and use a small, pitch-preserving `ffmpeg` tempo adjustment only after hearing the natural take.

Prefer sentence-level clips for dialogue replacement. Prefix files with a stable sortable pattern such as `<movie>-dialogue-02-take-01`. Keep alternates; do not overwrite an approved take.

## Compare Timing Before Approval

Measure source and generated durations. Flag each line whose generated duration exceeds the available source window or leaves a conspicuous gap. Report both values and the difference.

Avoid clipping terminal words. Preserve a short natural tail when generating or post-processing. If time compression is needed, apply the smallest pitch-preserving change that fits and review the ending again. Re-record with a faster model delivery when compression harms quality.

For sentence timing, transcribe the source with a connected Whisper/speech-to-text workflow and request the most granular timestamps the descriptor supports. Report whether it returns segment-, word-, or token-level timing. Do not promise timing levels absent from the contract.

## Compare Models Efficiently

For a voice/model audition, generate the same first sentence with each requested TTS model and matching voice reference. Keep text, sample, and prefix comparable. Report duration and cost information only when available from workflow history or descriptors. Let the user approve a model before generating the remaining lines.

For batch generation, wait for all line outputs, then provide a compact review table with line, exact text, take, duration, target window, difference, and file path. Generate alternate takes only for flagged or rejected lines.

## Measure Loudness Before Changing Gain

Inspect integrated loudness and peak level with a suitable meter such as `ffmpeg` loudnorm/ebur128 before changing timeline gain. Compare the target line with neighboring approved dialogue. Apply the smallest gain change needed and avoid clipping.

When editing movie JSON, keep decibel and linear fields consistent when both exist:

```text
linear volume = 10^(volumeDb / 20)
volumeDb = 20 * log10(linear volume)
```

Do not normalize every line blindly; match perceived dialogue level while preserving intentional dynamics.

## Use Other Audio Workflows Deliberately

- Text-to-audio: generate sound effects or short beds.
- Suno generation: prefer an available Kie Suno workflow for music; default to instrumental unless vocals are requested.
- Suno extend: lengthen an existing cached track instead of regenerating it.
- Audio-to-audio/Demucs: split stems.
- Audio-to-pulsemap: obtain tempo, beat, downbeat, sections, and RMS energy for beat-aware edits.
- Voice conversion/dubbing: audition on a short excerpt before converting a whole performance.

When the source audio is longer than the video, do not add the whole file blindly. Trim or split only after identifying the intended dialogue region.
