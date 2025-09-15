# 1) Product truth & non-goals

* **What Prim does:** joins meetings as a participant, **listens**, **keeps timeboxes**, **answers when asked**, and **asks for the floor before speaking** (panel/notification; optional single “ding”).
* **What Prim avoids:** barge-ins, overwrought avatars (until later), heavy infra in v0, vendor lock-in in core.

# 2) North-star UX (cross-platform)

* **In-meeting side panel** (Zoom App / Teams Meeting Tab / Meet Add-on): “Active Notes” for everyone; **Command Center** for managers/owners only.
* **Floor request flow:** banner/notification first; if no response after N seconds and silence is detected, **1x soft chime**; speak only on **Allow** (or per org policy).
* **Talk policy:** short, targeted utterances (≤7s). Longer answers go to the panel as cards.

# 3) Platform capability summary

(Keep tables to keywords/numbers per your instruction.)

### Media (send/receive) — target operating points

| Platform            | Audio in | Audio out                 | Sample rate (target) | Frame dur | Notes                                                             |
| ------------------- | -------- | ------------------------- | -------------------- | --------- | ----------------------------------------------------------------- |
| Zoom Video SDK      | ✔︎       | ✔︎                        | 44.1–48 kHz          | 10–20 ms  | Raw PCM 16-bit; mono recommended; SDK encodes to Opus/room mix    |
| Teams RTP Bot       | ✔︎       | ✔︎                        | 16 kHz               | 20 ms     | PCM 16-bit into bot; platform encodes (SILK/G.722); strict timing |
| Meet Media API (DP) | ✔︎       | ⚠︎ (tenant/feature-gated) | 48 kHz               | \~20 ms   | Virtual streams; enrollments/limits apply                         |

### In-meeting side panels

| Platform    | Panel | Modes                          | WS allowed?\*                                   | Auth handoff                       |
| ----------- | ----- | ------------------------------ | ----------------------------------------------- | ---------------------------------- |
| Zoom App    | ✔︎    | Collapsed / Expanded / Pop-out | Varies (allow-list/CSP); keep SSE/poll fallback | Zoom App context → backend JWT     |
| Teams Tab   | ✔︎    | Side panel / Stage             | ✔︎                                              | SSO w/ Teams SDK → backend session |
| Meet Add-on | ✔︎    | Side panel                     | ✔︎                                              | Add-on identity → backend session  |

\*Zoom Apps networking varies by tenant/config; keep a graceful fallback to SSE or short-polling.

# 4) System architecture (bird’s-eye)

```
                    ┌────────────────────────┐
                    │  In-meeting Panels     │  Zoom App / Teams Tab / Meet Add-on
                    │  (Active Notes, CC)    │
                    └─────────┬──────────────┘
                              │ HTTPS + WS/SSE (capability-aware)
                        ┌─────▼─────┐
                        │ Gateway   │  (Fastify/NestJS)
                        │  REST/WS  │  Auth, RBAC, webhooks, policy
                        └─────┬─────┘
     ┌────────────────────────┼─────────────────────────┐
     │                        │                         │
┌────▼─────┐            ┌─────▼─────┐            ┌──────▼─────┐
│ Zoom     │            │ Teams     │            │ Meet        │
│ Bot Adap │            │ Bot Adap  │            │ Bot Adap    │
│ (Video   │            │ (RTP Bot) │            │ (Media API) │
└────┬─────┘            └─────┬─────┘            └──────┬─────┘
     │   Raw A/V + events      │                        │
     └───────────────┬─────────┴──────────────┬─────────┘
                     │                        │
                ┌────▼─────┐           ┌─────▼────┐
                │ Voice    │           │ Notes/RAG│
                │ Pipeline │           │ Services │
                │ (Python  │           │ (TS/Py)  │
                │  Agent)  │           └──────────┘
                └────┬─────┘
                     │
           STT ⇆ LLM Policy ⇆ TTS
                     │
                ┌────▼─────┐
                │  Storage │  Postgres + S3 + Meilisearch + Yjs awareness
                └──────────┘
```

# 5) Code organization (monorepo, Yarn)

