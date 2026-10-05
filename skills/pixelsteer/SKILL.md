---
name: pixelsteer
description: Install and run PixelSteer alongside a frontend dev server, then apply visual feedback from its browser overlay. Use when a developer wants to point at a UI element, describe a change, and have Claude Code, Codex, or OpenCode edit the frontend source.
---

# PixelSteer

Invocation of this skill is the instruction to prepare and run the visual-feedback loop. Check the project, install or repair PixelSteer if needed, start the required servers in the background, verify the result, and keep handling feedback. The developer should only need to invoke the skill, open the URL you provide, and guide changes in the browser.

PixelSteer is a local reverse proxy and a file-based feedback channel. It writes tasks under the frontend project; this coding-agent session claims those files and owns source edits. Keep using the current agent session; do not launch a nested coding agent.

## Discover the frontend

Identify the frontend root, package manager, dependency lockfile, documented dev command, and framework configuration. In a monorepo, distinguish the workspace directory used to run commands from the frontend root passed to PixelSteer. Check runtime requirements, installed dependencies, and required environment configuration. Use existing project conventions and preserve the normal dev command.

Inspect available process information, logs, and local HTTP responses to find an existing dev server for this frontend. Reuse it when its project identity and URL are established; an occupied port alone is not evidence that it is the right app. Determine the actual URL from configuration and startup output, including any port selected at runtime.

Perform routine setup and startup yourself. If a command is blocked by sandbox or network restrictions, use the agent environment's approval mechanism and retry with the required permissions. Do not replace this with instructions for the developer to run a terminal command. Ask for input only when a real blocker remains, such as an ambiguous frontend, missing required configuration, unavailable execution tools, or denied permissions; explain the concrete blocker and the minimum action needed.

## Set up PixelSteer

Use a supplied development executable, an installed project-local launcher, or `pixelsteer` on `PATH`, in that order. Launch it directly without version or capability probes.

If PixelSteer is missing, install `pixelsteer` as a development dependency using the project's package manager. The npm package requires Node.js 18 or newer and supports Windows, macOS, and Linux on x64 and ARM64. Its postinstall downloads and verifies the native executable; invoke it through the package manager's local executable support rather than hard-coding a vendor path. For a frontend without a JavaScript package manifest, use an isolated tool installation rather than adding an unrelated project manifest.

If launch fails because the native binary is missing, repair the installation with the package manager's supported rebuild or install procedure, then retry. For npm, use `npm rebuild pixelsteer`. If lifecycle scripts were blocked, use the package manager's package-specific approval mechanism for PixelSteer instead of disabling script restrictions globally. Inspect installation failures and address their cause before retrying.

Ensure `.codex/tasks/` is ignored relative to the frontend root. Reuse an existing combined dev script when appropriate. Adding a combined script or another process-runner dependency is not required to run the servers from the agent.

Every PixelSteer server launch requires write access to its persistent license config directory, including an existing `PIXELSTEER_CONFIG_DIR` override. The CLI checks this before starting and reports the exact directory if access is blocked. Retry through the agent environment's approval mechanism with write access to that directory; keep the normal persistent location so licenses remain available across sessions. Do not work around denied access by moving license storage into a temporary or per-project directory. A browser URL is ready only after this startup check succeeds.

In PixelSteer's own application repository, the workspace-root `dev:example` and `dev:website` commands build the local executable and launch an agent session. Never invoke those commands from an agent: that would recursively launch another agent. Use the local executable supplied by that launcher and the selected frontend's own `dev` script. The example's frontend root is `examples/react-vite`; the website's is `website`.

## Start and verify the background processes

Use the agent's supported background-process or persistent terminal facility so the servers remain alive while you handle feedback. If that facility is unavailable, use the operating system's supported detached-process mechanism with stdin detached and output redirected to logs. A foreground command with a short timeout, or a bare shell `&` whose children are cleaned up when the tool exits, is not sufficient. Retain the process/session handles, commands, working directories, target URL, and log locations. Track which processes you started and which were reused; keep any saved runtime records and logs in an ignored location such as `.codex/tasks/runtime/`.

When the execution tool yields a session identifier for a long-running command, run each server in its own persistent tool session and retain that identifier. Redirect output to a log if needed and inspect it from separate short calls. Check readiness from a subsequent tool call after startup yields. Sandboxes can destroy all child processes when a command finishes, even when launched with `nohup` or `start_new_session=True`; a successful probe inside that launching command does not prove the servers will remain alive.

If no frontend server is running and an existing combined script starts both servers, use it once and verify both. Otherwise reuse a healthy frontend server and launch PixelSteer for this session.

