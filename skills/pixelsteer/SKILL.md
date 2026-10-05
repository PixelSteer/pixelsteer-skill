---
name: pixelsteer
description: Install and run PixelSteer alongside a frontend dev server, then apply visual feedback from its browser overlay. Use when a developer wants to point at a UI element, describe a change, and have Claude Code, Codex, or OpenCode edit the frontend source.
---

# PixelSteer

Invocation of this skill is the instruction to prepare and run the visual-feedback loop. Check the project, install or repair PixelSteer if needed, start the required servers in the background, verify the result, and keep handling feedback. The developer should only need to invoke the skill, open the URL you provide, and guide changes in the browser.

PixelSteer is a local reverse proxy and a file-based feedback channel. It writes tasks under the frontend project; this coding-agent session claims those files and owns source edits. Keep using the current agent session; do not launch a nested coding agent.

## Discover the project and running services

Identify the frontend root, package manager, dependency lockfile, documented dev command, and framework configuration. In a monorepo, distinguish the workspace directory used to run commands from the frontend root passed to PixelSteer. Check runtime requirements, installed dependencies, and required environment configuration. Use existing project conventions and preserve the normal dev command.

Inspect available process information, logs, and local HTTP responses to find an existing dev server for this frontend. Reuse it when its project identity and URL are established; an occupied port alone is not evidence that it is the right app. Determine the actual URL from configuration and startup output, including any port selected at runtime.

Read `<frontend-root>/.codex/tasks/server.json` if present. It contains `pid`, `url`, `startedAt`, and `updatedAt`. A recent `updatedAt`, normally within four seconds, suggests that PixelSteer is running. Confirm the process and proxy before reusing it. The heartbeat does not record the target URL or prove that the frontend is reachable; inspect the launch arguments or known launch metadata to establish the target. Avoid launching a second PixelSteer instance against the same task directory.

Perform routine setup and startup yourself. If a command is blocked by sandbox or network restrictions, use the agent environment's approval mechanism and retry with the required permissions. Do not replace this with instructions for the developer to run a terminal command. Ask for input only when a real blocker remains, such as an ambiguous frontend, missing required configuration, unavailable execution tools, or denied permissions; explain the concrete blocker and the minimum action needed.

## Ensure PixelSteer is usable

Prefer an existing project-local installation. If none exists, check for a usable `pixelsteer` executable on `PATH`. Verify the chosen launcher with `--help`; a dependency entry in `package.json` does not prove that the executable is installed. When checking a local package through its package manager, avoid implicitly downloading a package just to discover whether it exists.

If PixelSteer is missing, install `pixelsteer` as a development dependency using the project's package manager. The npm package requires Node.js 18 or newer and supports Windows, macOS, and Linux on x64 and ARM64. Its postinstall downloads and verifies the native executable; invoke it through the package manager's local executable support rather than hard-coding a vendor path. For a frontend without a JavaScript package manifest, use an isolated tool installation rather than adding an unrelated project manifest.

If the launcher exists but its native binary is missing, repair the installation with the package manager's supported rebuild or install procedure, then verify it again. For npm, the launcher recommends `npm rebuild pixelsteer`. If lifecycle scripts were blocked, use the package manager's package-specific approval mechanism for PixelSteer instead of disabling script restrictions globally. Inspect installation failures and address their cause before retrying.

Ensure `.codex/tasks/` is ignored relative to the frontend root. Reuse an existing combined dev script when appropriate. Adding a combined script or another process-runner dependency is not required to run the servers from the agent.

In PixelSteer's own application repository, the workspace-root `dev:example` and `dev:website` commands build the local executable and launch an agent session. Never invoke those commands from an agent: that would recursively launch another agent. Use the local executable supplied by that launcher and the selected frontend's own `dev` script. The example's frontend root is `examples/react-vite`; the website's is `website`.

## Start and verify the background processes

Use the agent's supported background-process or persistent terminal facility so the servers remain alive while you handle feedback. If that facility is unavailable, use the operating system's supported detached-process mechanism with stdin detached and output redirected to logs. A foreground command with a short timeout, or a bare shell `&` whose children are cleaned up when the tool exits, is not sufficient. Retain the process/session handles, commands, working directories, target URL, and log locations. Track which processes you started and which were reused; keep any saved runtime records and logs in an ignored location such as `.codex/tasks/runtime/`.

