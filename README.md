# Nx + TanStack AI Template

<a alt="Nx logo" href="https://nx.dev" target="_blank" rel="noreferrer"><img src="https://raw.githubusercontent.com/nrwl/nx/master/images/nx-logo.png" width="45"></a>

A streaming, type-safe AI chat with tool calling - TanStack Start and TanStack AI, wired up in an Nx monorepo.
<!-- BEGIN: nx-cloud -->
🚀 If you haven't connected to Nx Cloud yet, [complete your setup here](https://cloud.nx.app/get-started). Get faster builds with remote caching, distributed task execution, and self-healing CI. [See how your workspace can benefit](#nx-cloud).
<!-- END: nx-cloud -->

## Quick Start

Create a new workspace from this template:

```bash
npx create-nx-workspace@latest my-workspace --template nrwl/tanstack-ai-template
cd my-workspace
npm install
```

Set your Anthropic API key (the app builds and the chat UI renders without one - you only need it to actually talk to Claude):

```bash
export ANTHROPIC_API_KEY=sk-ant-...
# optional: pick a different Claude model (defaults to claude-sonnet-4-6)
export CHAT_MODEL=claude-sonnet-4-6
```

Start the dev server:

```bash
npx nx run @tanstack-ai-template/web:dev
```

Open the printed URL and start chatting. Try "Give me a random Nx tip" or "Show me the command to visualize the project graph" and watch the model call real tools.

---

## What's Inside

```
tanstack-ai-template/
- apps/
  - web/                           TanStack Start full-stack React app (the chat)
    - src/
      - routes/index.tsx           Streaming chat UI (useChat + SSE + tool rendering)
      - routes/api.chat.ts         Server SSE chat route (chat() -> Claude)
      - routes/__root.tsx          HTML shell + bundled Tailwind v4
      - components/NxCommandCard.tsx  Renders the showNxCommand tool output
      - router.tsx / client.tsx / ssr.tsx
    - vite.config.ts               Vite 8 + TanStack Start + Tailwind v4
- packages/
  - ai/                            Shared, isomorphic AI tools + types (@tanstack-ai-template/ai)
    - src/
      - tools.ts                   toolDefinition() tools: getNxTip, searchNxDocs, showNxCommand
      - nx-tips.ts                 Sample data the tools read
      - index.ts                   Public API + SYSTEM_PROMPT + DEFAULT_CHAT_MODEL
- nx.json                          Nx configuration
- package.json                     npm workspaces root
```

The app (`scope:web`) depends on the shared lib (`scope:shared`), so `nx affected` and Nx Cloud see a real project graph - run `npx nx graph` to view it.

---

## How It Works

This template is a thin, type-safe wiring of TanStack AI into TanStack Start on Nx.

### Server: a streaming chat route

`apps/web/src/routes/api.chat.ts` is a file-based TanStack Start server route. It turns the request into a `chat()` stream and returns it as Server-Sent Events:

```ts
import { chat, toServerSentEventsResponse, chatParamsFromRequestBody } from '@tanstack/ai';
import { anthropicText } from '@tanstack/ai-anthropic';
import { serverTools, SYSTEM_PROMPT } from '@tanstack-ai-template/ai';

const params = await chatParamsFromRequestBody(await request.json());
const stream = chat({
  adapter: anthropicText(MODEL),
  tools: Object.values(mergeAgentTools(serverTools, params.tools)),
  systemPrompts: [SYSTEM_PROMPT],
  messages: params.messages,
});
return toServerSentEventsResponse(stream);
```

### Client: useChat over SSE

`apps/web/src/routes/index.tsx` consumes that endpoint with `useChat`. It handles streaming text, loading and error states, a stop button, an empty state, and markdown rendering:

```ts
import { useChat, fetchServerSentEvents } from '@tanstack/ai-react';

const { messages, sendMessage, isLoading, error, stop } = useChat({
  connection: fetchServerSentEvents('/api/chat'),
  tools,
});
```

### Isomorphic tools

`packages/ai/src/tools.ts` defines tools once with `toolDefinition()` and shares them across server and client:

