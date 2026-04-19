# MeetingScribe

> AI meeting summaries with structured action items. Upload audio or paste a transcript; get a parseable JSON summary with decisions, owners, and open questions.

**🔗 Live demo:** https://meetingscribe.brightnwokoro.dev
**👤 Built by:** [Bright Nwokoro](https://brightnwokoro.dev) · [hello@brightnwokoro.dev](mailto:hello@brightnwokoro.dev)

![MeetingScribe demo](./docs/demo.gif)

---

## Why this exists

Every team wants AI meeting notes. The market leaders (Otter.ai, Fireflies, Fathom) charge $20+/user/month and hold the transcripts on their infrastructure. Many teams — legal, healthcare, founder-mode startups with sensitive product discussions — can't or won't send their meetings to a SaaS.

MeetingScribe is the self-hostable alternative. Upload audio or paste a transcript, get back a **structured** summary: not free-form prose, but a typed JSON object with decisions, action items (each with owner + optional due date), and open questions. The structured output is the point — downstream workflows (email, Notion, Linear, Jira) can consume it directly without fragile string parsing.

Built as a proof-of-craft: any mid-level developer can call an LLM. A senior pattern is *forcing* the LLM through a JSON schema so the output is always valid and your pipeline never breaks on a weird model mood.

## What it does

- **Inputs:** upload audio (`mp3`, `m4a`, `wav`, up to 500MB) or paste/upload a transcript
- **Transcription:** OpenAI Whisper (cloud) or local Whisper (self-hosted mode)
- **Structured extraction:** Claude with JSON schema tool calling returns:
  - `summary` — 3–5 sentence executive overview
  - `decisions[]` — things the meeting settled
  - `action_items[]` — `{ task, owner, due_date? }` tuples
  - `open_questions[]` — unresolved items surfaced for follow-up
- **Outputs:** copy-to-clipboard JSON, Markdown export, optional email to attendees
- **Auth + storage:** Supabase (email magic links + Postgres + object storage)
- **Self-hostable** end-to-end with a single `docker compose up`

## Architecture

```
  ┌──────────┐    upload    ┌─────────────┐
  │  Next.js │─────────────▶│  Supabase   │
  │   (UI)   │              │  Storage    │
  └──────────┘              └──────┬──────┘
        │                          │
        │ POST /summarize          │ signed URL
        ▼                          ▼
  ┌─────────────┐          ┌─────────────┐      ┌───────────┐
  │  Next.js    │─────────▶│  Whisper    │─────▶│  Claude   │
  │  API route  │ transcribe │  API/local │ prompt│  (tool)   │
  └──────┬──────┘          └─────────────┘      └─────┬─────┘
         │                                             │
         │             ┌───────────────────────────────┘
         ▼             ▼
  ┌──────────────────────┐
  │ Supabase Postgres    │
  │  meetings, summaries │
  └──────────────────────┘
```

- **Next.js** owns the UI + API routes. Acts as both client and server.
- **Supabase** — auth (magic links), Postgres (meetings + summaries), storage (raw audio).
- **Whisper** — OpenAI API for quick wins, swap to local `whisper.cpp` for self-host.
- **Claude** — Sonnet for extraction. Tool calling with strict JSON schema enforces output shape.

## Stack

| Layer          | Tech                                                      |
| -------------- | --------------------------------------------------------- |
| Frontend       | Next.js 14, React 18, TypeScript, Tailwind, shadcn/ui     |
| API            | Next.js Route Handlers (Node runtime)                     |
| Auth           | Supabase Auth (magic links)                               |
| Database       | Supabase Postgres 16                                      |
| Storage        | Supabase Storage (S3-compatible)                          |
| Transcription  | OpenAI Whisper API (default) · `whisper.cpp` (self-host)  |
| Extraction LLM | Claude Sonnet 4.6 (default) · GPT-4o (fallback)           |
| Email          | Resend (summary delivery)                                 |
| Deploy         | Vercel + Supabase (hosted) or Docker Compose (self-host)  |

## Quick start

```bash
git clone https://github.com/bright-nwokoro/meetingscribe
cd meetingscribe

cp .env.example .env.local
# fill:
#   OPENAI_API_KEY
#   ANTHROPIC_API_KEY
#   NEXT_PUBLIC_SUPABASE_URL
#   NEXT_PUBLIC_SUPABASE_ANON_KEY
#   SUPABASE_SERVICE_ROLE_KEY
#   RESEND_API_KEY

pnpm install
pnpm db:migrate            # pushes schema to Supabase
pnpm dev                   # http://localhost:3000
```

## Environment variables

```bash
# LLMs
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
ANTHROPIC_MODEL=claude-sonnet-4-6          # default extraction model

# Supabase
NEXT_PUBLIC_SUPABASE_URL=https://xxx.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=...
SUPABASE_SERVICE_ROLE_KEY=...

# Email (optional — for sending summaries)
RESEND_API_KEY=re_...
RESEND_FROM_ADDRESS=scribe@brightnwokoro.dev

# Self-host transcription toggle
USE_LOCAL_WHISPER=false                    # true → call local whisper.cpp
LOCAL_WHISPER_URL=http://localhost:9000
```

## Project structure

```
meetingscribe/
├── src/
│   ├── app/
│   │   ├── (app)/
│   │   │   ├── page.tsx                  # upload + paste UI
│   │   │   └── meetings/[id]/page.tsx    # summary view
│   │   ├── api/
│   │   │   ├── upload/route.ts           # signed upload URL
│   │   │   ├── summarize/route.ts        # kick off transcription + extraction
│   │   │   └── email/route.ts            # send summary to attendees
│   │   └── auth/callback/route.ts
│   ├── lib/
│   │   ├── transcribe.ts                 # Whisper wrapper
│   │   ├── extract.ts                    # Claude tool call with schema
│   │   ├── schema.ts                     # shared JSON schema for summary
│   │   └── supabase.ts
│   └── components/
│       ├── UploadZone.tsx
│       ├── SummaryCard.tsx
│       └── ActionItemRow.tsx
├── supabase/
│   └── migrations/
├── docs/
│   ├── DEPLOY.md
│   └── demo.gif
└── README.md
```

## How it works

### 1. Upload

- Client requests a short-lived signed URL via `POST /api/upload`.
- Browser PUTs the audio file directly to Supabase Storage (no bandwidth waste on the server).
- A `meetings` row is inserted with `status: 'uploaded'`.

### 2. Transcribe

- Worker (Next.js API route with longer timeout, or a Supabase Edge Function for self-host) reads the signed URL and calls Whisper.
- Transcript is stored in the `meetings.transcript` column.
- `status` advances to `'transcribed'`.

### 3. Extract (the senior move)

Claude is called with **tool calling**, not free-form prompting. The tool's input schema is the source of truth for the summary shape:

```ts
const summarySchema = {
  name: "record_summary",
  description: "Record a structured summary of the meeting transcript.",
  input_schema: {
    type: "object",
    properties: {
      summary: { type: "string" },
      decisions: {
        type: "array",
        items: { type: "string" },
      },
      action_items: {
        type: "array",
        items: {
          type: "object",
          properties: {
            task:     { type: "string" },
            owner:    { type: "string" },
            due_date: { type: "string", format: "date" },
          },
          required: ["task", "owner"],
        },
      },
      open_questions: {
        type: "array",
        items: { type: "string" },
      },
    },
    required: ["summary", "decisions", "action_items"],
  },
} as const;

const result = await anthropic.messages.create({
  model: process.env.ANTHROPIC_MODEL,
  max_tokens: 2048,
  tools: [summarySchema],
  tool_choice: { type: "tool", name: "record_summary" },
  messages: [{ role: "user", content: transcript }],
});

// result.content[0].input is guaranteed to match the schema.
```

Because `tool_choice` forces the model to call this specific tool, Claude *cannot* return free-form text. The output object is typed, parseable, and survives model swaps.

### 4. Render + export

- Summary is rendered in the UI with inline action items and owners.
- Copy-to-clipboard for JSON and Markdown.
- Optional email via Resend — summary is templated into a clean HTML email addressed to the attendee list.

## Cost profile

Measured on a typical 45-minute meeting (~6,000 transcript tokens):

| Operation                         | Cost (USD)  |
| --------------------------------- | ----------- |
| Whisper transcription (45 min)    | ~$0.27      |
| Claude Sonnet extraction          | ~$0.06      |
| **Per meeting total**             | **~$0.33**  |

A team of 10 running 4 meetings/day is ~$40/month in AI costs. Compare to $20/user × 10 = $200 for hosted competitors, not counting privacy considerations.

## Self-host mode

For air-gapped or privacy-sensitive deployments:

```bash
docker compose up -d
```

Spins up:
- Postgres (with migrations applied)
- `whisper.cpp` server (CPU-friendly; use `large-v3` for best accuracy)
- Next.js app (with `USE_LOCAL_WHISPER=true`)

The only external call is to Claude (or swap to a local model via Ollama — there's an adapter in `src/lib/llm/ollama.ts`).

## Roadmap

- [ ] Speaker diarization (Whisper + pyannote)
- [ ] Calendar integration (Google/Microsoft) to auto-import meetings
- [ ] Slack bot — `/scribe` command posts summary to a channel
- [ ] Retrieval over past meetings (pgvector on summaries)
- [ ] Fine-tuned extraction model on team-specific glossaries
- [ ] Full offline mode via local LLM (Ollama + `llama3.3`)

## Contributing

PRs welcome. Tests in `src/lib/__tests__/` use real mock transcripts to verify the schema contract — if you change the summary shape, update the golden files.

## License

MIT — see [LICENSE](LICENSE).

## Contact

Freelance AI engineering — RAG, chat widgets, AI copilots, end-to-end.
**Email:** hello@brightnwokoro.dev
**Portfolio:** https://brightnwokoro.dev
**Book a call:** https://calendly.com/brightnwokoro/intro
