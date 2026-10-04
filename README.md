# PixelSteer skill for visual frontend feedback

[PixelSteer](https://pixelsteer.com/) lets you steer your coding agent visually. Click an element in your running app, describe the change, and let Claude Code, Codex, or OpenCode edit the actual frontend source.

This Agent Skill provides the setup and operating workflow for PixelSteer's browser overlay: it connects visual UI feedback to the coding agent you already use, keeps edits in your real codebase, and shows the result through your existing dev server and hot reload.

## Install the PixelSteer skill

Install it from GitHub with the [`skills`](https://github.com/antfu/skills) CLI:

```bash
npx skills add PixelSteer/pixelsteer-skill
```

Install it for Codex, Claude Code, and OpenCode explicitly:

```bash
npx skills add PixelSteer/pixelsteer-skill \
  -a codex \
  -a claude-code \
  -a opencode
```

Install only this skill globally:

```bash
npx skills add PixelSteer/pixelsteer-skill \
  --skill pixelsteer \
  -g
```

Then open your frontend project and invoke the skill with your agent:

- Codex: `$pixelsteer`
- Claude Code: `/pixelsteer`
- OpenCode: ask it to use `pixelsteer`

The skill inspects the project, configures the PixelSteer development workflow, and tells you the single command to run.

## Visual feedback for AI coding agents

PixelSteer turns a running frontend into precise, actionable context:

1. Select an element or area in the browser overlay.
2. Describe the UI change in plain language.
3. Send the feedback to your coding agent.
4. Review the real source edit as hot reload updates the page.

There are no screenshots to annotate and no need to explain which button, card, or heading you mean. PixelSteer carries the selected element, its styles, and page context into a local file-based task queue. Your agent edits normal source files; PixelSteer does not create a separate page model.

## Supported coding agents

- [Claude Code visual feedback](https://pixelsteer.com/claude-code-visual-feedback/)
- [Codex visual feedback](https://pixelsteer.com/codex-visual-feedback/)
- [OpenCode visual feedback](https://pixelsteer.com/opencode-visual-feedback/)

PixelSteer runs alongside your existing frontend dev server and coding agent. Its proxy, browser connection, and task queue stay local; your agent's own model-provider policies still apply.

## Learn more

- [PixelSteer — visual steering for coding agents](https://pixelsteer.com/visual-steering/)
- [PixelSteer documentation](https://pixelsteer.com/docs/)
- [Claude Code visual editor](https://pixelsteer.com/claude-code-visual-editor/)
- [Codex visual editor](https://pixelsteer.com/codex-visual-editor/)

Personal, educational, and other non-commercial use is free. Commercial use requires the appropriate lifetime license; see the [PixelSteer EULA](https://pixelsteer.com/eula/) for terms.
