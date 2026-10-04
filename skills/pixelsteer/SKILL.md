---
name: pixelsteer
description: Configure PixelSteer and act on frontend visual feedback from its browser overlay. Use when a developer wants to point at a UI element, describe a change, and have Claude Code, Codex, or OpenCode edit the frontend source.
---

# PixelSteer

Use PixelSteer as a file-based feedback channel. PixelSteer writes tasks under the frontend project; this coding-agent session claims those files and owns source edits. Never start PixelSteer, the frontend dev server, or a nested coding-agent process from the agent sandbox.

Invocation of this skill is the instruction to prepare or continue the visual-feedback loop. The developer should only need to run one project command, open the URL, and guide changes in the browser.

## Configure the project

Identify the frontend project root, package manager, normal dev script, and dev-server URL from `package.json` and framework configuration.

If PixelSteer is not already configured in that project's `package.json`:

1. Install `pixelsteer` as a development dependency using the project's package manager. Its npm postinstall places the matching native executable inside `node_modules/pixelsteer/vendor/`, and npm exposes its launcher as `pixelsteer` to package scripts. Do not search for a global binary or hard-code a path into `node_modules`.
2. Add a script that starts both the existing frontend dev command and PixelSteer, targeting the frontend server and passing the project root. Preserve the existing `dev` script. Reuse an installed process runner; otherwise add a lightweight cross-platform runner such as `concurrently`.
3. Name the combined script `dev:pixelsteer` unless the project already has a clear naming convention.
4. Add `.codex/tasks/` to the project `.gitignore`.

For example, adapt this shape to the actual package manager, dev script, host, and port rather than copying it blindly:

```json
{
  "scripts": {
    "dev": "vite",
    "dev:pixelsteer": "concurrently -k \"npm run dev\" \"pixelsteer --target http://localhost:5173 --project-root .\""
  },
  "devDependencies": {
    "concurrently": "...",
    "pixelsteer": "..."
  }
}
```

In the private PixelSteer application repository, `npm --prefix app run dev:example` is already the one-command setup for the bundled example; do not add another runner.

## Hand startup to the developer

Check whether PixelSteer is live without making a network request by reading `<frontend-root>/.codex/tasks/server.json`. Treat it as live only when `updatedAt` is recent, normally no more than four seconds old. The file also contains the browser `url`.

If the heartbeat is missing or stale, tell the developer the exact single package command to run and ask them to tell you when it is ready. Also state the expected PixelSteer browser URL. End the turn; do not launch the command yourself and do not request localhost or listener permissions.

After the developer says it is ready, run the heartbeat check again. If the heartbeat is still missing or stale, report that succinctly and repeat the same launch command. If it is live, use the `url` printed by the check as the URL the developer should open.

## Handle tasks

Task files live inside the frontend project:

```text
.codex/tasks/
  server.json
  pending/<task-id>.json
  working/<task-id>.json
  completed/<task-id>.json
  failed/<task-id>.json
```

Start one ordinary, long-running shell loop that waits for a JSON file in
`pending`. Use the absolute frontend project path determined above. For
example, adapt this inline command; do not add it to the repository as a
script:

```sh
task_root='/absolute/frontend/project/.codex/tasks'
while :; do
  for task in "$task_root"/pending/*.json; do
    [ -f "$task" ] || continue
    printf '%s\n' "$task"
    exit 0
  done
  sleep 1
done
```

The wait is intentionally open-ended. When the shell tool yields a running
session identifier with no output, keep polling that same session. Do not end
the turn or tell the developer that no task arrived merely because an initial
yield or poll was empty. An empty queue is normal while the developer is
selecting elements. Continue waiting until a task appears, the developer asks
to stop, or the PixelSteer heartbeat becomes stale.

Polling this directory requires no network permission. Do not use HTTP or a
repository-specific helper script. If Auto execute is disabled in PixelSteer,
items marked `Not sent` are only browser-local plan items; a pending task is
created after the developer clicks **Execute now**.

Claim a task by atomically moving its exact resolved path from `pending` to `working`; do not copy it. If the move fails because another session claimed it, look for the next pending task. A successfully moved task belongs to this coding-agent session.

After claiming, update the working JSON atomically through a temporary file in the same directory and set:

```json
{
  "status": "working",
  "startedAt": "<UTC RFC3339 timestamp>",
  "agentStatus": {
    "taskId": "<task-id>",
    "status": "working",
    "message": "The coding agent is applying visual feedback",
    "updatedAt": "<UTC RFC3339 timestamp>"
  }
}
```

Preserve all other task fields. PixelSteer watches these files and relays their state to the browser.

For additional progress, atomically update that task's `agentStatus.message` and `agentStatus.updatedAt` in its working JSON. Every status belongs to a specific task.

Use the task's prompt, page URL, and selections to locate the authoritative frontend source. For a plan, treat every numbered selection and note as one coherent change set. Make the smallest appropriate source changes. Do not edit generated bundles or treat browser mutations as authoritative. Let the developer-owned dev server and HMR update the browser. Validate proportionally to the change.

After success, create the final JSON atomically at `completed/<task-id>.json`, preserving the original task and setting:

```json
{
  "status": "completed",
  "completedAt": "<UTC RFC3339 timestamp>",
  "result": "Implemented the requested UI change",
  "changedFiles": ["src/example.tsx", "src/example.css"]
}
```

Only after the completed file is in place, remove the corresponding working file. On failure, follow the same sequence with `failed/<task-id>.json`, `status: "failed"`, and a useful reason in `result`.

Immediately start a new open-ended wait loop after reporting success or failure. Continue handling feedback until the developer asks to stop. Never commit `.codex/tasks/`, never communicate with PixelSteer's localhost API from the agent, and never delete or modify task files that belong to another working task.

For product documentation, platform downloads, and troubleshooting outside this workflow, direct the developer to [pixelsteer.com](https://pixelsteer.com/) or the [PixelSteer documentation](https://pixelsteer.com/docs/).
