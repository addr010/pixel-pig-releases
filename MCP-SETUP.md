# PixelPig MCP Setup

PixelPig releases include a bundled MCP server for local AI clients.

## Fast path

If PixelPig is already running, connect your MCP client to:

- `http://127.0.0.1:7361/mcp`

Health check:

- `http://127.0.0.1:7361/health`

The desktop app starts and stops the MCP sidecar automatically during normal use.

## If PixelPig is not running

Most MCP clients do not discover servers automatically. They need an explicit command or URL.

Preferred setup:

- `stdio` launch on macOS:
  - `/Applications/PixelPig.app/Contents/Resources/app/PixelPig.McpServer --stdio`
- `stdio` launch on Windows:
  - `C:\Path\To\PixelPig\PixelPig.McpServer.exe --stdio`

Manual HTTP launch:

- macOS:
  - `/Applications/PixelPig.app/Contents/Resources/app/PixelPig.McpServer`
- Windows:
  - `C:\Path\To\PixelPig\PixelPig.McpServer.exe`

Default port:

- `7361`

Override it with `PIXELPIG_MCP_PORT` if needed.

## What the MCP server exposes

- Tools:
  - `pixelpig_list_workflows`
  - `pixelpig_describe_workflow`
  - `pixelpig_run_workflow`
  - `pixelpig_list_provider_tasks`
  - `pixelpig_recover_workflow_output`
  - `pixelpig_list_projects`
  - `pixelpig_create_project`
  - `pixelpig_list_project_movies`
  - `pixelpig_get_project_movie`
  - `pixelpig_get_movie_json_schema`
  - `pixelpig_update_project_movie`
  - `pixelpig_create_movie`
  - `pixelpig_add_movie_clip`
  - `pixelpig_add_movie_audio_clip`
  - `pixelpig_prepare_hyperframes_workspace`
  - `pixelpig_attach_hyperframes_layer`
  - `pixelpig_list_collections`
  - `pixelpig_get_collection`
- Tasks:
  - `tasks/get`
  - `tasks/result`
  - `tasks/list`
- Resources:
  - `pixelpig://workflows`
  - `pixelpig://workflows/{workflowId}`

Workflow discovery only shows providers that are configured on that machine. Model metadata also includes pricing hints and spicy `🌶️` moderation guidance when available.

Workflow generation uses MCP Tasks. Call `pixelpig_run_workflow` as a task-augmented `tools/call` with `params.task`; the returned `task.taskId` is checked with `tasks/get`, and final workflow outputs are returned by `tasks/result`. The old workflow run polling tools are not part of the public MCP contract.

## Troubleshooting

- If HTTP does not respond, start PixelPig first or launch `PixelPig.McpServer` manually.
- If `stdio` launch fails, verify the executable path matches your install location.
- If no workflows appear, open PixelPig and configure at least one provider API key.