- `getNxTip` - a server tool that returns a random Nx tip (the model calls it, the server runs it).
- `searchNxDocs` - a server tool that searches the sample tip catalog by keyword.
- `showNxCommand` - a "render in the UI" tool. The server has no implementation; the model emits the tool call and the browser renders a copy-ready command card (`NxCommandCard`). One definition, shared types, custom UI - the model decides what to show, the UI decides how.

Tools emit `tool:progress` custom events while running, which the UI surfaces as a small status pill.

### Provider-agnostic by design

The default provider is Anthropic (Claude), but TanStack AI is provider-agnostic. Switching is a one-liner in `api.chat.ts` - swap the adapter import and call:

```ts
// import { openaiText } from '@tanstack/ai-openai'         -> openaiText(model)
// import { geminiText } from '@tanstack/ai-gemini'         -> geminiText(model)
// import { openRouterText } from '@tanstack/ai-openrouter' -> openRouterText(model)
```

Everything else - the SSE route, the tools, `useChat` - stays the same.

---

## Useful Nx Commands

```bash
npx nx run @tanstack-ai-template/web:dev                       # start the chat dev server
npx nx run @tanstack-ai-template/web:build                     # production build
npx nx graph                         # visualize the project graph (web -> ai)
npx nx run-many -t build             # build everything (second run is cached, instant)
npx nx affected -t build,typecheck   # run only what your changes touched
npx nx show projects                 # list projects
```

### Module boundaries

Projects are tagged with `scope:` and `type:` tags (`scope:web`/`type:app` for the app, `scope:shared`/`type:ai` for the lib). Add `@nx/enforce-module-boundaries` rules to keep the graph clean, for example so apps cannot import other apps:

```json
{
  "depConstraints": [
    {
      "sourceTag": "type:app",
      "onlyDependOnLibsWithTags": ["type:ai", "scope:shared"]
    }
  ]
}
```

---

## Nx Cloud

Nx Cloud supercharges CI:

- **Remote caching** - build artifacts are shared across your team and CI runners, so nobody rebuilds what has not changed.
- **Distributed task execution** - tasks run in parallel across multiple agents, cutting wall-clock CI time.
- **Self-healing CI** - flaky tasks are detected and re-run automatically, so intermittent failures do not block your team.

Get started: [https://nx.dev/nx-cloud](https://nx.dev/nx-cloud)

---

## Tech Stack

| Layer           | Technology                           |
| --------------- | ------------------------------------ |
| Build system    | Nx 23                                |
| Framework       | TanStack Start 1.x                   |
| AI SDK          | TanStack AI (type-safe, streaming)   |
| Default model   | Anthropic Claude (claude-sonnet-4-6) |
| Bundler         | Vite 8                               |
| Styling         | Tailwind CSS v4                      |
| Language        | TypeScript 6.0 (strict)              |
| Package manager | npm workspaces                       |

---

## Install Nx Console

Nx Console is an editor extension that enriches your developer experience. It lets you run tasks, generate code, and improves code autocompletion in your IDE. It is available for VSCode and IntelliJ.

[Install Nx Console &raquo;](https://nx.dev/docs/getting-started/editor-setup?utm_source=nx_project&utm_medium=readme&utm_campaign=nx_projects)

## 🔗 Learn More

- [Nx Documentation](https://nx.dev/docs)
- [Crafting Your Workspace Tutorial](https://nx.dev/docs/getting-started/tutorials/crafting-your-workspace)
- [Module Boundaries](https://nx.dev/docs/features/enforce-module-boundaries)
- [TanStack AI](https://tanstack.com/ai)
- [TanStack Start Documentation](https://tanstack.com/start/latest)
- [Nx Cloud](https://nx.dev/nx-cloud)

## 💬 Community

Join the Nx community:

- [Discord](https://go.nx.dev/community)
- [X (Twitter)](https://twitter.com/nxdevtools)
- [LinkedIn](https://www.linkedin.com/company/nrwl)
- [YouTube](https://www.youtube.com/@nxdevtools)
- [Blog](https://nx.dev/blog)
