# Movie Editing

## Working While PixelPig Is Open

The user can be editing in the Movie Editor while you work. For an existing movie, use the live editor as the only writer: read its current JSON, make the edit, then send the full movie back through `pixelpig_update_project_movie`.

A separate working project can still be useful for generation and scratch assembly. It is just an empty folder passed as `projectRoot`. An explicit `projectRoot` needs no registration, so it stays out of the user's project list — `pixelpig_create_project` would register it. Direct mutation tools such as `pixelpig_create_movie` and the clip tools have nothing to conflict with inside that scratch project.

Media is the exception. Workflow outputs land in whichever project you run them against, and a movie referencing paths under your working folder breaks when that folder goes away. Running generations against the user's `projectRoot` puts assets in their media folders from the start. `pixelpig_synchronize_moved_file_paths` reconciles references if files move later.

While your workflow runs are going the user sees a robot badge with a live count, so they have some sense that work is happening.

### Updating Through The Live Editor

`pixelpig_update_project_movie` gives the full edited movie to the running app, which applies and saves it as one Movie Editor edit the user can undo.

Omit `movieId` and `expectedUpdatedUtc` to add a movie the project does not have yet — new movies are appended and the user's selection is left where it is, so a batch of them doesn't interrupt what they are editing. To replace an existing movie, pass its `movieId` and the `updatedUtc` read from `pixelpig_get_project_movie` as `expectedUpdatedUtc`. A `movie_changed` status means the movie changed while you were working: nothing was saved, and the response carries its current `updatedUtc`. Reread the movie, merge the intended edits into that current copy, and retry with the new `updatedUtc`.

There is no separate save or force call. A successful update means the live editor accepted and persisted the edit. Invalid input, a loading editor, a different open project, or an unavailable app fails clearly without saving.

`pixelpig_list_project_movies` returns `updatedUtc` for every movie in one call. Snapshotting those at the start gives you a cheap way to see what the user has touched, rather than finding out at handoff.

PixelPig must be open on the target project's Movie Editor for this workflow. If the tool reports that the editor or project is unavailable, surface that requirement instead of bypassing it with direct file mutation.

## Add Simple Clips

Use `pixelpig_add_movie_clip` for a straightforward video insertion in a scratch project and pass the workflow output's absolute `filePath`, insertion index, measured duration, and any requested trim or playback rate.

`pixelpig_add_movie_audio_clip` suits a new approved audio element in a scratch project with an explicit start, duration, trim, and initial gain. Source audio stays at `0 dB` unless the user asks to mute, duck, or replace it. Adding music does not imply muting the source.

For ordering, replacements, timing, fades, automation, or corrections, prefer a full fresh-read update:

1. `pixelpig_get_project_movie`
2. `pixelpig_get_movie_json_schema`
3. Change the minimal fields in the full JSON.
4. `pixelpig_update_project_movie`, passing the read `updatedUtc` as `expectedUpdatedUtc`.
5. Re-read and verify.

The clip tools write the movie file directly, so reserve them for scratch projects. Full updates to a user's project go through the open target Movie Editor. On `movie_changed`, reread, merge, and retry with the new revision token.

## Replace Approved Dialogue Safely

A generated line is ready to place once the user approves that exact take. Identify the source clip by ID, path, timeline start, and expected line text—not merely by array index or visual position.

For an approved line swap, preserve the existing clip's:

- ID
- timeline start
- gain/volume fields
- fades
- control points and other mix automation

That usually leaves only the source path, measured duration, and `trimEnd` to change, unless the user asks for a timing or mix change. Compare the dub duration with the source window and flag an overrun before writing. Verify the old path is absent from that clip and the new path is present after the update.

When replacing the last line, distinguish the old and new tracks by exact path and clip ID. A nearby or newly added clip is not on its own a reason to delete a track.

## Match Audio Levels

Measure integrated loudness and peak level before adjusting a clip. Compare against adjacent approved dialogue and apply a small gain change. Keep `volumeDb` and linear `volume` synchronized when both appear in the schema. Re-read and verify both representations after saving.

## Build Selected-Media Cuts

Read the live selection exactly as described in the main skill. Before changing movies or rendering a selected batch, provide one concise preflight and wait for approval. Confirm:

- one output per selected source versus one combined output
- output names and starting number
- trim, orientation, resolution, speed, fade, and destination
- source-audio and soundtrack policy
- whether work ends at verified rendering or includes publication

Default to one source per output, selection order, no reused sources, source audio at `0 dB`, movie resolution/orientation, 1x speed, configured project video folder, non-colliding names, existing movie fade, and no publication. Use a valid different soundtrack section per output when requested. Keep trims inside source duration.

For corrections, fetch each movie's current JSON and preserve intervening edits. Overwriting a prior deliverable needs explicit authorization. Verify saved timeline settings before rendering.

## Render And Verify

Use `pixelpig_render_movie` for native delivery. `suffix` is the sensible default collision policy; `overwrite` needs explicit authorization. Poll `pixelpig_get_movie_render` to `completed`, `failed`, or `cancelled`; cancel when the user stops the job.

Verify the finished file's path, dimensions, duration, video codec, and audio stream. `pixelpig_prepare_hyperframes_workspace(includeRoughMovie: true)` is not a finished export — its rough movie is composition context.

For accelerated social cuts, keep the chosen multiplier explicit. Embedded clip audio may follow clip speed; keep music and dialogue at natural pitch unless the user asks otherwise. Reposition or trim timeline audio so it does not outlast the video.

## Hand Off To HyperFrames

When the rough movie is ready or the user asks for a polished composition, call `pixelpig_prepare_hyperframes_workspace` with the project, movie ID, `includeRoughMovie: true`, and a concise creative brief. Use the applicable HyperFrames skills to author and render. Attach the verified render with `pixelpig_attach_hyperframes_layer` and record its run ID, path, layer name, and prompt summary.

Treat the run render and the user-facing delivery as two saved artifacts. Keep the canonical render inside `.pixelpig/hyperframes/runs/<runId>/renders/` so the layer retains its provenance, then copy the verified final non-destructively into the project's configured video folder with a clear delivery name. Never overwrite an existing delivery unless the user explicitly asks. Verify the delivery copy's duration and size (or checksum when useful), attach the canonical run render, and confirm the returned layer ID, enabled state, run ID, and render path.
