# Architecture

## Scope

Hermes Duplex Voice first ships one complete, tested product path:

> **OpenAI Realtime ↔ Hermes Desktop plugin over WebRTC**

The architecture preserves clean seams for later xAI, Discord, and Telegram work, but it does not implement their abstractions before they have a concrete consumer.

The live path is native audio-to-audio:

```text
local microphone ══ WebRTC ══► OpenAI Realtime
audio element    ◄═ WebRTC ═══ OpenAI Realtime
```

Transcripts support UI and continuity. They are not an intermediate STT → text-model → TTS pipeline.

See [the architecture review](architecture-review.md) for the keep/defer/delete analysis behind this design.

## Design rules

1. **Prove one vertical slice first.** Packaging, backend negotiation, Desktop media, call state, cleanup, and a live smoke test precede tools, persistence, other providers, and channels.
2. **Direct media path.** Desktop audio goes directly to OpenAI. Hermes handles authenticated session initialization, not media proxying.
3. **Backend-only permanent key.** `OPENAI_API_KEY` never enters renderer state, storage, logs, or the runtime bundle.
4. **OpenAI wire isolation.** OpenAI request/event types stay under `providers/openai` or the backend OpenAI session module.
5. **One cleanup owner.** The call controller idempotently closes tracks, peer connection, data channel, audio element, listeners, and pending work on every terminal path.
6. **Hermes owns actions.** When tools are added, schemas and calls use the same server-resolved scope and canonical Hermes dispatcher.
7. **Honest continuity.** Persist only finalized turns in a dedicated voice session; mark interruption and never claim unheard output was delivered.
8. **No raw audio persistence.** Recording is a separate, consented future feature.
9. **Extract on second use.** Generalize provider behavior when xAI is implemented; generalize endpoint/runtime behavior when Discord is implemented.
10. **Plugins, not checkout patches.** Use the verified Desktop runtime-plugin and backend plugin API surfaces. Missing capabilities become small generic Hermes upstream proposals.

## Verified Hermes extension seams

The current Hermes source provides:

- Desktop runtime plugins at `<HERMES_HOME>/desktop-plugins/<id>/plugin.js`;
- renderer execution with browser media/WebRTC APIs;
- imports limited to `@hermes/plugin-sdk`, `react`, and `react/jsx-runtime`;
- composer contribution areas including `composer.actions`;
- authenticated, plugin-scoped JSON REST through `ctx.rest()` to `/api/plugins/<id>/...` (ordinary bodies/responses are JSON; raw SDP must be wrapped);
- user backend plugin APIs at `plugins/<id>/dashboard/manifest.json` with `"api": "plugin_api.py"`; the module exports `router = APIRouter()`, is mounted once when the explicitly enabled backend starts, and route changes require backend restart;
- an exact ID match between Desktop plugin/folder and backend dashboard manifest so `ctx.rest()` reaches the intended namespace;
- hot reload/unload with contribution disposal for the Desktop half only.

These are implementation dependencies and need compatibility smoke tests. Runtime Desktop plugins are trusted renderer code, not a sandbox. Remote mode is a dual installation: the Desktop `plugin.js` stays on the local machine, while the directory backend plugin must be installed, enabled, and restarted on the remote Hermes host. A pip entry-point plugin alone does not expose `plugin_api.py` today.

## Initial system

```text
┌──────────────────────────────────────┐
│ Hermes Desktop runtime plugin        │
│                                      │
│  plugin.ts                           │
│  voice-panel.ts                      │
│  call-controller.ts                  │
│  providers/openai/client.ts          │
│  providers/openai/events.ts          │
└───────────────┬──────────────────────┘
                │ authenticated plugin REST
                ▼
┌──────────────────────────────────────┐
│ Hermes backend plugin                │
│                                      │
│  dashboard/plugin_api.py             │
│  openai_session.py                   │
│  later: hermes_compat.py             │
│                                      │
│  OPENAI_API_KEY                      │
└───────────────┬──────────────────────┘
                │ session initialization
                ▼
        OpenAI Realtime API

Desktop ◄════════ direct WebRTC media/events ════════► OpenAI
```

## Session initialization

Start with OpenAI's unified WebRTC interface unless a measured spike shows that the ephemeral-token path is materially better for Hermes remote backends.

Unified flow:

