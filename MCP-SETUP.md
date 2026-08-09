# PixelPig MCP Setup

PixelPig ships with a bundled MCP server for local AI clients.

## Default behavior

In normal desktop use, the PixelPig app starts and stops the MCP sidecar automatically.

- HTTP endpoint: `http://127.0.0.1:7361/mcp`
- Health check: `http://127.0.0.1:7361/health`

If PixelPig is already running, this is the simplest way for a local AI client to connect.

## If PixelPig is not running

Most MCP clients do not scan your filesystem for servers. They need explicit setup.

Use one of these launch modes:

- `stdio` mode:
  - macOS app bundle:
    - `/Applications/PixelPig.app/Contents/Resources/app/PixelPig.McpServer --stdio`
  - Windows release folder:
    - `C:\Path\To\PixelPig\PixelPig.McpServer.exe --stdio`
- HTTP mode:
  - macOS app bundle:
    - `/Applications/PixelPig.app/Contents/Resources/app/PixelPig.McpServer`
  - Windows release folder:
    - `C:\Path\To\PixelPig\PixelPig.McpServer.exe`

HTTP mode listens on port `7361` by default. Override with `PIXELPIG_MCP_PORT` if needed.

## What AI clients should expect

PixelPig MCP exposes:

- Tools:
  - `pixelpig_list_workflows`
  - `pixelpig_describe_workflow`
  - `pixelpig_run_workflow`
  - `pixelpig_get_workflow_run`
  - `pixelpig_find_workflow_for_file`
  - `pixelpig_retake_workflow_output`
  - `pixelpig_get_mcp_status`
  - `pixelpig_list_provider_tasks`
  - `pixelpig_recover_workflow_output`
  - `pixelpig_list_projects`
  - `pixelpig_create_project`
  - `pixelpig_synchronize_moved_file_paths`
  - `pixelpig_backup_project_movie`
  - `pixelpig_list_project_movies`
  - `pixelpig_get_project_movie`
  - `pixelpig_get_movie_json_schema`
  - `pixelpig_update_project_movie`
  - `pixelpig_create_movie`
  - `pixelpig_delete_project_movie`
  - `pixelpig_add_movie_clip`
  - `pixelpig_add_movie_audio_clip`
  - `pixelpig_split_movie_clip`
  - `pixelpig_list_movie_takes`
  - `pixelpig_select_movie_take`
  - `pixelpig_render_movie`
  - `pixelpig_get_movie_render`
  - `pixelpig_cancel_movie_render`
  - `pixelpig_prepare_hyperframes_workspace`
  - `pixelpig_attach_hyperframes_layer`
  - `pixelpig_list_collections`
  - `pixelpig_get_collection`
  - `pixelpig_get_post_schedule`
  - `pixelpig_set_post_times`
  - `pixelpig_schedule_post`
  - `pixelpig_cancel_scheduled_post`
  - `pixelpig_post_to_x`
- Resources:
  - `pixelpig://workflows`
  - `pixelpig://workflows/{workflowId}`

The workflow list only includes providers that are configured on this machine. `describe_workflow` also includes model guidance such as cost tier, cost text when available, and the spicy `🌶️` moderation hint pulled from the workflow catalog.

`pixelpig_list_provider_tasks` asks the provider directly for recent task/job state. It currently supports:

- EvoLink via `GET /v1/tasks`, with optional filtering for `status`, `model`, and `type`
- Freepik via the image/video upscaler task list endpoints, with optional filtering for `status`, `model`, and `type` (`image` or `video`)

`pixelpig_recover_workflow_output` is for cases where a provider task finished but the original client-side download failed. It currently supports:

- EvoLink video recovery by either a saved PixelPig `runId` or a raw `taskId` with `providerId: "evolink"`
- Freepik image/video upscale recovery by `runId`, or by `taskId` with `providerId: "freepik"` and an optional `workflowId` if you want to skip task-kind auto-detection

Recovered files are written to `~/Downloads/pixelpig-mcp` and returned as standard MCP output records with `filePath` populated.

`pixelpig_find_workflow_for_file` recovers workflow provenance from an existing output path. `pixelpig_retake_workflow_output` reruns one compatible item from a prior text-prompt batch while preserving traceable take numbering. Agents should use recovery and retake tools before paying for a blind replacement run.

Movie/project tools let agents use Pixel Pig as a production workspace: list or create configured projects, inspect or create project movies, edit validated movie JSON, render native deliveries, prepare a HyperFrames workspace with `movie-context.json`, and attach a user-managed HyperFrames render as a generated movie layer. `pixelpig_create_project` creates the root plus Pixel Pig's configured media folders and defaults to the OS videos/movies folder when no explicit location is provided. Project/movie responses include `projectStructure` with the configured media folders for images, video, audio, text, models, HyperFrames runs, and movie metadata. For nontrivial rough-cut edits, agents should use `pixelpig_get_project_movie`, optionally inspect `pixelpig_get_movie_json_schema`, then call `pixelpig_update_project_movie` with the returned movie's `updatedUtc` as `expectedUpdatedUtc`.

