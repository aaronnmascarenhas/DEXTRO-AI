# DEXTRO AI

An AI software-development workspace: part AI IDE, part project manager, part
multi-agent build environment.

This repository currently contains **Phase 1 — the foundation**. The shell,
design system, state, persistence, type contracts and provider abstraction are
real. Orchestration and real inference are deliberately **not** implemented
yet, and nothing in the UI pretends otherwise.

---

## What is DEXTRO AI?

The finished product takes a request such as *"Build a modern e-commerce site
with auth, database, API, admin dashboard and payments"* and:

1. analyses requirements → 2. decomposes into tasks → 3. selects models →
4. dispatches specialist agents → 5. merges results → 6. produces a plan →
7. executes it → 8. generates files → 9. tests → 10. debugs → 11. verifies.

Phase 1 builds the surface and the seams that each of those stages plugs into.

---

## Architecture

```text
USER
 ↓
WORKSPACE UI            src/components, src/stores, src/hooks
 ↓
APPLICATION SERVICES    src/lib/services        (the only callers of providers)
 ↓
ORCHESTRATION           src/lib/orchestration   (types + helpers; engine later)
 ↓
AI PROVIDER             src/lib/ai              (AIProvider ← MockAIProvider)
 ↓
EXECUTION / VERIFICATION                        (later phase)
```

Hard rule: **React components never call an AI provider or the database.**
They call an API route (or receive server-component props), which calls an
application service, which calls the provider.

```text
React component → /api/* route handler → service → AIProvider
```

---

## Tech stack

| Concern        | Choice                                        |
| -------------- | --------------------------------------------- |
| Framework      | Next.js 16 (App Router, React 19, Server Components) |
| Language       | TypeScript (strict)                           |
| Styling        | Tailwind CSS v4 with centralised design tokens |
| Primitives     | Radix UI (shadcn/ui conventions) + lucide-react |
| State          | Zustand (four small, focused stores)          |
| Validation     | Zod (API input **and** environment)           |
| Database       | PostgreSQL via Drizzle ORM                    |
| Tests          | Vitest                                        |
| Tooling        | ESLint, Prettier                              |

### Note on the database

The original spec sketched Prisma + SQLite for local development with
"PostgreSQL-ready" production. This workspace provisions PostgreSQL and wires
Drizzle ORM, so the schema is implemented directly against the production
target. The entity model is exactly the one specified — `User`, `Project`,
`Conversation`, `Message`, `Attachment`, `Plan`, `AgentTask`, `Execution` — and
all access goes through a single boundary module (`src/lib/database/client.ts`),
so the ORM remains replaceable.

---

## Folder structure

```text
src/
├── app/
│   ├── layout.tsx                 root layout, metadata, skip link
│   ├── page.tsx                   server component → AppShell
│   ├── globals.css                design tokens + base layer
│   ├── settings/page.tsx
│   └── api/
│       ├── health/route.ts        GET  /api/health
│       ├── development/route.ts   GET  /api/development
│       ├── conversations/…        REST: list/create, get/patch/delete, POST messages
│       ├── projects/…             REST: list/create, get
│       └── projects/route.ts      GET|POST /api/projects
├── components/
│   ├── ui/                        button, badge, card, input, overlays,
│   │                              tabs, segmented-control, scroll-area,
│   │                              feedback, code-block, file-chip, avatar
│   ├── layout/                    app-shell, top-bar, brand
│   ├── sidebar/                   sidebar, new-project-dialog
│   ├── chat/                      empty state, message-list, composer
│   ├── workspace/                 workspace-files
│   ├── activity/                  activity-panel
│   └── settings/                  settings-view
├── lib/
│   ├── ai/                        types, provider, mock-provider, agents
│   ├── orchestration/             types + helpers (readyTasks, events)
│   ├── workspace/                 types + path safety
│   ├── files/                     attachment contracts + validation
│   ├── services/                  chat, project, system, validation
│   ├── database/client.ts         single DB boundary
│   ├── config/env.ts              Zod-validated environment
│   ├── constants/                 app copy, starter prompts
│   └── utils/                     cn, id, format, logger, errors, api
├── db/                            Drizzle schema + pooled client
├── hooks/                         use-chat, use-autosize-textarea, use-media-query
├── stores/                        mode, chat, activity, workspace
└── types/                         public type barrel

tests/unit/                        vitest suites
```

---

## Local setup

```bash
npm install
cp .env.example .env     # adjust DATABASE_URL if needed
npm run db:push          # create tables
npm run dev              # http://localhost:3000
```

