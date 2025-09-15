# VISION

**Prim** is an open, respectful, real-time **meeting manager**. She joins your Zoom/Teams/Meet calls, keeps timeboxes, answers when asked, and requests the floor (politely) before speaking. Prim is built **open-source first** so teams can inspect, extend, and trust the system that runs their meetings.

## Why we’re pivoting to OSS

* **Trust & transparency.** Meeting audio, decisions, and actionables are core IP. Teams must be able to audit the code path end-to-end.
* **Integrations at the edge.** Every org’s stack is different. OSS lets the community add adapters, actions, and model backends faster than any single vendor.
* **Sustainability.** The core should run on a laptop or a single VM. SaaS is an add-on, not a requirement.
* **Vendor drift resilience.** Zoom/Teams/Meet/LLM/STT/TTS APIs evolve. Adapters keep platform drift contained.

## What we’re building

* **Prim, the facilitator.** Joins as a participant; **listens**, **keeps timeboxes**, **summarizes on request**, **answers questions**, and **asks for the floor** through a banner/notification flow. Optional single **“ding”** chime as a universal cue.
* **In-meeting side panel UI** for each platform: shared **Active Notes** for everyone; a **Command Center** module (permission-gated) for managers/owners to cue lines, adjust timers, and trigger summaries.
* **Voice loop**: low-latency **STT → policy/LLM → TTS** with a strict talk budget (≤7s) to avoid monologues.
* **Open adapters**: Zoom first; Teams + Meet behind flags. Swappable **STT/TTS/LLM** via a tiny port interface.

## Principles

1. **Respect the room.** No barge-ins; request the floor first; short utterances; clear audit trail.
2. **Reality-first architecture.** Capability-aware adapters; degrade gracefully (panel text if audio-out is gated).
3. **Local-first option.** Can run without cloud models (desktop capture + whisper.cpp + local TTS).
4. **Small, sharp interfaces.** Ports over frameworks; adapters keep platform quirks boxed in.
5. **Performance budgets.** First audio under \~1s (hosted STT/TTS), \~1.2s (local).
6. **Security by default.** Consent surfaces, RBAC, encryption, and explicit retention policies.

## Product pillars

* **Real-time facilitation.** Timeboxes, drift detection, “we’re 3 minutes over,” fast Q\&A.
* **Active Notes (CRDT).** Shared, low-latency notes (Yjs) with presence, decisions, and action items.
* **Command Center (pro).** For managers/owners to queue lines, nudge timeboxes, and approve Prim’s floor requests.
* **Memory & recap.** Meeting decisions/actions persisted; recap and exports after the call.
* **Admin & audit.** Clear logs of every floor request, chime, and utterance.

## Architecture (high-level)

* **Frontends**

  * Zoom App / Teams Meeting Tab / Meet Add-on side panels (React/TypeScript, shared components).
  * Minimal web app (Next.js) for auth, orgs/workspaces, meeting records, settings, exports.

* **Server**

  * **Gateway (Fastify/NestJS, TypeScript):** REST/WS, auth/RBAC, webhooks, policy, audit, file pre-signing.
  * **Adapters (TypeScript):**

    * `zoom/bot`: Zoom Video SDK join/recv/speak; floor-request banner; optional chime.
    * `teams/bot`: Teams real-time media bot; in-meeting notifications.
    * `meet/bot`: Meet Media API (Developer Program) with feature flags.
    * `*/ui`: platform panel wrappers for the shared React UI.
  * **Voice workers (Python):** LiveKit-Agents-style structure: VAD/diarization, intent/policy, STT/LLM/TTS streaming loop.

* **Data**

  * **Postgres** (meetings, participants, utterances, decisions, actions, audit).
  * **S3-compatible** object store (audio chunks, exports, snapshots).
  * **Meilisearch** for keywords now; embeddings store later.
  * **Yjs** for Active Notes; WS/SSE/WebTransport provider with reconnect.

* **Speech/AI (swappable)**

  * **STT**: hosted streaming w/ stable partials + timestamps; fallback **whisper.cpp** locally.
  * **TTS**: default **Kokoro 82M** (tiny, fast). Options: **Chatterbox**, **Orpheus**, **Dia**.
  * **LLM**: small fast model for live interjections; bigger model off-path for summaries.

## Public interfaces (stable commitments)

* **Bot adapter port** (join, onAudio, speak, participant events, capability manifest).
* **UI adapter port** (mount, UI→command, server→events).
* **Speech ports**: `stt.stream()`, `tts.synthesizeStream()`, `llm.complete()` with uniform streaming envelopes.
* **Event/webhook bus**: `meeting.created`, `agenda.started`, `floor.requested`, `utterance.spoken`, `decision.recorded`, `action.created`.

