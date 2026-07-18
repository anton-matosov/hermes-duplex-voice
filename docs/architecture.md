# Architecture

## 1. Purpose

Hermes Duplex Voice adds low-latency, full-duplex, native speech-to-speech conversation to Hermes Desktop using OpenAI's Realtime API. It is delivered as extensions around Hermes's existing plugin boundaries rather than as a long-lived fork.

The defining requirement is a direct audio model path:

```text
microphone audio → Realtime model → generated audio
```

Text transcripts are an observability and continuity artifact, not an intermediate inference pipeline.

## 2. Design principles

1. **Audio stays direct.** Once a call is established, WebRTC media flows directly between Electron's Chromium renderer and OpenAI.
2. **Secrets stay trusted.** A standard OpenAI API key is loaded only by the Hermes backend. The desktop receives a short-lived client secret.
3. **Hermes remains the action boundary.** The Realtime model can request tools; the backend resolves scope, approvals, execution, and output.
4. **One authority per concern.** OpenAI owns the live Realtime conversation, the desktop owns media/UI state, and Hermes owns durable sessions, tools, and policy.
5. **Fail closed.** Unknown tools, malformed arguments, expired calls, missing session scope, and approval failures do not execute.
6. **Plugin first.** Use public extension seams where available. Isolate Hermes-version compatibility code and make unsupported versions fail with a clear diagnostic.
7. **No hidden telemetry.** Keep operational logs local and redact transcripts, tool arguments, credentials, SDP, and tokens by default.

## 3. System context

```text
┌──────────────────────────────────────────────────────────────────┐
│ Hermes Desktop renderer                                         │
│                                                                  │
│  Composer action / call panel                                    │
│      │                                                           │
│      ├── MediaDevices + RTCPeerConnection ─══════════════════╗    │
│      │                                                        ║    │
│      ├── Realtime data channel                                ║    │
│      │   ├── lifecycle/VAD/transcript events                  ║    │
│      │   └── function call requests                           ║    │
│      │                                                        ║    │
│      └── ctx.rest() ───────────────────────────────┐           ║    │
└───────────────────────────────────────────────────┼───────────║────┘
                                                    │           ║
                                      authenticated │           ║ WebRTC
                                      plugin REST   │           ║ audio
                                                    ▼           ║ + events
┌──────────────────────────────────────────────────────────────────┐
│ Hermes backend                                                   │
│                                                                  │
│  Duplex Voice plugin API                                         │
│      ├── session bootstrap ───────────────► OpenAI REST API       │
│      ├── tool catalog + execution ─────────► Hermes dispatcher    │
│      ├── approval/guardrail integration                          │
│      └── transcript/call persistence ──────► Hermes SessionDB     │
└──────────────────────────────────────────────────────────────────┘
                                                                ║
                                                                ▼
                                                    ┌─────────────────────┐
                                                    │ OpenAI Realtime API │
                                                    │ speech ↔ speech     │
                                                    └─────────────────────┘
```

## 4. Components

### 4.1 Desktop runtime plugin

The desktop extension contributes a composer action and a native-looking call panel. It is developed in TypeScript/React and bundled into a single ESM `plugin.js`, because Hermes runtime plugins may only import `@hermes/plugin-sdk`, `react`, and `react/jsx-runtime` at runtime.

Responsibilities:

- request microphone permission and enumerate/select audio inputs;
- create and own `MediaStream`, `RTCPeerConnection`, remote `<audio>`, and the `oai-events` data channel;
- request a client secret and call descriptor from the backend plugin;
- negotiate SDP directly with `POST https://api.openai.com/v1/realtime/calls` using the ephemeral credential;
- translate raw Realtime events into a small typed client state machine;
- render connection, listening, thinking, speaking, tool-running, muted, reconnecting, and failed states;
- send function calls to the backend and return `function_call_output` plus `response.create` to OpenAI;
- batch final transcript/call events to the backend; and
- stop all tracks, close channels, release audio elements, and invalidate local state on hang-up or unload.

The runtime plugin executes in the renderer realm and can use browser WebRTC APIs directly. It must treat itself as privileged local code and avoid loading remote scripts.

