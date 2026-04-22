# MeetingScribe — Design

**Date:** 2026-04-22
**Status:** Approved (design phase), pre-implementation
**Author:** Bright Nwokoro (via brainstorming with Claude)

## 1. Purpose

Public portfolio repository demonstrating AI-engineering craft. The distinctive artifact is a single structured-output schema driving two LLM providers (Anthropic Claude + OpenAI), streaming partial JSON to the UI in real time. Everything else in the project exists to frame that artifact.

This document supersedes the aspirational surface area of the initial README. The README will be rewritten during implementation to match what is actually built.

## 2. Scope

### In scope (MVP)

- Next.js 15 App Router application, TypeScript strict.
- Stateless request/response — no database, no auth, no persisted meetings.
- Transcript paste input only. No audio upload, no Whisper integration.
- Claude Sonnet 4.6 default extraction provider. GPT-4o fallback provider. Both consume the same JSON Schema.
- Streaming extraction: partial tool-call JSON surfaces progressively in the UI.
- Hybrid demo: 3 hand-authored sample transcripts unlocked by default; bring-your-own-key for custom input.
- Rate limiting via Upstash Redis (free tier) when env vars present; gracefully no-ops otherwise.
- Export generated summary as JSON or Markdown (copy to clipboard).
- Dark, minimal UI (Vercel/Linear aesthetic). Landing page + `/app` tool.
- Vitest unit tests including a schema-contract test against committed golden fixtures.
- GitHub Actions CI running lint, typecheck, test, build.

### Out of scope (deferred to roadmap, called out honestly in README)

- Audio upload and transcription (Whisper API or local whisper.cpp).
- Supabase auth, Postgres persistence, object storage.
- Resend email delivery.
- Docker Compose self-host stack.
- Speaker diarization, calendar integration, Slack bot, retrieval over past meetings, fine-tuned extraction, offline local LLM.
- E2E tests (Playwright).
- Theme toggle, user settings beyond BYO-key.

## 3. Architecture

Single Next.js application. No external services at runtime except the LLM providers and optional Upstash Redis for rate limiting.

### Routes

| Path | Type | Purpose |
|---|---|---|
| `/` | page | Landing. One viewport. Headline, pitch, CTA to `/app`. |
| `/app` | page | The tool. Two-column layout on desktop, stacked on mobile. |
| `/api/summarize` | POST, SSE stream | Receives transcript, streams partial Summary JSON. |

### Runtime

`/api/summarize` uses Edge runtime (`export const runtime = 'edge'`).

Rationale:
- 30-second timeout on Vercel Hobby (vs 10s for Node Serverless). Claude extraction on long transcripts can exceed 10s.
- Native Web streaming APIs (`ReadableStream`, `TransformStream`) are first-class.
- No filesystem or Node-only dependencies needed (audio was deferred).
- Tradeoff accepted: if audio is added later, that specific route migrates to Node.

### Request shape

```ts
POST /api/summarize
Content-Type: application/json

{
  "transcript": string,          // required, 1..50_000 chars
  "model": "claude-sonnet-4-6" | "gpt-4o",  // optional, default claude
  "byoKey": string | undefined   // optional, provider key from the browser
}
```

### Response shape (SSE)

```
event: partial
data: {"summary":"The team...","decisions":[],"action_items":[],"open_questions":[]}

event: partial
data: {"summary":"The team discussed...","decisions":["Ship v2 by Friday"],...}

event: done
data: {"summary":"...","decisions":[...],"action_items":[...],"open_questions":[...],"_meta":{"provider":"claude","model":"claude-sonnet-4-6","usage":{"input":1234,"output":456}}}
```

Error path:
```
event: error
data: {"message":"Invalid API key for openai","code":"invalid_key"}
```

## 4. Data flow