```
prim/
  packages/
    core/                 # tiny ports (interfaces), events, capability manifest, zod schemas
    speech/               # OpenAI-compatible client: adapters for STT/TTS/LLM
    adapters/
      zoom/
        bot/              # Video SDK adapter (Node)
        ui/               # Zoom App wrapper (panel mount API)
      teams/
        bot/              # RTP media bot
        ui/               # Meeting tab wrapper
      meet/
        bot/              # Media API adapter (feature-flagged)
        ui/               # Add-on wrapper
    ui/                   # shared React (Active Notes, CC widgets)
    yjs-transport/        # WS/SSE/WebTransport provider; awareness/presence glue
  apps/
    web/                  # Next.js (auth, orgs, settings, meeting records, exports)
    gateway/              # Fastify/NestJS REST/WS; auth, RBAC, webhooks
    agent/                # Python voice workers (LiveKit-Agents-style runtime)
```

**Why split UI and bot per platform?** Different SDKs, auth, release cadences. Shared **ports** keep the product logic agnostic.

# 6) Core contracts (keep them tiny)

### Media bots

```ts
export interface MeetingBotAdapter {
  id: 'zoom'|'teams'|'meet';
  capabilities(): {
    audioIn: boolean; audioOut: boolean;
    handSignal: 'native'|'notification'|'panel';
    panel: boolean; transcripts: { live: boolean; post: boolean };
  };
  join(desc: JoinDescriptor): Promise<ActiveSession>;
}

export interface ActiveSession {
  onAudio(cb: (pcm: Float32Array, tsMs: number) => void): void; // room → agent
  speak(pcm: Float32Array): Promise<void>;                      // agent → room
  onParticipantEvent(cb: (e: ParticipantEvent) => void): void;
  leave(): Promise<void>;
}
```

### In-meeting panels

```ts
export interface InMeetingUIAdapter {
  mount(el: HTMLElement, ctx: { meetingId: string; role: Role }): UnmountFn;
  onCommand(cb: (cmd: UICommand) => void): void; // SAY_NEXT, TIMEBOX, CREATE_ACTION
  emit(event: UIEvent): void;                    // notes diffs, timers, floor state
}
```

### Floor request + chime

```ts
export interface FloorRequest {
  request(reason?: string): Promise<void>; // banner/notification
  withdraw(): Promise<void>;
}
// helper
export async function playDing(session: ActiveSession) { /* synth 200ms dual-tone and speak() */ }
```

# 7) Voice pipeline (Python-first worker)

* **Turn taking:** VAD + basic diarization + RMS “barge-in” guard.
* **Intent triggers:** wake phrase (“Prim?”) and panel/CC commands; also detect agenda drift/timebox breach.
* **STT:** hosted streaming (stable partials + word timestamps) with backpressure; **fallback:** local Whisper.cpp.
* **Policy/LLM:** short-form generation (≤7s audio). Keep a “fast model” path for live interjections; defer longer summaries to panel.
* **TTS:** streaming synthesis; first audio chunk <200 ms; viseme stream later for avatar v1.
* **Backpressure:** if TTS lags, compress utterance or push to panel.

*Implementation hints:* structure workers like LiveKit Agents (room observers + pipelines) without binding the whole system to one vendor. The bot adapters push PCM frames into the agent; the agent emits PCM back via `session.speak()`.

# 8) TTS choices (open-model path, practical now)

* **Default (tiny / good quality / easy):** **Kokoro** 82M — run it first for “always-on” low-latency speech.
* **Higher-quality options behind the same interface:** **Chatterbox** (great naturalness & emotion), **Orpheus** (low latency, multilingual), **Dia** (dialogue & non-verbal cues; heavier).
* **Runtime:** Python gRPC or HTTP microservice with streaming chunk API; TS client wraps all behind `speech.tts.synthesizeStream()`.

**Operational defaults**

* Sample rate: 24 kHz output from TTS; resample to platform target (48 kHz for Zoom/Meet, 16 kHz Teams).
* Cold start: pre-warm model; keep N workers hot per node.
* Safety: profanity pass-through (manager can toggle); duration cap (≤7s).

# 9) STT choices (hosted + local)

* **Hosted primary:** pick a provider with **stable partials + word timestamps** and WebSocket streaming; budget for 1 hr/day/user.
* **Local fallback:** **whisper.cpp** with low-latency flags (16 kHz mono, small context, beam=1, partials on). Expose as a local service the desktop app can invoke when privacy mode is on.
* **Abstraction:** `speech.stt.stream()` returns partials+finals with char offsets and word timings; identical for hosted/local.

