
Browser library for LLM chat with tool calls: streaming from Anthropic,
OpenAI and Gemini in a Web Worker, a tool-call loop, Valibot tool
definitions, and a pluggable model registry. Split out of treb-llm-support,
which adds the TREB spreadsheet tools on top.

```ts
import { Stream, AddUserChatMessage, FindModel, SetModels, defineTool, CreateToolExecutor } from 'llm-support-core';
import LLMWorker from 'llm-support-core/worker?worker';
```