```
Client                         /api/summarize (Edge)                  Provider
------                         ---------------------                   --------
transcript + optional key  ──▶ 1. Zod-validate body (413 if too long)
                               2. Rate-limit check (skipped if byoKey)
                               3. Pick adapter from model id
                               4. adapter.stream({ transcript, key }) ─▶ tool_choice: record_summary
                                                                        (Claude)
                                                                        response_format: json_schema
                                                                        strict: true (OpenAI)
                                                                    ◀── tool_use input_json_delta
                                                                        OR content delta
                               5. Accumulate buffer; parse partial
                                  via `partial-json`; yield chunks
partial events (render       ◀── 6. SSE: event: partial + JSON
progressively) ──────────────┐
                             │
                             ▼
                               7. On message_stop / stream end:
                                  SummarySchema.parse(buffer)
                                  → SSE: event: done + final
final event (hide skeletons, ◀── 8. Close stream
show export buttons)
```

## 5. LLM adapter layer

The centerpiece. One schema, two providers, streaming partial JSON.

### 5.1 Schema — single source of truth

`src/lib/llm/schema.ts` exports both a JSON Schema (for providers) and a Zod schema (for runtime validation of parsed output).

```ts
export const summaryJsonSchema = {
  type: "object",
  properties: {
    summary: { type: "string", description: "3–5 sentence executive overview." },
    decisions: { type: "array", items: { type: "string" } },
    action_items: {
      type: "array",
      items: {
        type: "object",
        properties: {
          task: { type: "string" },
          owner: { type: "string" },
          due_date: { type: "string", format: "date" },
        },
        required: ["task", "owner", "due_date"],
        additionalProperties: false,
      },
    },
    open_questions: { type: "array", items: { type: "string" } },
  },
  required: ["summary", "decisions", "action_items", "open_questions"],
  additionalProperties: false,
} as const;

// Runtime validator, mirrors the JSON Schema
// due_date empty string → undefined via .transform()
export const SummarySchema = z.object({
  summary: z.string(),
  decisions: z.array(z.string()),
  action_items: z.array(z.object({
    task: z.string(),
    owner: z.string(),
    due_date: z.string().transform(s => (s === "" ? undefined : s)),
  })),
  open_questions: z.array(z.string()),
});
export type Summary = z.infer<typeof SummarySchema>;
```

Design note: OpenAI strict structured outputs require every property in `required` and `additionalProperties: false` everywhere. `due_date` is required at the JSON Schema level but semantically optional — providers are instructed via description to use empty string when unknown. The Zod schema accepts empty string and normalizes it to `undefined` post-parse.

### 5.2 Provider interface

```ts
// src/lib/llm/types.ts
import type { Summary } from "./schema";

export type PartialSummary = {
  summary?: string;
  decisions?: string[];
  action_items?: Array<{ task?: string; owner?: string; due_date?: string }>;
  open_questions?: string[];
};

export interface StreamChunk { partial: PartialSummary }

export interface StreamResult {
  final: Summary;
  usage: { input: number; output: number };
  provider: "claude" | "openai";
  model: string;
}

export interface LLMAdapter {
  name: "claude" | "openai";
  defaultModel: string;
  stream(args: {
    transcript: string;
    apiKey: string;
    model?: string;
    signal?: AbortSignal;
  }): AsyncGenerator<StreamChunk, StreamResult>;
}
```

### 5.3 Anthropic adapter (`src/lib/llm/anthropic.ts`)

- Uses `@anthropic-ai/sdk` `messages.stream()`.
- Request body:
  - `model`: from arg or `ANTHROPIC_MODEL` env or `claude-sonnet-4-6`.
  - `max_tokens`: 2048.
  - `tools`: `[{ name: "record_summary", description, input_schema: summaryJsonSchema }]`.
  - `tool_choice`: `{ type: "tool", name: "record_summary" }` — forces the model to invoke exactly this tool.
  - `system`: concise system prompt explaining the role.
  - `messages`: `[{ role: "user", content: transcript }]`.
- Streaming loop: iterate events, collect `content_block_delta` events with `delta.type === "input_json_delta"`, append `delta.partial_json` to buffer.
- After each delta, run buffer through `partial-json` library → yield `{ partial }` chunk.
- On stream end, validate complete buffer with `SummarySchema.parse()`. Return `StreamResult` with usage from the final `message_delta`.

### 5.4 OpenAI adapter (`src/lib/llm/openai.ts`)