# 10) Notes & collaboration

* **Active Notes:** Yjs CRDT document; awareness for cursors/presence; minimal payload diffs to keep panel snappy.
* **Transport:** WS when permitted; else SSE fallback; reconnect with exponential backoff and snapshot re-sync.
* **RAG:** shard transcripts and decisions by meeting; Meilisearch for keywords now; embeddings (Qdrant) later.

# 11) Data model (Postgres + S3)

* **Meetings:** id, platform, tenant, start/end, participants.
* **Utterances:** ts, speaker, text, audio\_ref, policy (“answer\_on\_ask” / “timebox\_nudge”).
* **Decisions / Action items:** owner, due, linkage to utterances/snippets.
* **Artifacts:** transcripts, audio chunks, exports (S3).
* **Audit:** every Prim utterance (text→speech), floor requests, chimes, CC actions.
* **RBAC:** org → workspace → role (`owner`,`manager`,`member`,`guest`).

# 12) Security & privacy

* **Consent surfaces:** “Assistant present” + “May speak” banners; org-policy modes: *coach-only*, *answer-on-ask*, *proactive timekeeper*.
* **Secrets:** per-adapter credentials; tenant-scoped; rotated.
* **At rest:** row-level encryption for transcripts; selective redaction for exports.
* **Network:** allow-listed domains for Zoom Apps; panels never expose secrets; always server-mediated signed URLs.
* **Recording awareness:** tag chimes and agent utterances; make them obvious in post-meeting recap.

# 13) Latency budgets (end-to-end target)

| Stage                       | Budget                                           |
| --------------------------- | ------------------------------------------------ |
| Capture → STT first partial | ≤ 250 ms (hosted) / ≤ 500 ms (local whisper.cpp) |
| Policy (short reply)        | ≤ 300–400 ms                                     |
| TTS first chunk             | ≤ 200 ms                                         |
| Adapter inject              | ≤ 100 ms                                         |
| **Total to first audio**    | **≤ 900 ms** (hosted) / **≤ 1.2 s** (local)      |

Operational tricks: pre-warm TTS, pre-compute “expected” phrases (e.g., timebox nudges), keep replies **short**.

# 14) Reliability, scaling, & backpressure

* **Bot adapters** are stateless processes; **Agent workers** scale horizontally with a simple “room assignment” queue.
* **Backpressure:** if STT backlog grows, downsample / throttle; if TTS backlog grows, shorten reply or route to panel only.
* **Failure modes:** platform audio-out down? → fall back to **panel text** + **host-only coach audio**; WS blocked? → SSE/poll; Meet audio-out gated? → panel only.
* **Regioning:** run media bots close to platform edge (per region); pin agent workers to same region for <100 ms hop.

# 15) Observability & SLOs

* **Metrics:** latency to first word, utterance duration, floor-request acceptance rate, chime count, STT/TTS errors.
* **Tracing:** join→speak spans; correlate panel actions to bot utterances.
* **Logs:** utterance text, policy reason, adapter ids; PII scrubbing on export.
* **SLOs:** 99% of short utterances start <1 s; <0.5% failed joins; <1% duplicate chimes per 60 min.

# 16) Testing strategy

* **Contract tests** per adapter: join, receive N frames, `speak()` audible loopback, participant events, “floor request” path.
* **Panel contract tests:** mount/unmount; auth handshake; command emission; reconnection under WS drop.
* **Latency harness:** replay corpora; measure full pipeline wall-clock.
* **Chaos:** kill STT/TTS mid-utterance; ensure graceful degrade to panel text.

# 17) Deployment & release strategy

### v0.0.1 **OSS**

* Docker Compose: `gateway`, `agent`, Postgres, MinIO, Meilisearch.
* **Zoom only**: Zoom App panel + ZoomBotAdapter join/speak; floor-request banners; optional **ding** (room/host-only toggles).
* Minimal web: org/roles, meeting list, audit, notes, exports.

### v1 **(OSS + SaaS wrapper)**

* Add **Teams** bot + tab; **Meet** add-on + bot (feature-flagged).
* Multi-tenant isolation, billing, metrics, alerts; keep **core** identical.