1. User clicks the Duplex Voice action.
2. Desktop acquires one microphone track and creates a peer connection, remote autoplay audio element, and `oai-events` data channel.
3. Desktop creates an SDP offer, calls `await pc.setLocalDescription(offer)`, and sends `{ sdp: pc.localDescription.sdp }` through authenticated `ctx.rest('/session', { method: 'POST', body: ... })`.
4. The JSON wrapper is required because the current Desktop `ctx.rest()` bridge serializes request bodies as JSON and parses successful responses as JSON; it cannot carry raw `application/sdp` end to end.
5. Backend validates the bounded SDP string, builds the authoritative pinned OpenAI session configuration, and forwards a multipart request to `POST /v1/realtime/calls` with `OPENAI_API_KEY`.
6. Backend captures OpenAI's provider call ID from the response `Location` header, creates an opaque application `callId`, and stores a minimal expiring record bound to the active profile and provider call.
7. Backend returns `{ callId, sdp: answer }` with `Cache-Control: no-store` and redacted errors.
8. Desktop sets the remote description and waits for peer/data-channel readiness.
9. Media and provider events then flow directly between Desktop and OpenAI.
10. Hang-up, unload, failed negotiation, disconnect, or record expiry closes the local peer and asks the backend to terminate the upstream call idempotently.

If the unified path is unsuitable, the alternate flow mints a short-lived secret through `/v1/realtime/client_secrets`; it must still establish the same opaque application call lifecycle and give the backend enough provider identity to perform authenticated upstream hang-up. Do not maintain both production paths without measured evidence.

The backend chooses the initial model, voice, instructions, VAD mode, allowed endpoint, safety identifier, request limits, and timeout. Runtime Desktop plugins are trusted renderer code, so v0.1 enforces this boundary by exposing no user-controlled override and by never sending `session.update` for those protected fields. If strict authority against a modified renderer or dynamic server-side control is required, attach OpenAI's server sideband connection and keep protected updates and business logic there. The renderer never supplies an arbitrary upstream URL or permanent credential.

The call record is deliberately smaller than a generic call registry: application call ID, provider call ID, profile binding, creation/expiry state, and terminal reason. It exists so cancellation cannot orphan an upstream session and so later tool/finalize routes can verify active call ownership.

## Desktop components

### Plugin registration

`plugin.ts` registers:

- one explicit composer action; and
- one compact call panel or pane.

It does not replace or monkey-patch Hermes dictation. The built runtime artifact is one ESM `plugin.js` with only Hermes-supported externals.

### Call controller

The controller owns connection state and every acquired resource.

```text
idle
  → requesting_permission
  → negotiating
  → connected
  → closing
  → idle

requesting_permission | negotiating | connected → failed → closing → idle
```

Required commands:

- `start()` from an explicit gesture;
- `setMuted(boolean)` by toggling the local track;
- `close(reason)` idempotently from any state.

Conversation display state is separate:

```text
listening ↔ user_speaking → responding ↔ assistant_speaking
```

Tools later add `tool_running`; they do not alter connection ownership.

### OpenAI client

The OpenAI client owns:

- `RTCPeerConnection` and data-channel setup;
- remote-track attachment;
- sending supported client events;
- parsing the small set of server events the UI and later features consume;
- interruption status; and
- provider resource cleanup.

It emits small domain events such as `session.ready`, `speech.started`, `speech.stopped`, `assistant.started`, `assistant.interrupted`, `transcript.delta`, `transcript.final`, `response.done`, and `error`.

The v0.1 client event allowlist excludes protected `session.update` fields. A future sideband-controlled implementation may update them from the backend instead.

This is not yet a public provider adapter contract. It is an isolated implementation from which a contract can be extracted when xAI exists.

## Barge-in

With OpenAI VAD and WebRTC, provider output buffering and automatic truncation handle unheard audio. The Desktop client still:

- reflects user speech and interruption immediately in UI;
- records the response as interrupted for later persistence;
- never persists a provisional transcript as delivered speech; and
- supports explicit cancellation/hang-up.

Barge-in must be tested with real audio in the opt-in smoke test and with deterministic mocked provider events in CI before release.

## Backend API by milestone

### v0.1

| Method | Route | Purpose |
|---|---|---|
| `GET` | `/config` | Safe API version, pinned model/voice, readiness, and limits |
| `POST` | `/session` | Negotiate OpenAI WebRTC session; return opaque call ID plus SDP answer or short-lived authorization |
| `POST` | `/calls/{call_id}/close` | Idempotently terminate the provider call and expire backend call state |

### v0.2 additions

| Method | Route | Purpose |
|---|---|---|
| `POST` | `/calls/{call_id}/tools/{tool_call_id}` | Reauthorize and execute one scoped Hermes tool call |
| `POST` | `/calls/{call_id}/finalize` | Append a bounded batch of finalized transcript/tool records and close metadata |

A generic provider/endpoint call router is not part of the Desktop/OpenAI release.

## Tool boundary (v0.2)

