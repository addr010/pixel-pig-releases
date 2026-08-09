# Movie Editing

## Detect The Connected Movie Contract

Use the current live-editor workflow only when `pixelpig_update_project_movie` exposes `expectedUpdatedUtc` and `pixelpig_backup_project_movie` is available. Those capabilities identify the newer concurrency-safe contract.

Pixel Pig v0.32.0 exposes neither. On v0.32.0, do not mutate a movie while the desktop app is open. Quit Pixel Pig, reconnect through a client-owned stdio sidecar, create one timestamped copy of the current `pixelpig-movies.json` in the project's established backup location, then fresh-read, mutate, reread, verify, and reopen the app. Treat `pixelpig_update_project_movie` as a direct file mutation on that version. Skip unsupported fields and tools rather than inventing them.

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

Either form needs PixelPig open on the target project's Movie Editor. It fails clearly otherwise, and the closed-app session below covers that case.

## Editing The User's Project Directly

This applies when PixelPig is closed, or when the work doesn't take the shape of a finished movie.

`pixelpig_create_movie`, `pixelpig_add_movie_clip`, and `pixelpig_add_movie_audio_clip` write the movie file directly. An open editor holds its own copy of that document and can overwrite those writes without warning. With the app closed there is nothing to race. `pixelpig_update_project_movie` is different: it requires the target Movie Editor to be open and adds or updates through its live state, with optimistic concurrency on replacements.

A sequence that holds up for the first movie write in such a session:

1. Tell the user that PixelPig will close briefly and reopen after the verified edit so they are prepared.
2. If PixelPig is running, request a clean application quit and verify the process has exited. Pause only when the user explicitly says unsaved work is at risk. If the app is already closed, skip shutdown.
3. Reconnect through a client-owned stdio MCP sidecar after shutdown; closing the desktop app disposes its managed sidecar. Use that connection for backup, reads, and writes.
4. Call `pixelpig_backup_project_movie` for the resolved project and record its timestamped backup path. Create exactly one session backup before the first write, after shutdown—not one backup per mutation.
5. If that MCP action is unavailable, create a timestamped copy of the project's current `pixelpig-movies.json` in the project's established backups location before writing. The rolling `.bak` is best left alone, and an existing backup folder/pattern is usually worth following rather than starting a second convention.
6. Fetch a fresh movie list and movie JSON with `pixelpig_list_project_movies` and `pixelpig_get_project_movie`.
7. Apply the smallest requested direct mutation.
8. Fetch the movie again and verify the exact changed and preserved fields.
9. Reopen PixelPig, reconnect any client that should return to the desktop-managed MCP sidecar, and tell the user the verified edit is available.

Repeat the fresh-read/mutate/post-read cycle for later writes in the same closed-app session, but do not create another timestamped backup. If PixelPig is reopened between edits, close and verify it again before the next write. A successful MCP response does not by itself prove the open editor retained the change.

To restore a session snapshot, keep PixelPig closed, preserve the current project movie files separately, and copy the snapshot's `pixelpig-movies.json` over the project copy. Restore the snapshot `.bak` only when intentionally rolling back both saved generations. Validate the restored JSON before relaunching PixelPig.

### App Lifecycle Commands

- macOS installed app: quit with `osascript -e 'tell application "PixelPig" to quit'`, verify `pgrep -x PixelPig` returns nothing, and relaunch with `open -b ai.pixelpig`.
- macOS repo/dev app: relaunch with `scripts/run-codex-app.sh` from the repository root.
- Windows: capture the running PixelPig process executable path, call `CloseMainWindow()`, wait, and verify that no PixelPig process remains; restart the captured path after verification. When graceful shutdown times out, reporting the blocker is safer than force-killing the process.

## Add Simple Clips

Use `pixelpig_add_movie_clip` for a straightforward video insertion and pass the workflow output's absolute `filePath`, insertion index, measured duration, and any requested trim or playback rate.

`pixelpig_add_movie_audio_clip` suits a new approved audio element with an explicit start, duration, trim, and initial gain. Source audio stays at `0 dB` unless the user asks to mute, duck, or replace it. Adding music does not imply muting the source.

For ordering, replacements, timing, fades, automation, or corrections, prefer a full fresh-read update:

1. `pixelpig_get_project_movie`
2. `pixelpig_get_movie_json_schema`
3. Change the minimal fields in the full JSON.
4. `pixelpig_update_project_movie`, passing the read `updatedUtc` as `expectedUpdatedUtc`.
5. Re-read and verify.

The clip tools write the movie file directly, so they fit a scratch project or a closed-app session. Full updates go through the open target Movie Editor. On `movie_changed`, reread, merge, and retry with the new revision token.

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