- Uses `openai` SDK `chat.completions.create({ stream: true })`.
- Request body:
  - `model`: from arg or `OPENAI_MODEL` env or `gpt-4o-2024-08-06` (first GA with strict json_schema).
  - `response_format`: `{ type: "json_schema", json_schema: { name: "record_summary", schema: summaryJsonSchema, strict: true } }`.
  - `messages`: `[{ role: "system", content: systemPrompt }, { role: "user", content: transcript }]`.
- Streaming loop: iterate chunks, append `choices[0].delta.content` fragments to buffer.
- After each fragment, same `partial-json` parse → yield chunk.
- On final chunk (`finish_reason === "stop"`), validate with `SummarySchema.parse()`. Usage read from the last chunk's `usage` field (OpenAI emits it when `stream_options: { include_usage: true }`).

### 5.5 System prompt

Shared across providers, defined once in `src/lib/llm/prompt.ts`:

> You extract structured summaries from meeting transcripts. You must always call the record_summary tool with fields that faithfully reflect the transcript. Owners should be named individuals when identifiable; use "unassigned" when an action has no clear owner. Due dates should be ISO dates (YYYY-MM-DD) when stated; use empty string when not stated. Open questions are items raised in the meeting that were not resolved. Do not invent facts.

### 5.6 Adapter selection

`src/lib/llm/index.ts` exports `pickAdapter(model: string): LLMAdapter`. Model string prefixes decide:
- `claude-*` → AnthropicAdapter
- `gpt-*` → OpenAIAdapter
- unknown → throw 400

### 5.7 Failure modes and error taxonomy

| Situation | Handling |
|---|---|
| Transcript empty or too long | Pre-flight Zod check; return 400/413 with message. |
| Invalid BYO-key | Provider SDK throws 401; caught, SSE `error` with `code: "invalid_key"`. |
| Rate limited | Rate-limit check returns 429 before streaming starts; JSON body, not SSE. |
| Provider returns malformed JSON (should be impossible with strict mode; defensive) | `SummarySchema.parse` throws ZodError; SSE `error` with `code: "schema_mismatch"`. |
| Client aborts | `AbortSignal` plumbed to provider; stream closes; no further cost incurred. |
| Upstream provider 5xx | Caught; SSE `error` with `code: "provider_unavailable"`; original error logged server-side. |

All error events close the SSE stream after emission.

## 6. Rate limiting

`src/lib/rate-limit.ts` wraps `@upstash/ratelimit` with `@upstash/redis`.

- Key: client IP (from `x-forwarded-for`, fallback `x-real-ip`).
- Limit: 10 requests per 24 hours per IP.
- Bypassed entirely when `byoKey` is present on the request.
- Bypassed entirely when `UPSTASH_REDIS_REST_URL` or `UPSTASH_REDIS_REST_TOKEN` are unset — `console.warn` once at module init, then no-op. This is the default for a fresh fork.

## 7. UI structure

### 7.1 Aesthetic

- Dark only, no toggle. Background `zinc-950`, surfaces `zinc-900`, borders `zinc-800`.
- Text `zinc-100` / `zinc-400` / `zinc-500`.
- Single accent: `emerald-400`. Used on the CTA, streaming caret, and JSON syntax highlight keys.
- Type: Inter (UI), JetBrains Mono (JSON result card).
- No shadows except on the primary CTA. Borders and negative space do the work.

### 7.2 Landing (`/`)

Single viewport, no scroll-jacking. Header (sparse), hero headline ("Parseable meeting summaries. Not prose."), two-line subhead, CTA pair (emerald primary → `/app`, ghost → GitHub), short "why structured output" explainer, inline mini JSON preview. No logos, testimonials, or marketing fluff.

### 7.3 Tool (`/app`)

Two-column desktop, stacked on mobile:

- Left: transcript textarea + sample picker (3 chips) + `Summarize →` button + settings gear for BYO-key dialog.
- Right: summary card with four sections (summary, decisions, action_items, open_questions), model picker, export buttons (copy JSON / copy Markdown).

### 7.4 Streaming UX