*We follow SemVer: breaking changes to public ports bump major. Adapters live under their own version gates.*

## Data & persistence (OSS-friendly)

* **Drizzle ORM** (Postgres) for typed SQL + migrations.
* Gateway owns persistence (Python never hits DB).
* Append-first (utterances); decisions/actions normalized; audit everything.
* Optional RLS once auth/RBAC is stable.
* Retention defaults with per-org overrides; nightly S3 exports.

## Security & privacy

* Consent banners and a visible “Assistant present” indicator.
* RBAC for Command Center; org policies: *coach-only*, *answer-on-ask*, *proactive timekeeper*.
* Row-level encryption for transcripts; PII redaction on exports.
* No secrets in panels; all tokens server-side; pre-signed URLs to blobs.
* Clear “recording awareness”: chimes and utterances tagged in recap.

## Roadmap (OSS → SaaS-optional)

**v0.0.1 (OSS MVP):**

* Zoom adapter (join/recv/speak), floor-request banner, optional single chime.
* Active Notes (Yjs) panel + Command Center (gated).
* Python voice worker (hosted STT/TTS) with <1s first audio.
* Postgres + S3 + Meilisearch; exports; audit trail.

**v1 (OSS parity + SaaS wrapper):**

* Teams adapter (bot + tab), then Meet (add-on + bot).
* Multi-tenant isolation, billing, metrics/alerts; core unchanged.

**Later (opt-in modules):**

* Avatar with visemes; richer diarization; embeddings/RAG; plugin marketplace.

## Modularity commitments

* **Adapters** isolate Zoom/Teams/Meet drift.
* **Speech providers** are plug-and-play; swapping STT/TTS is a config change.
* **Transport** (WS/SSE) is capability-aware; works under strict CSPs.
* **Actions plugins** (e.g., Slack/Jira/ClickUp) register into Command Center + recap.

## Performance budgets

* Capture → STT first partial: ≤250 ms hosted / ≤500 ms local.
* Policy/LLM (short answer): ≤300–400 ms.
* TTS first chunk: ≤200 ms.
* Inject to room: ≤100 ms.
* **Total to first audio:** ≤900 ms hosted / ≤1.2 s local.
* Talk window: ≤7 s; longer content goes to the panel.

## OSS license & governance

* **License**: Apache-2.0 (recommended) for permissive use + contribution; third-party models/tools keep their own licenses.
* **Governance**:

  * RFCs for public ports and adapters.
  * CODEOWNERS for each adapter; CI contract tests gate merges.
  * Security disclosures via private channel; patch windows published.
* **Telemetry**: off by default; opt-in minimal metrics for OSS (latency histograms, error counters).

## Success metrics

* Time-to-first-audio under budget on commodity hardware.
* Floor-request acceptance rate (signal respects the room).
* Decision/action capture rate (how much “sticks” post-meeting).
* Adapter stability across SDK releases (contract tests green).
* Community contributions (adapters, actions, speech backends).

## Non-goals (for now)

* Heavy avatar/animation stacks.
* Proprietary “black-box” policy engines.
* Forced cloud dependency (SaaS is optional, not required).
* Replacing the conferencing platform—we integrate with it.

## Quick start (developer)

1. `docker compose up` (Postgres, MinIO, Meilisearch).
2. `pnpm db:migrate` then `pnpm dev` for Gateway + Web.
3. `python apps/agent` (voice worker).
4. Add Zoom credentials; run the Zoom bot adapter; open the Zoom App panel.
5. Speak “Prim?” and approve the floor request in the panel—Prim responds in ≤1s.

## Why now

* Platform media APIs (Zoom/Teams/Meet) are mature enough to support respectful, low-latency agent participation.
* Open TTS/STT quality + latency are “good enough” for real-time facilitation with tiny models on commodity hardware.
* Teams want control over conversation data and the assistant’s behavior—OSS is the only credible answer.

---

### Appendix: Terminology

* **Active Notes** — shared, real-time notes (CRDT/Yjs) visible to everyone.
* **Command Center** — gated module inside Active Notes for managers/owners.
* **Floor request** — platform-appropriate banner/notification asking permission to speak; optional single chime.
* **Adapters** — platform modules for media + UI; hide SDK quirks behind small ports.
* **Speech providers** — STT/TTS/LLM modules behind an OpenAI-compatible interface.
