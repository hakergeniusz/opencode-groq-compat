# opencode-groq-compat

Native **OpenCode V2** plugin port of [`pi-groq-compat`](https://github.com/hakergeniusz/pi-groq-compat).
It keeps the `groq` provider usable by replacing its stale model catalog with a
curated chat-model list and forcing Groq's `max_tokens` request field, and adds a
`/groq-compat` command with `status`, `refresh` and `test` subcommands.

## What the original did

`pi-groq-compat` re-registered pi's built-in `groq` provider with:

- `compat.supportsDeveloperRole: false` so pi's OpenAI-completions adapter sent
  `role: "system"` instead of `role: "developer"` (Groq's llama.cpp templates
  reject `developer` with a 400 minijinja error).
- `compat.maxTokensField: "max_tokens"` (Groq has no `max_completion_tokens`).
- Verified context/output limits and a curated model list that dropped audio
  models, prompt-guard classifiers and retired enterprise-only ids.
- A `/groq-compat` command: status / refresh (re-fetch `/models`) / test (probe
  the active groq model with a developer-role request).

## How it maps to OpenCode V2

| pi original | OpenCode V2 plugin |
|---|---|
| `pi.registerProvider("groq", {...})` | `ctx.provider.transform((editor) => …)`: patch the built-in `groq` provider in place with `editor.update(...)` + `editor.models.set(...)`, falling back to `editor.add(...)` if no built-in provider exists |
| `compat.supportsDeveloperRole: false` | **No equivalent field exists in OpenCode V2** — see Gaps below |
| `compat.maxTokensField: "max_tokens"` | `compatibility.maxTokensField = "max_tokens"` on the provider and every model |
| `contextWindow` / `maxTokens` | `limit.context` / `limit.output` (plus `limit.input`) on each `Model.Info`, built via `Model.Info.default(...)` |
| excluded models | curated model list passed to `editor.models.set("groq", …)` |
| `pi.registerCommand("groq-compat", …)` | `ctx.command.transform((editor) => editor.add({ name: "groq-compat", … }))` |
| `ctx.ui.notify(...)` | `ctx.session.synthetic({ sessionID, text })` (no `ui.notify` for server plugins) |
| `register(...)` again on refresh | mutate captured `source.models`, then `ctx.provider.reload()` |
| `ctx.model` / `ctx.modelRegistry` | `ctx.session.get({ sessionID }).model` and the captured curated model list |
| config via env | `GROQ_API_KEY` / `GROQ_API_KEY_2` / `GROQ_API_KEY_1`; plugin options via `ctx.options` |

Auth is inherited from OpenCode's built-in `groq` provider — the plugin does not
modify its credentials or connection. Only the optional `test` command reads
`GROQ_API_KEY` to send a one-off probe, exactly like the original.

## Load it

As a published package, add it to `opencode.json(c)`:

```jsonc
{
  "plugins": [
    "opencode-groq-compat",
    {
      "package": "opencode-groq-compat",
      "options": { "refreshAtStartup": false }
    }
  ]
}
```

Or drop the directory into `.opencode/plugins/` for automatic loading.

## Options

Passed through the object form of `plugins` and read from `ctx.options`:

| option | default | meaning |
|---|---|---|
| `refreshAtStartup` | `false` | Fetch the live `GET /models` list during setup instead of only on `/groq-compat refresh`. The original deliberately avoided startup network calls; this keeps that default. |

## Commands

```
/groq-compat           status: key presence, active model, registered models, compat state
/groq-compat refresh   re-fetch the live /models list and re-register (picks up new models)
/groq-compat test      send a developer-role probe to the active groq model and report OK/FAIL
```

Command output is posted as a synthetic message in the session where the command
was invoked (OpenCode V2 server plugins have no `ui.notify`).

## Model notes (from the original, verified on the free tier)

- `qwen/qwen3.8-27b`, `openai/gpt-oss-*`: full agent use (tools enabled).
- `allam-2-7b`: no tool calling on Groq — kept for plain chat, `tools` capability
  is disabled. Context ~4k (3500-word prompt OK, 4000-word prompt 400s).
- `meta-llama/llama-prompt-guard-*`: excluded (classifiers reject system messages).
- `whisper-*`, `canopylabs/orpheus-*`: excluded (audio endpoints).
- `llama-3.1-8b-instant`, `llama-3.3-70b-versatile`, `qwen/qwen3.6-27b`: excluded
  (enterprise-only, absent from the free tier's `/models`).

## Gaps / not preserved

1. **`supportsDeveloperRole` fix is a no-op in OpenCode V2.** The V2 provider/
   model compatibility schemas have no `supportsDeveloperRole` field, and
   OpenCode's `openai-compatible` provider emits `role: "system"` for system
   prompts (OpenCode has no "developer" message role internally). The original
   400-error bug does not occur in this host, so there is nothing to patch. The
   `test` subcommand is kept as a diagnostic and still sends a `developer` probe.
2. **`supportsStrictMode`** from the pi model definitions has no V2 equivalent;
   not set.
3. **Reasoning metadata** (`reasoning: true`) has no direct `Model.Info` field in
   OpenCode V2's schemas; capabilities are set to `tools`/text I/O only. Model
   limits and tool capability are preserved.
4. **Multi-account failover** (`GROQ_API_KEY_1/2` with `pi-multi-account`) is a pi
   extension feature with no V2 plugin equivalent here; the key fallback chain
   is retained only for the `test` probe.
5. Output uses session synthetic messages rather than toast notifications; a TUI
   part was not added because the extension is not TUI-focused.

## Develop

```bash
bun install
bun run typecheck
bun test test/     # requires @opencode/plugin installed
```

## License

MIT — 2026 hakergeniusz.