- On click `Summarize`, summary card fields show skeleton placeholders sized to likely content.
- As `partial` events arrive, each field populates in place with content.
- Arrays grow row-by-row. A subtle emerald caret blinks at the current append point.
- No layout shift: skeleton heights reserve space matching median content.
- On `done` event: skeletons drop, copy/export buttons fade in.

### 7.5 BYO-key dialog

- Opened by settings gear at bottom-left of `/app`.
- Two fields: Anthropic key, OpenAI key.
- Stored in `localStorage` under `ms.keys.v1`. Never persisted server-side, never logged.
- Dialog copy: *"Keys stay in your browser. They are sent server-side only when you request a summary, used once to call the provider on your behalf, and not stored."*
- When at least one key is present, the rate-limit badge in the tool changes to `unlimited — your key`.

### 7.6 Component tree

```
components/
├── ui/                          # shadcn primitives (button, dialog, select, badge, textarea, toast)
├── Header.tsx
├── landing/
│   ├── Hero.tsx
│   ├── WhyStructured.tsx
│   └── JsonPreview.tsx
├── app/
│   ├── TranscriptInput.tsx
│   ├── SamplePicker.tsx
│   ├── SummarizeButton.tsx
│   ├── ModelPicker.tsx
│   ├── KeyDialog.tsx
│   ├── SummaryCard.tsx
│   ├── SummaryField.tsx
│   ├── DecisionsList.tsx
│   ├── ActionItemsList.tsx
│   ├── ActionItemRow.tsx
│   ├── OpenQuestionsList.tsx
│   ├── ExportMenu.tsx
│   └── StreamingCaret.tsx
```

### 7.7 Client hook

`src/lib/hooks/useSummaryStream.ts` — consumes SSE via `fetch` + `ReadableStream` (not `EventSource`, which cannot POST). Exposes:

```ts
{
  partial: PartialSummary;
  final: Summary | null;
  error: { message: string; code?: string } | null;
  status: "idle" | "streaming" | "done" | "error";
  start: (args: { transcript: string; model?: string }) => void;
  abort: () => void;
}
```

## 8. Sample transcripts

Three hand-authored transcripts, 600–900 words each, committed at `src/lib/samples/`:

1. **product-kickoff.ts** — PM + design + eng aligning on Q2 feature. Clear decisions, named owners, one open question.
2. **eng-standup.ts** — standup with blockers, a brief incident retrospective line, due dates.
3. **customer-call.ts** — sales discovery. Softer decisions, more open questions.

These triple as fixtures for the schema-contract test (paired with `src/lib/__tests__/fixtures/golden/*.json`).

## 9. Testing

Vitest only. No Playwright in MVP.

### 9.1 Files

```
src/lib/__tests__/
├── schema.test.ts              # Zod parses known-good; rejects known-bad
├── partial-json.test.ts        # streaming fragments → expected partials
├── adapters.test.ts            # mocked provider HTTP → verifies glue
└── fixtures/
    └── golden/
        ├── product-kickoff.json
        ├── eng-standup.json
        └── customer-call.json
```

### 9.2 Schema contract test

For each sample, the adapter test:
1. Mocks the provider HTTP with a canned response stream that emits the golden output.
2. Runs the adapter end-to-end.
3. Asserts the final `StreamResult.final` deep-equals the golden file.

Changing the schema breaks the goldens — an intentional forcing function, matching the README's claim.

### 9.3 HTTP mocking

`undici` MockAgent (works in Edge-compatible code and Node tests) or `msw/node`. Prefer MockAgent for lower dependency surface.

### 9.4 Not tested

- Live provider calls (flaky, cost money, would gate CI on secrets).
- UI components beyond smoke-level render.
- Rate limiting (trivial wrapper around a well-tested library).

## 10. Developer experience

- Package manager: `pnpm` (matches README claim).
- Lint + format: **Biome** (single tool, fast, no ESLint+Prettier+plugins).
- TypeScript strict mode.
- Husky + lint-staged: format staged files on commit.
- Scripts: `dev`, `build`, `start`, `test`, `test:watch`, `lint`, `lint:fix`, `typecheck`.

## 11. CI / deploy

### 11.1 GitHub Actions