---

## Environment variables

| Variable | Required | Notes |
| --- | --- | --- |
| `DATABASE_URL` | yes | PostgreSQL connection string |
| `APP_URL` | no | defaults to `http://localhost:3000` |
| `NODE_ENV` | no | `development` by default |
| `LOG_LEVEL` | no | `debug` \| `info` \| `warn` \| `error` |
| `AI_PROVIDER` | no | `mock` (default) or `nvidia` (not implemented — fails fast) |
| `NVIDIA_API_KEY` | no | placeholder, unused in this phase |
| `NVIDIA_NIM_BASE_URL` | no | placeholder, unused in this phase |
| `MAX_PARALLEL_AGENTS` | no | default `5` |
| `MAX_PLAN_ITERATIONS` | no | default `3` |
| `MAX_ACT_RETRIES` | no | default `3` |
| `MAX_FILE_SIZE_MB` | no | default `100` |

Environment is parsed and validated once by `src/lib/config/env.ts`. Only
non-secret runtime flags are ever sent to the browser.

---

## Development commands

```bash
npm run dev           # dev server
npm run build         # production build
npm run start         # serve the production build
npm run lint          # eslint
npm run typecheck     # tsc --noEmit
npm run test          # vitest (unit)
npm run format        # prettier --write .
npm run db:generate   # drizzle migration files
npm run db:push       # apply schema to the database
npm run db:studio     # drizzle studio
```

---

## AI provider abstraction

Everything depends on the `AIProvider` interface, never on an implementation:

```ts
export interface AIProvider {
  readonly name: string;
  readonly simulated: boolean;
  chat(request: ChatRequest): Promise<ChatResponse>;
  stream?(request: ChatRequest): AsyncIterable<string>;
  listModels?(): Promise<ModelDescriptor[]>;
  health?(): Promise<ProviderHealth>;
}
```

`src/lib/ai/provider.ts` binds one implementation based on `AI_PROVIDER`. The
later NVIDIA phase adds a single `case "nvidia": return new NvidiaNIMProvider(…)`
branch — no changes to the chat UI, services, stores, schema or types.

`MockAIProvider` marks every response `simulated: true`, and the UI renders a
`stub` badge plus an explicit notice. No fake "thinking" states exist anywhere.

---

## Workspace: chat, conversations & projects (Prompt 2)

The workspace is persistent: every conversation, message and project lives in
PostgreSQL, and the URL says where you are.

### Routes

| Route                          | What it is                                                    |
| ------------------------------ | ------------------------------------------------------------- |
| `/`                            | Empty workspace — DEXTRO wordmark + hero composer             |
| `/chat/[conversationId]`       | A saved conversation (thread, header, docked composer)        |
| `/projects/[projectId]`        | Project overview: conversations, files, plans, workspace      |
| `/projects/[projectId]/files`  | Workspace Files (empty state until ACT exists)                |
| `/workspace`                   | Workspace Explorer — one entry per project                    |
| `/settings`                    | Appearance · Workspace · AI · Data · About                    |
| `/my-dextro` `/plugins` `/scheduled` `/inspiration` `/slides` | Honest "coming later" pages |

Unknown ids render a DEXTRO not-found state; route groups add loading and error
states. (Not-found pages stream, so Next reports HTTP 200 with a `noindex` meta
tag instead of 404 — the UI is correct, the status code is a known streaming
limitation.)

### API (all Zod-validated, `{ ok, data | error }` envelope)

```text
GET    /api/conversations?limit&cursor&q&projectId&includeEmpty   keyset-paginated recents + title search
POST   /api/conversations                                         { projectId?, title?, mode? }
GET    /api/conversations/[id]                                    conversation + messages
PATCH  /api/conversations/[id]                                    { title?, mode?, projectId? }
DELETE /api/conversations/[id]
POST   /api/conversations/[id]/messages                           { content, mode?, attachments?, retryUserMessageId? }
GET    /api/projects        POST /api/projects        GET /api/projects/[id]
```

### How a message flows

```text
Composer → useChat → POST /api/conversations/[id]/messages
        → chat service: validate → load conversation → SAVE user message
        → AIProvider (mock) → SAVE assistant message → bump updatedAt → return
```

- The user message is committed *before* the provider runs, so a failure never
  loses input. The error carries `userMessageId`; **Retry** re-runs the provider
  for that saved message (`retryUserMessageId`) without duplicating it. A
  message that never reached the server is re-sent in full. Retrying an
  already-answered message is rejected (`409 CONFLICT`).