`pixelpig_update_project_movie` forwards the full validated movie to the running app instead of writing behind the Movie Editor's back. Pixel Pig applies and saves it as one undoable edit. If the movie changed after the agent read it, nothing is saved and the tool returns `status: "movie_changed"` with the current `updatedUtc`; reread the movie, merge the intended edits into that current copy, and retry with its `updatedUtc`. The call fails cleanly without saving when Pixel Pig is closed, the target Movie Editor is still loading, or it is open on a different project. Omitting `movieId` and `expectedUpdatedUtc` adds a movie the project does not have yet; new movies are appended without changing the user's current selection.

`pixelpig_render_movie` queues a native MP4 or animated GIF delivery and returns a `renderId`. Poll `pixelpig_get_movie_render` until completion, failure, or cancellation; use `pixelpig_cancel_movie_render` when the user stops the job. Ask for output format, orientation, pace, destination, filename, fade, and collision policy before rendering. The default `suffix` collision policy preserves existing files; use `overwrite` only with explicit approval.

Before the first direct movie-file mutation (`pixelpig_create_movie` or either clip-add tool) in an external editing session, close the Pixel Pig desktop app and call `pixelpig_backup_project_movie` with the absolute `projectRoot` and `desktopAppClosed: true`. It creates a timestamped snapshot under the Pixel Pig app-data `project-movie-backups` folder, copies the current `pixelpig-movies.json` and rolling `.bak` when present, writes a manifest, and returns the exact backup and source-file details. Full `pixelpig_update_project_movie` calls instead require the target Movie Editor to be open because the live editor owns that write.

To restore a snapshot, keep Pixel Pig closed, preserve the project's current movie files separately, then copy the snapshot's `pixelpig-movies.json` over the project copy. Restore the snapshot's `.bak` only when intentionally rolling back both saved generations; relaunch Pixel Pig after verifying the restored JSON.

### Pixel Pig v0.32.0 compatibility

Version 0.32.0 predates `pixelpig_backup_project_movie`, live-editor routing, and revision fields such as `expectedUpdatedUtc`. On that version, close Pixel Pig before every movie mutation, reconnect through a client-owned stdio sidecar, create one timestamped copy of the current `pixelpig-movies.json`, then fresh-read, mutate, reread, verify, and reopen the app. Upgrade to the latest Pixel Pig release before relying on live-editor concurrency or the newer movie automation tools.

`pixelpig_synchronize_moved_file_paths` repairs durable PixelPig references after files have already been moved or renamed outside the current MCP call. It accepts explicit absolute `originalPath`/`newPath` mappings, defaults to validation-only preview, and rejects missing destination files. Before applying, close the PixelPig desktop app so its in-memory state cannot overwrite the repaired files, then pass both `apply: true` and `desktopAppClosed: true`. Every applied repair creates a timestamped snapshot under the PixelPig app-data `file-path-sync-backups` folder before updating workflow history, collections, post history, project movies, and project albums. Mappings are exact and non-transitive: if a file moved from A to B and later from B to C, provide A→C when stored A references should resolve directly to C.

## Workflow runs

- `pixelpig_run_workflow`
  - starts a workflow and returns immediately with a `runId`
  - accepts:
    - optional `projectRoot` from `pixelpig_list_projects.projects[].projectRoot`; when provided, workflow outputs are written inside that project root
- If `projectRoot` is omitted, workflow outputs are written to a run-scoped MCP downloads folder instead of any current PixelPig project. Agents that need project-local results should call `pixelpig_list_projects` first and pass the discovered `projectRoot` back to `pixelpig_run_workflow`.
- `pixelpig_get_workflow_run`
  - checks one `runId` and returns immediately
  - returns `status`, `statusMessage`, `pollIntervalSeconds`, and `outputs[].filePath` when complete
- Image workflows usually take 15-240 seconds; video workflows usually take at least 1 minute per 5 seconds of generated video, so start once and poll with `pixelpig_get_workflow_run` until complete.

This works across providers because PixelPig persists workflow run state locally and records provider task identifiers underneath it.

## Recommended client setup

If your MCP client supports `stdio`, prefer that because it can launch the server even when PixelPig is closed.

If your MCP client supports HTTP-only connections, launch PixelPig first and connect to:

- `http://127.0.0.1:7361/mcp`

## MCP verification discipline

When the task is "verify MCP works" or "test MCP workflow wiring", the test must go through a real MCP client/tool invocation, not an ad hoc HTTP probe.

- Good MCP verification:
  - `pixelpig_list_workflows` returns the expected workflow
  - `pixelpig_describe_workflow` shows the expected parameters/models
  - `pixelpig_get_mcp_status` returns the expected `serverVersion` and `workflowContract`
  - `pixelpig_run_workflow` returns a `runId` and `pixelpig_get_workflow_run` checks that same run
- Acceptable HTTP-only checks:
  - `/health` confirms the sidecar is up
  - raw `/mcp` inspection while debugging transport/framing problems
- Not acceptable as a substitute for MCP verification:
  - declaring workflow wiring "tested" based only on `curl` against `/mcp`
  - declaring a tool integrated without exercising the corresponding MCP tool call path

Reason: the product contract being tested is the MCP tool surface, not just the underlying web server.

## Troubleshooting

- If the MCP client cannot connect over HTTP, make sure PixelPig is open or launch `PixelPig.McpServer` manually.
- If the MCP client cannot launch the server in `stdio` mode, confirm the executable path matches your install location.
- If `pixelpig_list_workflows` returns no workflows, configure at least one provider API key inside PixelPig first.
