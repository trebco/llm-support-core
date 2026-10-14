# llm-support-core

Browser library for LLM chat: multi-provider streaming in a Web Worker, the tool-call loop, tool definitions and a generic tool executor, a pluggable model registry, and markdown rendering for responses. It has no spreadsheet dependency. Split out of `treb-llm-support` (../treb-llm-support), which builds the TREB spreadsheet tools on top of it and re-exports it.

## Build & Check

```bash
npm install          # install dependencies
npm run typecheck    # tsc --noEmit
npm run audit        # fails on moderate+ advisories in runtime deps
```

Uses **npm** (not bun). There is no build step -- consumers import the TypeScript source directly (`src/index.ts`), linked with `file:../llm-support-core`. No test suite currently.

Consumers link this package and import its source, so imports here resolve against **this** package's `node_modules`. That's why the SDKs are regular `dependencies`, not peers.

**Consumers' dev servers can miss edits here.** Vite may keep serving a stale copy of a changed file from a linked package. If an edit seems to have no effect (or a worker silently fails to start), restart the consumer's dev server.

**valibot must stay at 1.5.0 or later** (`^1.5.0`): 1.4.1 and earlier have a moderate advisory (GHSA-5qjj-4xww-7phc, `flatten()` can throw on `record()` issue paths with inherited property names). Don't lower the range when adjusting dependencies, and run `npm run audit` after dependency changes.

## Architecture

- **Browser only**: `Stream()` uses `window.setTimeout`/`setInterval` and a `Worker`; SDK clients use `dangerouslyAllowBrowser`
- **Web Worker model**: `llm-worker.ts` runs in a Web Worker (apps load it with `import W from 'llm-support-core/worker?worker'`), receives an `InitMessage` via `postMessage`, and streams raw provider chunks back to the main thread
- **Provider-native transcripts**: messages are kept in each provider's own format (`TypedChatMessages`: `anthropic` / `gemini` / `openai-responses`), not translated to a common one. `stream.ts` has a chunk processor and a tool-result formatter per provider
- **Providers**: Anthropic (and Anthropic-API-compatible: DeepSeek, local servers), OpenAI Responses API, Google Gemini. The chat-completions path (`openai-chat-completion`: Together, OpenRouter, Kimi) is currently disabled in the worker (`ENOTIMPL`)
- **Tools**: a tool is a `ToolDefinition` (Valibot schema + JSON Schema, via `defineTool`). `CreateToolExecutor(tools, handlers)` returns a function that parses, null-strips, validates and dispatches a tool call to a `(context, input)` handler, with best-effort partial application while streaming. `context` is whatever the app's handlers need

### Key modules

| File | Purpose |
|------|---------|
| `stream.ts` | Main-thread orchestrator: `Stream()`, per-provider chunk processing, tool-call loop, `AbortStream`, `AddUserChatMessage`, `DropDanglingToolCalls` |
| `llm-worker.ts` | Web Worker entry point; builds provider clients from `provider.api` / `provider.base_url` |
| `llm.ts` | Provider streaming calls (`StreamAnthropicResponse`, `StreamResponsesAPI`, `StreamGeminiResponse`) |
| `tool-definition.ts` | `ToolDefinition`, `defineTool`, converters to Anthropic / OpenAI / Gemini formats, OpenAI strict mode |
| `tool-result.ts` | `ToolHandlerResponseType` (object / image / error) |
| `tool-executor.ts` | `CreateToolExecutor`, `StripNullProperties` |
| `chat-message.ts` | `ToolCallContent` / `ToolResultContent` |
| `models.ts` | Model/provider registry: `Models` (live array), `SetModels`, `FindModel`, `DefaultModels`, `ModelSchema`, `ResolveProvider` |
| `md.ts` | Markdown rendering (markdown-it + highlight.js + KaTeX), `Format`, `formatConfig` |

## Conventions

- Tool schemas are defined with **Valibot** and converted to JSON Schema for the APIs. The canonical format is Anthropic's `input_schema` shape
- The worker picks the SDK from `provider.api` and the endpoint from `provider.base_url` -- never from the provider name. Adding a compatible provider is data, not code
- Models are pluggable: apps call `SetModels(list)` (validated with `ModelSchema`, throws on bad input or duplicate names, leaves the list unchanged on failure). `Models` is replaced **in place**, so don't reassign it; code that derived lists from it must rebuild
- Apps persist whole `Model` objects and send them to the worker, so the worker must cope with old shapes. `ResolveProvider` fills `api`/`base_url` from the built-in provider of the same name when `base_url` is missing
- `Stream()` edits `params.messages` **in place**, and calls the optional `params.changed(messages)` hook after each batch of writes (chunk drain, tool-result push, error report, exit cleanup). Consumers whose state only accepts writes through a setter (e.g. a Solid store) pass a plain working copy and copy it back in `changed`. Any new write to `messages` in `stream.ts` must be followed by `Notify(params)`, or those consumers won't see it
- Keep this package free of app- or spreadsheet-specific code; that belongs in the consuming library (e.g. treb-llm-support)