# 18) Risk register & defaults

| Risk                              | Mitigation / Default                                                                        |
| --------------------------------- | ------------------------------------------------------------------------------------------- |
| Platform API drift (Teams/Meet)   | Adapters isolated; semver-pinned; contract tests gate deploy                                |
| Zoom App WS blocked by tenant CSP | yjs-transport supports SSE/poll; keep payloads tiny; backoff                                |
| TTS stalls                        | duration caps; pre-baked phrases; downgrade to Kokoro “tiny” or send text                   |
| Echo/feedback                     | bot audio is server-side (no AEC needed); never loop back room mix; RMS guard before “ding” |
| Over-talking                      | silence detector; host gating; policy “answer-on-ask” default                               |
| Cost spikes (STT/TTS hosted)      | 1-hour/day/user budget; local whisper.cpp fallback; nightly batch for heavy summaries       |

# 19) Initial interfaces & helpers (ready to code)

### TypeScript: capability manifest & floor request

```ts
export type CapabilityManifest = {
  audioIn: boolean; audioOut: boolean;
  handSignal: 'native'|'notification'|'panel';
  panel: boolean; transcripts: { live: boolean; post: boolean };
};

export async function requestFloor(session: ActiveSession, reason?: string) {
  // 1) panel/notification
  // 2) 6s timer → if no Allow and silence window, optional playDing(session)
  // 3) on Allow, stream TTS → session.speak()
}
```

### Tiny “ding” synthesis (TS)

```ts
export function synthDing(sr=48000, ms=200, f1=660, f2=990) {
  const N = Math.floor(sr * ms / 1000), atk = Math.floor(sr*0.005), rel = Math.floor(sr*0.04);
  const pcm = new Float32Array(N);
  for (let n=0;n<N;n++){ const t=n/sr, env=n<atk?n/atk:n>N-rel?(N-n)/rel:1;
    pcm[n]=0.2*env*(Math.sin(2*Math.PI*f1*t)+0.5*Math.sin(2*Math.PI*f2*t)); }
  return pcm;
}
```

### Python voice loop (sketch)

```python
for audio in room.stream_audio():                # from adapter
    stt.feed(audio)
    if stt.has_partial(): update_panel_partial(stt.partial())
    intent = policy.consider(stt, agenda, timers)
    if intent.should_request_floor:
        floor.request(reason=intent.reason)
    if intent.allowed_to_speak:
        text = llm.reply(intent, context)
        for chunk in tts.stream(text):           # low-latency TTS
            room.speak(chunk)                    # to adapter
```

# 20) Milestones & acceptance

**M1 – Zoom MVP (OSS v0.0.1)**

* Joins Zoom; **floor request** banner; optional single **ding**; speaks short responses with latency <1 s.
* Panel: Active Notes to all; Command Center to managers; timeboxes/drift alerts; action items persisted.
* Audit: every Prim utterance; exportable recap.

**M2 – Teams parity**

* Teams tab; RTP bot send/receive; same floor-request pattern via in-meeting notification.

**M3 – Meet parity**

* Meet add-on panel; Media API bot for enrolled tenants; clean fallback to panel-only when outbound audio is gated.


---

# Appendix A — Data & ORM Plan (Postgres + Drizzle)

## A.1 Goals (why this approach)

* Strong TypeScript types end-to-end.
* Simple, explicit SQL you can tune.
* First-class migrations.
* Append-heavy, auditable writes (utterances, actions).
* Clear separation: **Gateway owns persistence**; **Python agent never hits the DB**.

## A.2 Decision

* **ORM/query builder:** **Drizzle ORM** (Postgres).
* **Migrations:** `drizzle-kit` (SQL migrations checked in).
* **Connection:** `pg` with prepared statements; add **pgBouncer** when needed.
* **Search:** Meilisearch for keywords; embeddings later.
* **Blobs:** S3-compatible (e.g., MinIO in OSS).

---

## A.3 Repo layout (data layer)

```
prim/
  packages/
    schemas/                 # Drizzle schema + zod shapes
    core/                    # Repo interfaces (ports), DTOs, events
    db/                      # Drizzle runtime + migrations + helpers
  apps/
    gateway/                 # API that calls repos (owns persistence)
    agent/                   # Python voice workers (no DB access)
```