The current `plugin_api.py` module is independently imported and receives no `PluginContext`, active agent, session, tool policy, or approval identity. Therefore tool execution has a host-seam gate before the flow below: prefer a generic upstream Hermes server-side execution context; otherwise use a version-pinned `hermes_compat.py` internal adapter only for one harmless read-only tool and contract-test exact scope propagation. If that cannot be proven, tools stay disabled without blocking voice release.

1. Resolve one explicitly configured, harmless read-only Hermes tool through the verified seam.
2. Generate its schema through the Hermes compatibility adapter.
3. Supply the schema to OpenAI.
4. Receive final function arguments on the data channel.
5. POST the tool request to the backend with the opaque application call ID and provider tool-call ID; the provider session/call ID remains backend-only.
6. Recheck active call, exact tool scope, arguments, timeout, output limit, and idempotency.
7. Dispatch through Hermes's canonical tool path.
8. Return a `function_call_output`, then request the next response.

No approval-requiring or destructive tool ships until Hermes exposes or the implementation verifies a server-side execution/approval context and native approve/reject/timeout behavior end to end. HTTP authentication alone is not approval. Scope lookup failure exposes zero tools.

## Continuity (v0.2)

Each call writes to a dedicated, non-running Hermes voice session. Only finalized user/assistant turns and canonical tool call/result pairs are appended. Records carry provider item/call IDs for deduplication and interruption metadata for assistant output.

Do not mutate an active text-agent session. Hermes currently exposes no public plugin API for appending externally produced voice turns. Unless an upstream writer lands first, isolate version-pinned `SessionDB.create_session()`/`append_message()` use in `hermes_compat.py`, contract-test it against a disposable real database, and persist only metadata supported by the verified API. Never issue ad-hoc SQL.

## Security and privacy

- Backend route inherits Hermes authentication and active profile.
- Backend configuration, not request JSON, chooses credentials and upstream endpoint.
- Session responses use `Cache-Control: no-store`.
- Limit SDP/body size, session creation rate, timeout, and concurrent calls.
- Redact authorization, SDP, transcripts, data-channel payloads, tool arguments/results, and raw audio from normal logs.
- Runtime plugin stores no token or transcript unless explicitly required.
- Hot unload, route unmount, profile/connection switch, window close, and failures all invoke cleanup.

## Testing and release gates

### Offline CI

- Typecheck, lint, bundle, and unsupported-import inspection.
- Backend API auth, bounds, no-store, error redaction, timeout, and mocked OpenAI requests.
- Mocked microphone, peer, data channel, remote track, autoplay, and network behavior.
- Every state transition and cleanup from every intermediate state.
- Transcript delta/final and interruption handling.
- Plugin/backend installation and load in a disposable `HERMES_HOME`.
- No default test requires a key, network, microphone, or paid API.

### Opt-in live smoke

- Pinned OpenAI model and strict budget/time cap.
- Real bidirectional Desktop audio.
- Barge-in while assistant audio plays.
- Explicit hang-up and verified resource cleanup.
- v0.2 additionally verifies one harmless tool and valid dedicated Hermes history.

Desktop/OpenAI releases independently. xAI, Discord, and Telegram cannot block it.

## Expansion without speculative infrastructure

### xAI provider

Add xAI only after Desktop/OpenAI passes its release gate. Extract the minimum provider contract from OpenAI's working interface, then implement xAI WebSocket + AudioWorklet media behind it. Transport/media remain provider-owned; do not force WebRTC tracks and PCM frames into one lowest-common-denominator abstraction.

### Discord endpoint

Discord introduces the first server-hosted, multi-user endpoint. At that point add server provider sessions, endpoint contracts, audio format conversion, participant identity, floor control, linked-text approvals, and worker ownership. `hermes serve` and the gateway are separate processes, so explicitly place endpoint media/provider transport in the gateway or a dedicated voice worker and use authenticated bounded IPC/API calls for backend tools, persistence, and control; never assume an in-memory registry is shared. Reuse Hermes's existing Discord identity and connection through a supported generic media seam; never patch the installed checkout.

### Telegram endpoints

Telegram Bot API voice notes are an asynchronous endpoint and can follow Discord or proceed independently if prioritized. Live Telegram group calls are a separate optional MTProto/tgcalls sidecar project. They do not share a release gate and must never be presented as the same mode.

## Explicit non-goals for Desktop/OpenAI

- xAI support;
- Discord or Telegram;
- generic provider/endpoint registries;
- media format bridge or codecs;
- participant identity/floor control;
- server-hosted provider sessions;
- resumption/recovery framework;
- distributed leases;
- arbitrary endpoints or provider marketplace;
- raw-audio recording;
- transparent provider handoff.