### 4.2 Backend plugin

The backend extension is a Hermes Python plugin with authenticated routes mounted below `/api/plugins/hermes-duplex-voice/`. Desktop code reaches only this namespace through `ctx.rest()`.

Responsibilities:

- read `OPENAI_API_KEY` (or a future explicitly supported Hermes credential resolver) on the backend;
- validate the authenticated profile and create a unique call ID;
- build the authoritative Realtime session configuration;
- mint a short-lived client secret through `POST /v1/realtime/client_secrets`;
- bind a privacy-preserving `OpenAI-Safety-Identifier` at secret creation;
- resolve the Hermes toolsets allowed for this voice call;
- convert Hermes OpenAI-format schemas to Realtime function schemas;
- execute tool calls through Hermes's dispatcher with identical enabled/disabled toolset scope;
- preserve Hermes pre/post-tool hooks, middleware, approval behavior, and session/task identifiers;
- create and append to a dedicated voice session; and
- expire call state and reject replayed or cross-call requests.

### 4.3 OpenAI Realtime session

The initial implementation uses the GA Realtime WebRTC interface and keeps the model configurable. Documentation examples may name a current model, but production defaults must be centralized and capability-checked rather than scattered through the code.

Initial session posture:

- `type: realtime`;
- `output_modalities: ["audio"]` while still consuming transcript events;
- output voice configurable before first audio is emitted;
- `semantic_vad` by default;
- automatic response creation and interruption enabled;
- concise spoken instructions that acknowledge tools without narrating internal details; and
- a narrow function-tool catalog supplied at session creation/update.

For WebRTC, OpenAI manages output buffering and automatically truncates unplayed audio after an interruption. The client still observes speech/response events so the UI and tool state remain correct.

### 4.4 Tool bridge

Function tools are preferred over exposing a public MCP endpoint because Hermes must remain the authenticated executor and approval boundary.

Flow:

1. Backend resolves the exact enabled and disabled toolsets for the voice session.
2. Backend obtains Hermes tool definitions using that scope.
3. Desktop sends those schemas in `session.update`.
4. Realtime emits a completed function-call item with `name`, `call_id`, and JSON arguments.
5. Desktop posts the request with the call ID and voice session ID to the backend.
6. Backend verifies the tool remains in scope and dispatches it with the same toolset constraints.
7. Backend returns a bounded JSON-string result or a structured denial/error.
8. Desktop creates a `function_call_output` item and sends `response.create`.

The first implementation should allow a curated, low-risk toolset. Approval-requiring and destructive tools remain disabled until the native approval round-trip is proven end to end.

### 4.5 Durable session continuity

A live Realtime conversation and a Hermes text-agent loop cannot both be authoritative for the same in-memory message list. Each call therefore gets a dedicated Hermes session with source metadata identifying Duplex Voice.

Persistence policy:

- create the Hermes voice session before minting a client secret;
- append only finalized user and assistant transcripts, never unstable deltas;
- preserve interruption status so unheard assistant text is not represented as fully delivered;
- store tool calls/results using Hermes's canonical message representation where possible;
- record call start/end, model, voice, duration, and terminal reason without storing raw audio;
- make the session available in normal Hermes history after the call; and
- treat summary/memory extraction as an explicit post-call operation, not an implicit second LLM turn during live audio.

If Hermes lacks a stable public session append API, the compatibility adapter may use `SessionDB` only for a dedicated, non-running voice session. It must never mutate the active desktop text session directly.

## 5. API contracts

The exact schemas are specified during implementation, but the backend surface is intentionally small:

| Method | Route | Purpose |
|---|---|---|
| `GET` | `/capabilities` | Version, model/config readiness, supported integration seams |
| `POST` | `/calls` | Create voice session, resolve tools, mint client secret |
| `POST` | `/calls/{call_id}/tools/{call_id_from_model}` | Validate and execute one Realtime function call |
| `POST` | `/calls/{call_id}/events` | Idempotently append finalized transcript and lifecycle events |
| `POST` | `/calls/{call_id}/close` | Finalize metadata and invalidate call state |