When the execution tool yields a session identifier for a long-running command, run each server in its own persistent tool session and retain that identifier. Redirect output to a log if needed and inspect it from separate short calls. Check readiness from a subsequent tool call after startup yields. Sandboxes can destroy all child processes when a command finishes, even when launched with `nohup` or `start_new_session=True`; a successful probe inside that launching command does not prove the servers will remain alive.

If an existing combined script starts both servers and neither is running, use it once and apply the readiness checks below to both. Otherwise reuse healthy existing servers and start only the missing ones.

1. If no suitable frontend server is running, install its missing dependencies using the existing lockfile and package manager, then launch its documented dev command in the background. Inspect its output and wait for the intended page to respond at its actual URL. Use bounded startup waits and request timeouts; if the process exits or readiness times out, inspect logs and fix the concrete problem before retrying. Do not invent a dev workflow when the project has none.
2. If no matching PixelSteer instance is running, start it in the background against that verified URL, passing the absolute frontend root. The command shape is `pixelsteer --target <actual-dev-url> --project-root <absolute-frontend-root> --port <available-proxy-port>`, invoked through the launcher established above. PixelSteer defaults to port `3100`; choose another available port if necessary. Keep its listener local. Do not kill an unrelated process to free a port. If another instance already owns this frontend's task directory with a different target, resolve that conflict before starting another instance; do not stop or retarget a reused process without authorization.
3. Wait for a fresh heartbeat and a successful request to `/__pixelsteer/health` at the recorded proxy URL. Neither the heartbeat nor this endpoint checks the target. Fetch the intended page through the proxy as well, confirm it is the expected frontend HTML with `/__pixelsteer/client.js` injected, and check that the injected asset loads. A listening process serving a proxy error or framework error page is not ready.
4. If browser tooling is available, check that the overlay renders and connects, and inspect failures such as content-security-policy or HMR errors. Resolve supported configuration issues without broadly weakening application security. If browser tooling is unavailable, state the limits of the checks performed; do not claim browser verification. If the frontend cannot be proxied successfully, report the specific incompatibility.

Once verified, give the developer the actual PixelSteer browser URL and immediately enter the task-watching loop in this same turn. Do not wait for a chat reply confirming startup. Keep both servers alive while waiting for and applying feedback.

During the session, inspect process exits and stale heartbeats. Recover processes you started when the logs identify a fix, recheck readiness, and resume waiting. Do not keep restarting an unchanged failing command. Report unresolved failures with the relevant log location and reason. When the developer ends the session, stop the watcher and the processes you started, including their child processes, unless asked to leave them running; leave reused processes alone. Also clean up processes you started if setup fails and cannot continue.

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

Start one long-running directory-wait loop using the shell or runtime available
on this platform and the absolute frontend root. Each iteration must check
`server.json` freshness, look for a JSON file in `pending`, and otherwise sleep
briefly (about one second). Return the exact pending path when a task appears.
If the heartbeat is missing, malformed, or stale, recheck after a short delay
to allow for a concurrent file replacement; if it remains unhealthy, exit the
wait and follow the recovery steps above. Do not use a loop that watches only
`pending` and can wait forever after the server dies.

The wait is intentionally open-ended. When the shell tool yields a running
session identifier with no output, keep polling that same session with bounded
tool waits so you can respond to messages and process failures. Do not end
the turn or tell the developer that no task arrived merely because an initial
yield or poll was empty. An empty queue is normal while the developer is
selecting elements. Continue waiting until a task appears, the developer asks
to stop, or the PixelSteer heartbeat becomes stale.

Use the filesystem for task delivery, claims, progress, and results; HTTP
requests are for startup verification and troubleshooting. Do not submit or
claim tasks through PixelSteer's HTTP API. If Auto execute is disabled in PixelSteer,
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

Use the task's prompt, page URL, and selections to locate the authoritative frontend source. For a plan, treat every numbered selection and note as one coherent change set. Make the smallest appropriate source changes. Do not edit generated bundles or treat browser mutations as authoritative. Let the running dev server and HMR update the browser. Validate proportionally to the change.

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

Immediately start a new open-ended wait loop after reporting success or failure. Continue handling feedback until the developer asks to stop or an unresolved runtime failure prevents further work. Never commit `.codex/tasks/`, and never delete or modify task files that belong to another working task.

For product documentation, platform downloads, and troubleshooting outside this workflow, direct the developer to [pixelsteer.com](https://pixelsteer.com/) or the [PixelSteer documentation](https://pixelsteer.com/docs/).