1. If no suitable frontend server is running, install its missing dependencies using the existing lockfile and package manager, then launch its documented dev command in the background. Inspect its output and wait for the intended page to respond at its actual URL. Use bounded startup waits and request timeouts; if the process exits or readiness times out, inspect logs and fix the concrete problem before retrying. Do not invent a dev workflow when the project has none.
2. Launch `pixelsteer --target <actual-dev-url> --project-root <absolute-frontend-root> --port <proxy-port>` in the background through the chosen launcher. Use the requested browser port or default to `3100`. Launch without checking for or stopping an existing PixelSteer instance: the CLI replaces the previous proxy for the same frontend host and target port, including localhost aliases, even across different browser ports. Task files are preserved. If the browser port belongs to an unrelated service or another frontend's proxy, choose another port. Keep the listener local and use a separate frontend root for each target.
3. Wait for a fresh `<frontend-root>/.codex/tasks/server.json` heartbeat from this launch and a successful request to `/__pixelsteer/health` at its recorded `url`. The heartbeat includes `pid`, `startedAt`, and `updatedAt`; `updatedAt` should normally be within four seconds. Neither the heartbeat nor this endpoint checks the target. Fetch the intended page through the proxy as well, confirm it is the expected frontend HTML with `/__pixelsteer/client.js` injected, and check that the injected asset loads. A listening process serving a proxy error or framework error page is not ready.
4. If browser tooling is available, check that the overlay renders and connects, and inspect failures such as content-security-policy or HMR errors. Resolve supported configuration issues without broadly weakening application security. If browser tooling is unavailable, state the limits of the checks performed; do not claim browser verification. If the frontend cannot be proxied successfully, report the specific incompatibility.

Once verified, give the developer the actual PixelSteer browser URL and immediately enter the task-watching loop in this same turn. Do not wait for a chat reply confirming startup. Keep both servers alive while waiting for and applying feedback.

During the session, inspect process exits and stale heartbeats. An exit caused by replacement is expected: verify the successor before taking further action, and do not restart the superseded proxy. Recover other processes you started when the logs identify a fix, recheck readiness, and resume waiting. Do not keep restarting an unchanged failing command. Report unresolved failures with the relevant log location and reason. When the developer ends the session, stop the watcher and the processes you started, including their child processes, unless asked to leave them running; leave reused processes alone. Also clean up processes you started if setup fails and cannot continue.

## Handle tasks

Use the verified PixelSteer launcher for queue operations. In the examples below,
replace `pixelsteer` with that launcher and `/absolute/frontend` with the frontend
root. Tasks remain in `.codex/tasks/{pending,working,completed,failed}`; the CLI
handles their atomic updates and preserves all selection context and other fields.
Do not write a custom polling script or hand-edit lifecycle JSON.

Start one persistent command immediately after verifying the servers:

```sh
pixelsteer tasks next --project-root /absolute/frontend --wait
```

The command watches the queue, checks the server heartbeat, and quietly waits.
It atomically claims one task, records working status, and prints one JSON line
with `event: "task"` and the full `task` object, including its ID, prompt, page URL,
and selections. The command then exits. That task already belongs to this agent
session; use the returned context directly without another read or claim command.
Run only one outstanding `next` or `complete --wait` command per agent session.

When the execution tool yields a running session identifier, resume that same
session with bounded waits (at most 30 seconds). A completion acknowledgement
alone does not mean the waiting process has exited. Do not start another waiter,
end the turn, or report an empty queue because a tool poll returned no task.
Continue until a task appears, the developer asks to stop, or the command reports
an unhealthy heartbeat. On heartbeat failure, follow the recovery steps above.
If interrupted after claiming but before receiving its output, inspect working
files and recover only the claim made by this session; do not claim another task
and leave the first one stranded.

Use the task's prompt, page URL, and selections to locate the authoritative
frontend source. For a plan, handle all numbered selections and notes as one
coherent change set. Make the smallest appropriate source edits, let the dev
server update the browser, and validate proportionally. Do not edit generated
bundles or treat browser mutations as authoritative.

After success, save the result and start waiting for the next task in one command:

```sh
pixelsteer tasks complete TASK_ID --project-root /absolute/frontend \
  --result "Updated the heading and checked the rendered page" \
  --changed-file src/App.tsx --wait
```

Repeat `--changed-file` for each file changed, using paths relative to the frontend
root. Supply a truthful result reflecting the validation actually performed.
Completion atomically publishes the final file before removing the working file.
It prints a `completed` acknowledgement, then waits and returns the next claimed
`task` in the same command. Continue the edit → complete-and-wait loop immediately.
When explicitly ending the session, omit `--wait` from the final completion.

For a longer edit, optionally report useful progress with:

```sh
pixelsteer tasks progress TASK_ID --project-root /absolute/frontend \
  --message "Updating the layout and checking the mobile view"
```

If a task cannot be completed, use `tasks fail TASK_ID --result "Specific reason"`
with the same root and `--wait`; report any files already changed with
`--changed-file`. Failure saves the result before removing the working file and
then waits for further feedback. Fix command errors before continuing. If a final
acknowledgement was printed before a wait failed, that result is already saved;
recover the server and resume with `tasks next --wait`, without completing it again.

A user or test harness may supply a stop-marker file. Pass `--stop-file PATH` to
every waiting command so it can exit with `event: "stopped"` when that marker
appears. Otherwise, interrupt the current waiter when the developer asks to stop.
Then clean up owned servers as described above. An empty queue is normal and is
never a stop request.

Use these filesystem-backed commands for task delivery and results; HTTP requests
are for startup verification and troubleshooting. With Auto execute disabled,
`Not sent` items exist only in the browser until the developer clicks **Execute
now**. Never commit `.codex/tasks/` or modify another session's working task.

For product documentation, platform downloads, and troubleshooting outside this workflow, direct the developer to [pixelsteer.com](https://pixelsteer.com/) or the [PixelSteer documentation](https://pixelsteer.com/docs/).