---

## A.4 Install & configure

### Dependencies

```bash
# in repo root
pnpm add -w drizzle-orm pg dotenv zod
pnpm add -w -D drizzle-kit @types/pg tsx
```

### `packages/db/drizzle.config.ts`

```ts
import { defineConfig } from "drizzle-kit";
export default defineConfig({
  dialect: "postgresql",
  schema: "./packages/schemas/src/*.{ts,tsx}",
  out: "./packages/db/migrations",
  dbCredentials: {
    url: process.env.DATABASE_URL!, // e.g. postgres://user:pass@host:5432/prim
  },
  strict: true,
});
```

### `packages/db/src/client.ts`

```ts
import { drizzle } from "drizzle-orm/node-postgres";
import { Pool } from "pg";

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: Number(process.env.DB_POOL_MAX ?? 10),
  ssl: /localhost|127\.0\.0\.1/.test(process.env.DATABASE_URL ?? "") ? false : { rejectUnauthorized: false },
});

export const db = drizzle(pool, { logger: process.env.DB_LOG === "1" });
```

### `package.json` (workspace scripts)

```json
{
  "scripts": {
    "db:gen": "drizzle-kit generate --config=packages/db/drizzle.config.ts",
    "db:migrate": "drizzle-kit migrate --config=packages/db/drizzle.config.ts",
    "db:studio": "drizzle-kit studio --config=packages/db/drizzle.config.ts"
  }
}
```

---

## A.5 Schema (tables & types)

> Notes: `timestamptz` everywhere; ULIDs for external ids; referential integrity; sensible indexes.

### `packages/schemas/src/db.ts`