- **Stop** aborts the request; the signal reaches the provider, no reply is
  saved, and the client re-syncs from the database.
- The first message generates a deterministic title (`lib/chat/title.ts`, no AI
  call); renaming never reorders recents, new messages do.
- Mode (`instant` / `plan` / `act`) belongs to the conversation and is restored
  when you return. PLAN and ACT are the primary modes; Instant is deliberately
  quiet.
- Starting from `/` creates the conversation, queues the first message in an
  in-memory *outbox*, navigates to `/chat/[id]`, and the thread view delivers it.
- Auto-scroll follows new content only while you are near the bottom; scrolled
  up, you get a "Jump to latest" pill instead.

### State: what lives where

| Where                         | What                                                          |
| ----------------------------- | ------------------------------------------------------------- |
| PostgreSQL (source of truth)  | projects, conversations, messages (+ metadata), mode, title   |
| `LibraryProvider` (context)   | sidebar projects + recents, refetched on change / tab focus   |
| `chat-store` (Zustand)        | active thread view, draft, attachments, optimistic/failed msgs |
| `workspace-store` (+localStorage) | sidebar collapse, selected project, developer mode — *only* |
| `toast-store`, `ui-store`     | notifications, dialogs                                        |

### Database changes (Drizzle, applied with `npm run db:push`)

- `messages.metadata jsonb` — mode + attachment references per message
- `messages.seq bigserial` — monotonic insertion order (deterministic ordering)
- `conversations.title` default `"New chat"`; index on `conversations.project_id`;
  composite index `messages(conversation_id, seq)`

(The original brief mentioned Prisma; this project uses Drizzle + PostgreSQL, so
the equivalent of `prisma generate && prisma db push` is `npm run db:push`.)

### Mock provider development affordances

`MockAIProvider` is deterministic: a fixed `delayMs` (450 ms outside tests) so
loading states are observable, markdown output so rendering is demonstrable, and
an explicit `/mock-fail` trigger — include it in a message to exercise the
failure + Retry UX. Every reply is flagged `simulated` and labelled "dev stub".

### Testing

```bash
npm test            # 100 unit + database-integration tests (Vitest)
npm run test:e2e    # opt-in browser suite (see header of tests/e2e/workspace.e2e.mjs)
```

Integration tests run against the local PostgreSQL and clean up after
themselves; they skip automatically if the database is unreachable.

## Current MVP status

Implemented:

- premium dark design system with centralised tokens (no scattered hex values)
- reference-led creation workspace: dense 248px navigation rail + calm main canvas
- collapsible desktop sidebar, mobile drawer and responsive workspace controls
- optional activity/files rail that mounts contextually; no width reserved on home
- Instant / PLAN / ACT interaction state in Zustand (default PLAN)
- hero + dock composer variants: autosize, ⏎ send / ⇧⏎ newline, errors
- quick-action prompt starters, keyboard New chat (`⌘/Ctrl+K`), project context
- mock chat flow end-to-end: UI → `/api/conversations/[id]/messages` → chat service → provider → DB
- database-backed recent conversations can be restored from the sidebar
- activity stream fed by real `WorkflowEvent`s emitted by the service
- Drizzle schema for the full entity model, applied with `db:push`
- health + development introspection endpoints, structured logging, error model
- unit tests for mode state, provider contract, env validation, utils, health

Not implemented (by design, later phases):

- NVIDIA NIM provider, model discovery, routing, specialist agents
- PLAN generation / ACT execution engines, verification runs
- file ingestion pipeline (100 MB uploads), code generation, sandboxed execution
- authentication, GitHub integration, deployment automation

---

## Future roadmap

1. **Phase 2 — PLAN**: planner agent, task graph, plan review UI.
2. **Phase 3 — Providers**: NVIDIA NIM provider, model catalogue, routing.
3. **Phase 4 — ACT**: execution engine, file generation, retries.
4. **Phase 5 — Files**: chunked upload, extraction, context injection.
5. **Phase 6 — Verification**: build/test runners, failure triage.
6. **Phase 7 — Platform**: auth, collaboration, deploys.

---

## Security notes

- provider credentials stay server-side; `env.ts` is never imported by client code
- only `NEXT_PUBLIC_*` style, non-secret flags reach the browser
- every API payload is validated with Zod before it reaches a service
- errors are normalised by `AppError` / `toApiError`; stack traces never ship
- workspace paths are validated against traversal before use
- no shell execution, no arbitrary filesystem writes in this phase