`POST /calls` returns only the ephemeral secret, expiry, effective session configuration, tool schemas, Hermes voice-session ID, and call ID. It never returns the standard API key.

Every mutating request carries a client-generated idempotency key. The backend binds requests to the authenticated profile, call ID, and expected Realtime call/tool IDs.

## 6. State machines

### 6.1 Client call state

```text
idle
  → requesting_permission
  → bootstrapping
  → negotiating
  → connected/listening
      ↔ user_speaking
      ↔ model_thinking
      ↔ model_speaking
      ↔ tool_running
  → reconnect_required | failed
  → closing
  → idle
```

A dropped Realtime peer connection is not transparently resumed because the server-side conversation may no longer be recoverable. The UI offers a new call seeded with the durable transcript rather than pretending the old media session survived.

### 6.2 Backend call state

```text
created → active → closing → closed
              └────────────→ expired
```

Only `active` calls may execute tools or accept transcript events. Closing is idempotent. A periodic or request-time sweep expires abandoned calls.

## 7. Security model

- Standard OpenAI credentials never enter renderer storage, logs, transcript events, or Git.
- Client secrets are short-lived, held in memory, and discarded when negotiation finishes or fails.
- Session bootstrap is available only through Hermes-authenticated plugin REST.
- Tool schemas and execution use the same resolved scope; the backend rechecks scope on every call.
- Tool name is selected from the server-side catalog, not trusted from the browser.
- Tool arguments and outputs have size/depth limits and JSON validation.
- Consequential tools require a proven approval path; until then they are excluded.
- No raw audio is persisted by default.
- Local logs redact authorization headers, client secrets, SDP, transcript text, and tool payloads by default.
- Remote Hermes backends remain supported: microphone/WebRTC stay on the desktop; only bootstrap, tools, and transcript events traverse the authenticated backend connection.

## 8. Reliability and performance

Targets for the first production candidate:

- median call bootstrap under 2 seconds on a healthy connection;
- audible barge-in stop under 250 ms after OpenAI emits speech-start/interruption signals;
- no leaked microphone tracks or peer connections after close/unload;
- tool calls idempotent across client retry;
- bounded event queues and transcript batches;
- clear handling for token expiry, permission denial, missing input device, ICE failure, OpenAI rate limits, backend restart, and unsupported Hermes versions; and
- no backend audio proxy or audio transcoding in the normal path.

## 9. Compatibility strategy

Hermes's desktop plugin SDK is a supported runtime surface, but tool and session internals may evolve. All Hermes-specific imports and schema conversions belong in a small backend compatibility module with:

- startup probes for required functions and signatures;
- a supported Hermes version range;
- contract tests against the supported checkout(s);
- fail-closed diagnostics from `/capabilities`; and
- a documented upstream request whenever a private seam becomes necessary.

The repository must not carry patched Hermes source. If an essential capability cannot be expressed through plugins, the preferred outcome is a minimal upstream extension seam followed by a normal released dependency.

## 10. Testing strategy

- **Desktop unit tests:** event reducer, function-call extraction, idempotency, resource cleanup, device changes, and UI states.
- **Backend unit tests:** config validation, secret redaction, schema conversion, scope checks, idempotency, expiry, and persistence.
- **Contract tests:** desktop request/response fixtures against backend models; captured Realtime event fixtures against the reducer.
- **Hermes integration tests:** load plugins in a temporary `HERMES_HOME`, resolve real tool schemas, execute a harmless tool, and verify a dedicated voice session.
- **Browser tests:** mocked media/WebRTC for deterministic CI plus an opt-in live Electron smoke test.
- **Live OpenAI smoke:** opt-in, budget-capped test validating WebRTC negotiation, bidirectional audio, transcript events, interruption, and one harmless function tool.

## 11. Delivery phases

The implementation is divided into the ordered specifications in [`.agents/specs/`](../.agents/specs/). Each spec ends in a separately verifiable artifact and keeps unsafe tool execution out of the critical path until the underlying policy integration is demonstrated.