```ts
import { pgTable, varchar, text, timestamp, boolean, integer, jsonb, uuid, index, unique, primaryKey } from "drizzle-orm/pg-core";
import { relations } from "drizzle-orm";

// Helpers
const ulid = (name: string) => varchar(name, { length: 26 }); // ULID string

export const orgs = pgTable("orgs", {
  id: ulid("id").primaryKey(),
  name: text("name").notNull(),
  createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
});

export const workspaces = pgTable("workspaces", {
  id: ulid("id").primaryKey(),
  orgId: ulid("org_id").notNull().references(() => orgs.id, { onDelete: "cascade" }),
  name: text("name").notNull(),
  createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
}, (t) => ({
  byOrg: index("workspaces_org_idx").on(t.orgId),
}));

export const users = pgTable("users", {
  id: ulid("id").primaryKey(),
  orgId: ulid("org_id").notNull().references(() => orgs.id, { onDelete: "cascade" }),
  email: text("email").notNull(),
  displayName: text("display_name").notNull(),
  role: text("role").$type<"owner"|"manager"|"member"|"guest">().notNull(),
  createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
}, (t) => ({
  uniqEmailOrg: unique().on(t.orgId, t.email),
}));

export const meetings = pgTable("meetings", {
  id: ulid("id").primaryKey(),
  orgId: ulid("org_id").notNull().references(() => orgs.id, { onDelete: "cascade" }),
  workspaceId: ulid("workspace_id").notNull().references(() => workspaces.id, { onDelete: "cascade" }),
  platform: text("platform").$type<"zoom"|"teams"|"meet">().notNull(),
  externalMeetingId: text("external_meeting_id").notNull(),
  startedAt: timestamp("started_at", { withTimezone: true }).notNull(),
  endedAt: timestamp("ended_at", { withTimezone: true }),
}, (t) => ({
  byOrgWs: index("mtg_org_ws_idx").on(t.orgId, t.workspaceId),
  extKey: unique().on(t.platform, t.externalMeetingId),
}));

export const participants = pgTable("participants", {
  meetingId: ulid("meeting_id").notNull().references(() => meetings.id, { onDelete: "cascade" }),
  participantId: ulid("participant_id").notNull(), // internal id
  externalUserId: text("external_user_id"),       // zoom/teams/meet id
  displayName: text("display_name"),
}, (t) => ({
  pk: primaryKey({ columns: [t.meetingId, t.participantId] }),
  byMeeting: index("parts_meeting_idx").on(t.meetingId),
}));

export const utterances = pgTable("utterances", {
  id: ulid("id").primaryKey(),
  meetingId: ulid("meeting_id").notNull().references(() => meetings.id, { onDelete: "cascade" }),
  ts: timestamp("ts", { withTimezone: true }).notNull(),
  speaker: text("speaker").notNull(), // "prim" or participantId
  text: text("text").notNull(),
  policy: text("policy").$type<"answer_on_ask"|"timebox_nudge"|"coach_only">().notNull(),
  audioRef: text("audio_ref"), // s3://...
  isChime: boolean("is_chime").default(false).notNull(),
}, (t) => ({
  byMeetingTs: index("utt_meeting_ts_idx").on(t.meetingId, t.ts),
}));

export const decisions = pgTable("decisions", {
  id: ulid("id").primaryKey(),
  meetingId: ulid("meeting_id").references(() => meetings.id, { onDelete: "set null" }),
  title: text("title").notNull(),
  detail: text("detail"),
  createdBy: ulid("created_by").references(() => users.id, { onDelete: "set null" }),
  createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
}, (t) => ({
  byMeeting: index("dec_meeting_idx").on(t.meetingId),
}));

export const actions = pgTable("actions", {
  id: ulid("id").primaryKey(),
  meetingId: ulid("meeting_id").references(() => meetings.id, { onDelete: "set null" }),
  title: text("title").notNull(),
  owner: ulid("owner").references(() => users.id, { onDelete: "set null" }),
  due: timestamp("due", { withTimezone: true }),
  status: text("status").$type<"open"|"done"|"blocked">().default("open").notNull(),
  createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
}, (t) => ({
  byOwner: index("act_owner_idx").on(t.owner),
  byMeeting: index("act_meeting_idx").on(t.meetingId),
}));

export const artifacts = pgTable("artifacts", {
  id: ulid("id").primaryKey(),
  meetingId: ulid("meeting_id").references(() => meetings.id, { onDelete: "set null" }),
  kind: text("kind").$type<"transcript"|"audio"|"export"|"notes">().notNull(),
  uri: text("uri").notNull(),
  meta: jsonb("meta"),
  createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
}, (t) => ({
  byMeetingKind: index("art_meeting_kind_idx").on(t.meetingId, t.kind),
}));

export const audit = pgTable("audit", {
  id: uuid("id").defaultRandom().primaryKey(),
  orgId: ulid("org_id").notNull(),
  ts: timestamp("ts", { withTimezone: true }).defaultNow().notNull(),
  actor: text("actor").notNull(),     // 'system'|'prim'|userId
  event: text("event").notNull(),     // 'floor.request'|'ding'|'speak'|'cc.action'...
  payload: jsonb("payload").notNull()
}, (t) => ({
  byOrgTs: index("audit_org_ts_idx").on(t.orgId, t.ts),
}));
```

---

## A.6 Migrations

### Generate & apply

```bash
# read schema & emit SQL
pnpm db:gen

# apply to DATABASE_URL
pnpm db:migrate
```

### Zero-downtime pattern

* Additive changes first (new table/column **nullable** / defaulted).
* Backfill with idempotent script.
* Switch reads/writes.
* Drop old column in a later migration.

### Rollback

* Keep a down migration next to each up.
* Never drop data without a snapshot (see backups).

---

## A.7 Repository interfaces (ports)

### `packages/core/src/repos/utterances.ts`

```ts
export interface UtteranceRepo {
  append(u: {
    meetingId: string; ts: Date; speaker: string; text: string;
    policy: "answer_on_ask"|"timebox_nudge"|"coach_only";
    audioRef?: string; isChime?: boolean;
  }): Promise<void>;

  appendBatch(rows: Array<{
    meetingId: string; ts: Date; speaker: string; text: string;
    policy: "answer_on_ask"|"timebox_nudge"|"coach_only";
    audioRef?: string; isChime?: boolean;
  }>): Promise<number>;

  listForMeeting(meetingId: string, limit?: number): Promise<Array<{
    id: string; ts: Date; speaker: string; text: string; policy: string; audioRef?: string;
  }>>;
}
```

### Drizzle implementation

