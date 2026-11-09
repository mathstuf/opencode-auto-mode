# OpenCode Auto Mode

`opencode-auto-mode` is an OpenCode V2 server plugin that classifies tool invocations before execution. It automatically permits a conservative set of read-only operations, statically blocks selected dangerous commands, and sends other operations to a model for an `ALLOW` or `BLOCK` decision. Reviews fail closed.

## Requirements

- OpenCode V2.
- A configured model for operations that need LLM review.

## Install

Add the plugin and permission rules to `opencode.json` or `opencode.jsonc`:

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "plugins": [
    {
      "package": "opencode-auto-mode",
      "options": {
        "enabled": true,
        "model": "provider/model-id",
        "timeoutMs": 60000
      }
    }
  ],
  "permissions": [
    { "action": "*", "resource": "*", "effect": "ask" }
  ]
}
```

The permission rule routes actions through V2 permission evaluation. Explicit `deny` rules remain final. The plugin handles non-tool permission requests there and delegates tool-originated requests to its pre-execution hook, which reviews every tool invocation and blocks rejected or unreviewable operations. If the plugin is disabled or unavailable, the `ask` rule leaves actions under native permission handling.

For a local checkout, use the checkout directory as the package path (V2 configured local plugins are directories):

```jsonc
"plugins": [
  {
    "package": "/absolute/path/to/opencode-auto-mode",
    "options": { "enabled": true, "model": "provider/model-id" }
  }
]
```

## Options

| Option | Type | Default | Description |
| --- | --- | --- | --- |
| `enabled` | boolean | `true` | Disables the plugin when false. |
| `model` | string | Context/default model | Reviewer model in `provider/model` format. |
| `timeoutMs` | number | `60000` | LLM review timeout, clamped to 1–300 seconds. |

`OPENCODE_AUTO_REVIEWER_MODEL` and `OPENCODE_AUTO_REVIEWER_TIMEOUT_MS` can provide defaults. The configured `model` option takes precedence. If no model can be resolved or a review fails, the operation is blocked.

## Review policy

- **Static allow:** a narrow set of read-only shell commands and safe local introspection tools.
- **Static block:** selected high-risk commands such as `sudo`, root recursive deletion, disk formatting, and shutdown operations.
- **LLM review:** other tool calls, including edits, external paths, network calls, installs, and remote mutations.

The LLM receives bounded, redacted context from recent user messages, recent assistant text, Git state, and a small project-document excerpt. It must return exactly one line: `ALLOW: <reason>` or `BLOCK: <reason>`. Oversized, ambiguous, or incompletely redacted requests fail closed. The review prompt is in `src/auto-reviewer-prompt.md`; static rules are in `src/index.ts`.

The plugin uses V2's `ctx.generate.text` for reviews, rather than creating a reviewer session or adding a hidden agent. Logging is written to the OpenCode server console; the V2 server plugin API does not provide the V1 TUI toast API.

## Development

```sh
bun test
tsc --noEmit
```

The package exposes its server plugin from `src/index.ts` and depends on `@opencode/plugin`.

## Security notes

- This plugin reduces permission fatigue; it is not an operating-system sandbox.
- LLM decisions are probabilistic. Keep hard security invariants in static rules.
- Explicit V2 permission denials are never overridden by the permission hook.
- Reviewer context may be sent to a different provider than the parent conversation; select the model according to your data-handling requirements.

## License

MIT
