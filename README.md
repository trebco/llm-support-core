
Browser library for LLM chat with tool calls: streaming from Anthropic,
OpenAI and Gemini in a Web Worker, a tool-call loop, Valibot tool
definitions, and a pluggable model registry. Split out of treb-llm-support,
which adds the TREB spreadsheet tools on top.

```ts
import { Stream, AddUserChatMessage, FindModel, SetModels, defineTool, CreateToolExecutor } from 'llm-support-core';
import LLMWorker from 'llm-support-core/worker?worker';
```

There is no build step: apps import the TypeScript source. Check this repo
out next to the app and depend on it as `"llm-support-core":
"file:../llm-support-core"`, installed with npm (which symlinks it, so edits
here reach the app; restart the app's dev server if one seems missed).