Single workflow `.github/workflows/ci.yml`:
- Triggers: push to `main`, PRs to `main`.
- Steps: checkout → setup pnpm → `pnpm install --frozen-lockfile` → `pnpm lint` → `pnpm typecheck` → `pnpm test` → `pnpm build`.
- No secrets required — tests use mocks.

### 11.2 Vercel

- No `vercel.json` needed; defaults fit.
- Required env vars on Vercel: `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`.
- Optional env vars on Vercel: `UPSTASH_REDIS_REST_URL`, `UPSTASH_REDIS_REST_TOKEN` (if omitted, rate limiting is disabled and the owner bears any abuse).
- Without any keys set, the app still loads but BYO-key is required to summarize anything, and samples without owner keys will return a clear 400.

## 12. Repo structure (final)

```
meetingscribe/
├── .github/workflows/ci.yml
├── .env.example
├── biome.json
├── docs/
│   └── superpowers/specs/2026-04-22-meetingscribe-design.md
├── public/
│   └── (assets, og image)
├── src/
│   ├── app/
│   │   ├── layout.tsx
│   │   ├── globals.css
│   │   ├── page.tsx                   # landing
│   │   ├── app/page.tsx               # tool
│   │   └── api/summarize/route.ts     # SSE endpoint, Edge runtime
│   ├── components/                    # see 7.6
│   ├── lib/
│   │   ├── llm/
│   │   │   ├── types.ts
│   │   │   ├── schema.ts
│   │   │   ├── prompt.ts
│   │   │   ├── anthropic.ts
│   │   │   ├── openai.ts
│   │   │   └── index.ts
│   │   ├── hooks/useSummaryStream.ts
│   │   ├── samples/
│   │   │   ├── product-kickoff.ts
│   │   │   ├── eng-standup.ts
│   │   │   └── customer-call.ts
│   │   ├── rate-limit.ts
│   │   ├── validate.ts
│   │   └── export.ts                  # toMarkdown(summary)
│   └── lib/__tests__/                 # see 9.1
├── LICENSE
├── README.md                          # rewritten to match reality
├── package.json
├── pnpm-lock.yaml
└── tsconfig.json
```

## 13. README revisions (honesty pass)

The existing README overstates the MVP. Specific edits during implementation:

**Remove:**
- Audio upload and Whisper (cloud + local).
- Supabase auth, Postgres, Storage sections.
- Resend email delivery.
- Docker Compose self-host.
- Per-meeting cost table (no transcription path in MVP).
- Speaker diarization, calendar integration, Slack bot, retrieval, fine-tuned extraction, offline LLM.

**Move to Roadmap:**
- All of the above (except the things that are permanently out of scope for a portfolio piece).

**Sharpen and keep:**
- "Why structured extraction matters" pitch.
- Architecture diagram (simplified — drop Supabase and Whisper boxes).
- Stack table (trimmed to what actually ships).
- Claude tool-calling code sample.

**Add:**
- OpenAI side of the adapter (dual-provider is committed).
- Streaming extraction story.
- BYO-key pattern.
- "What this repo is (and isn't)" section up top — honest framing: *"a focused proof-of-craft around structured LLM output. Roadmap items are real but deliberately deferred."*

**Keep unchanged:**
- MIT license.
- Contact section.
- Demo GIF reference (owner records after build).

## 14. Deliberate non-goals

Documented here so they don't creep back in:

- No accounts, no history, no save button.
- No theme toggle.
- No share-by-link.
- No summary editing — structured output is not a doc editor.
- No analytics/telemetry in MVP.
- No i18n.
- No admin panel.
- No mobile-native considerations beyond responsive layout.

## 15. Open implementation questions (resolve during planning)

- Exact Biome config ruleset.
- Whether to include a minimal shadcn `toast` primitive or use a simpler inline banner for errors.
- Whether the `partial-json` package or a small hand-rolled parser is the better dependency choice. (Default: `partial-json`, unless it has issues.)
- Exact streaming event naming if SSE proves fragile on Edge — fall back to newline-delimited JSON over a plain fetch stream.

These are minor and can be decided during implementation without changing the architecture.