```ts
// packages/db/src/repos/utterances.drizzle.ts
import { db } from "../client";
import { utterances } from "@prim/schemas/db";
import { sql } from "drizzle-orm";
import type { UtteranceRepo } from "@prim/core/repos/utterances";

export function makeUtteranceRepo(): UtteranceRepo {
  return {
    async append(u) {
      await db.insert(utterances).values({
        id: crypto.randomUUID().replace(/-/g, "").slice(0,26), // quick ULID substitute if needed
        ...u,
      });
    },
    async appendBatch(rows) {
      if (!rows.length) return 0;
      await db.execute(sql`
        WITH src AS (
          SELECT * FROM jsonb_to_recordset(${JSON.stringify(rows)}::jsonb)
          AS x(meeting_id text, ts timestamptz, speaker text, text text, policy text, audio_ref text, is_chime bool)
        )
        INSERT INTO utterances (id, meeting_id, ts, speaker, text, policy, audio_ref, is_chime)
        SELECT lpad(encode(gen_random_bytes(13),'hex'),26,'0'), meeting_id, ts, speaker, text, policy, audio_ref, coalesce(is_chime,false)
        FROM src;
      `);
      return rows.length;
    },
    async listForMeeting(meetingId, limit = 200) {
      return db.select().from(utterances)
        .where(utterances.meetingId.eq(meetingId))
        .orderBy(utterances.ts.asc())
        .limit(limit);
    },
  };
}
```

> For real ULIDs, add a small ulid lib; or generate in app and pass into `values`.

---

## A.8 Transactions & concurrency

* **Isolation:** `READ COMMITTED` default; bump to `REPEATABLE READ` for multi-step updates.
* **Advisory locks** for single-flight jobs (e.g., one recap per meeting).

```ts
await db.execute(sql`SELECT pg_advisory_xact_lock(${meetingIdHash})`);
```

* **Optimistic concurrency**: add `version int` column to rows users can edit; check `version = version + 1`.

---

## A.9 Tenancy & RLS (optional hardening)

**Column strategy:** every table ties to `org_id` or joins through a parent with `org_id`.

**Enable RLS** and policies (sample for `meetings`, repeat pattern):

```sql
ALTER TABLE meetings ENABLE ROW LEVEL SECURITY;

-- role 'app_user' set on connections from gateway
CREATE POLICY meetings_isolation ON meetings
USING (org_id = current_setting('app.current_org', true));

-- set org id per request in gateway
SELECT set_config('app.current_org', $1, true);
```

> Keep RLS behind a feature flag early on; it’s powerful but adds debugging complexity.

---

## A.10 Security & privacy

* **At rest encryption:** use cloud-encrypted disk + `pgcrypto` for specific PII.

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;
-- Example: encrypted email (if needed)
-- INSERT uses pgp_sym_encrypt(email, :key); SELECT uses pgp_sym_decrypt(...)
```

* **S3 objects:** server-side encryption (SSE-S3/KMS), pre-signed URLs from the gateway.
* **Redaction:** redact PII in exports; audit which fields were redacted.
* **Secrets:** no secrets in panels; panels call gateway → gateway signs pre-signed URLs.

---

## A.11 Performance & indexing

* **Hot paths:** append utterances (batch), list recent utterances per meeting.
* **Indexes**

  * `utterances(meeting_id, ts)`
  * `actions(owner, status)`
  * `artifacts(meeting_id, kind)`
* **Prepared statements:** enabled by default in `pg` pool.
* **Pooling:** start `max=10`; add **pgBouncer** (transaction pooling) when scaling.

---

## A.12 Observability

* **Query logging:** enable `logger` in Drizzle via env in dev.
* **Slow query threshold:** log queries > 200 ms; capture SQL + plan in prod debug.
* **Migrations audit:** record applied migration + app build SHA in an `infra_meta` table.

---

## A.13 Testing & CI

* **Local DB:** Docker `postgres:16`; `pnpm db:migrate` on startup.
* **Integration tests:** spin ephemeral DB, run migrations, seed minimal data.
* **CI:**

  * Step 1: lint/types/tests
  * Step 2: `db:gen` (verify schema compiles)
  * Step 3: `db:migrate` against a disposable DB
  * Step 4: run repo tests that touch DB

---

## A.14 Backups & retention

* **PITR:** enable on managed Postgres (e.g., Neon/Neon PITR or RDS).
* **Nightly export:** `utterances`, `decisions`, `actions` → S3 (CSV/Parquet).
* **Retention policy** (example):

  * Default: transcripts 365 days; audio 90 days.
  * Per-org overrides; hard delete pipeline with audit record of deletion.

---

## A.15 Gateway owns persistence (Python agent pattern)

* Python **never** hits the DB. It sends/receives via the Gateway:

  * `POST /v1/utterances:append`
  * `WS /v1/meetings/:id/notes` (Yjs updates)
  * `POST /v1/audit`

Example Gateway handler (TS):

```ts
// apps/gateway/src/routes/utterances.ts
app.post("/v1/utterances:append", async (req, res) => {
  const body = await req.validate(z.object({
    meetingId: z.string(), ts: z.string().datetime(),
    speaker: z.string(), text: z.string().min(1),
    policy: z.enum(["answer_on_ask","timebox_nudge","coach_only"]),
    audioRef: z.string().optional(), isChime: z.boolean().optional(),
  }));
  const repo = makeUtteranceRepo();
  await repo.append({ ...body, ts: new Date(body.ts) });
  res.status(204).send();
});
```

---

## A.16 Seeds & fixtures

* **Org + workspace + one meeting**
* **Two users** (manager, member)
* **3–5 utterances** + one decision + one action

Seeder (TS):

```ts
import { db } from "@prim/db/client";
import { orgs, workspaces, meetings, utterances } from "@prim/schemas/db";

export async function seed() {
  const orgId = "01HZZZZZZZZZZZZZZZZZZZZZZZ"; // ULID
  await db.insert(orgs).values({ id: orgId, name: "Prim Labs" });
  const wsId = "01HYYYYYYYYYYYYYYYYYYYYYYY";
  await db.insert(workspaces).values({ id: wsId, orgId, name: "Core" });
  const mtgId = "01HXXXXXXXXXXXXXXXXXXXXXXXX";
  await db.insert(meetings).values({
    id: mtgId, orgId, workspaceId: wsId, platform: "zoom",
    externalMeetingId: "zoom:abc123", startedAt: new Date()
  });
  await db.insert(utterances).values({
    id: "01HWWWWWWWWWWWWWWWWWWWWWWW",
    meetingId: mtgId, ts: new Date(), speaker: "prim",
    text: "Kicking off. We’re on a 15-minute timebox.",
    policy: "timebox_nudge", isChime: false
  });
}
```

---

## A.17 Yjs persistence (optional)

For server snapshots or offline export:

* Periodically snapshot the Yjs doc (binary) to `artifacts(kind='notes')`.
* Keep last N snapshots; on reconnect, client prefers live state; snapshots used for recovery.

---

## A.18 Rolling to SaaS

* Keep **schema identical** for OSS/SaaS.
* Add:

  * Multi-tenant limits (PG schemas or org\_id partitioning).
  * Billing tables (`subscriptions`, `usage_events`).
  * Read replicas for analytics; OLTP stays primary.
* Migrations run per tenant (single shared DB with `org_id` + RLS), or per-tenant DBs behind a router — both supported by the repo pattern.

---

## A.19 Appendix code: minimal consumer example

List recent utterances (Gateway service):

```ts
import { makeUtteranceRepo } from "@prim/db/repos/utterances.drizzle";

export async function listRecent(meetingId: string) {
  const repo = makeUtteranceRepo();
  return repo.listForMeeting(meetingId, 200);
}
```

Batch append from an adapter (Gateway service):

```ts
export async function appendRoomUtterances(rows: Array<any>) {
  const repo = makeUtteranceRepo();
  // rows already validated
  return await repo.appendBatch(rows);
}
```

---

## A.20 Operational defaults (copy/paste)

* Pool: `DB_POOL_MAX=10`
* Statement timeout: 10s
* `UTC` only; `timestamptz` only
* ULIDs for ids exposed to clients
* Index review: monthly
* Analyze/vacuum: defaults ok on managed PG; monitor bloat
* Drizzle migrations: **every PR that changes schema includes a generated migration**

---

### TL;DR (what to implement now)

1. Add the schema file, client, and config above.
2. Generate & apply migrations.
3. Implement the `UtteranceRepo` shown.
4. Route all DB access through Gateway (repositories).
5. Add nightly exports and set default retention.
6. (Optional) Turn on RLS once auth/RBAC is stable
