# Duplex voice architecture review

## Verdict

The proposal has the right security instincts and accurate platform distinctions, but the implementation plan is **too horizontal, too broad, and too contract-first**. It asks the first implementation to solve two providers, four endpoint modes, three runtime placements, multi-participant identity, media conversion, recovery, distributed ownership, and release compatibility before proving one useful call.

That is backwards. The first shippable product is one trusted pair:

> **OpenAI Realtime + Hermes Desktop, using WebRTC, as a paired Desktop plugin and Hermes backend plugin.**

Build and test that vertical slice first. Extract only the seams that it actually exercises. Add xAI next to prove the provider seam; add Discord next to prove the endpoint/runtime seam; split Telegram voice notes and Telegram live calls because they are different products.

## What the proposal gets right

Keep these decisions:

1. Native audio-to-audio, not STT → text LLM → TTS.
2. OpenAI WebRTC directly between Desktop and OpenAI.
3. Permanent API keys stay in the backend; renderer gets only short-lived authorization.
4. Hermes remains the tool execution and approval boundary.
5. A dedicated voice session receives finalized turns; a live text session is not mutated concurrently.
6. Barge-in must cancel generation and represent interrupted output honestly.
7. Raw audio is not persisted by default.
8. Provider wire events stay out of UI, tool, and persistence code.
9. Telegram Bot API voice notes are asynchronous and must not be called duplex.
10. Discord/Telegram participant presence does not imply Hermes tool authority.

These are invariants, not optional architecture polish.

## What is overengineered or premature

### 1. Two-dimensional plugin framework before one call works

The current architecture defines provider and endpoint registries, runtime-placement negotiation, capability fingerprints, deletion builds, extension envelopes, recovery plans, and reusable contract suites before a production transport exists. Most of this may become useful, but its correct shape cannot be known from paper designs alone.

**Decision:** for the Desktop/OpenAI release, use one small `OpenAIRealtimeClient` behind a narrow call-controller interface. Do not implement a generic endpoint registry, call router, media bridge, or provider marketplace.

### 2. xAI as a prerequisite rather than a proving expansion

Requiring OpenAI and xAI together forces WebRTC and AudioWorklet/WebSocket media stacks into the first release. That roughly doubles implementation and test complexity before product value is proven.

**Decision:** xAI is the second provider milestone. Its addition is when the provider interface is generalized. Before then, keep OpenAI-specific payloads isolated in `providers/openai/`, but do not invent capabilities solely for xAI.

### 3. Channel semantics contaminating the Desktop core

`ParticipantRef`, source mapping, active-speaker floor control, text-side approvals, runtime placement, sidecars, codec bridges, and distributed leases solve Discord/Telegram problems. A local Desktop user has one trusted profile, one microphone, and one output device.

**Decision:** the first call model has `callId`, `profile`, `providerSessionId`, and local call state. Participant/floor/lease concepts do not exist until a multi-user endpoint is implemented.

### 4. Premature compatibility protocol

Internal contract ranges, adapter versions, capability fingerprints, model/protocol matrices, deletion builds, and drift canaries are release-engineering mechanisms for an ecosystem that does not yet exist.

**Decision:** version the plugin package and backend API with one integer `apiVersion`. Pin the OpenAI model in configuration. Add adapter contract ranges and capability fingerprints only after a second implementation creates a real compatibility boundary.

### 5. Spec sequence blocks the tracer bullet

The existing order builds packaging, bootstrap abstractions, provider contracts, endpoint contracts, and test kits before the first real Desktop call. This maximizes the time until assumptions are tested against Electron, Hermes auth routing, microphone permissions, SDP negotiation, autoplay, hot unload, and cleanup.

**Decision:** after a seam probe and minimal packaging, build a real call immediately. No tools, persistence, provider-neutral framework, rich settings, or channels in the tracer bullet.

### 6. Unnecessary first-release backend surface

A call registry with generic endpoint/provider negotiation, `/capabilities`, event ingestion, approvals, close metadata, and server-session factories is too much for the first call.

**Decision:** start with:

- `GET /config` — non-secret OpenAI model/voice defaults and API version;
- `POST /session` — accept bounded JSON `{sdp}` through the current JSON-only `ctx.rest()` bridge, use OpenAI's unified WebRTC interface, capture the provider call ID, and return `{callId, sdp}`; or establish an equivalent tracked lifecycle if ephemeral bootstrap is selected after the seam spike;
- `POST /calls/{callId}/close` — idempotently terminate the upstream call and expire the profile-bound call record;
- later, `POST /tools/{callId}/{toolCallId}`;
- later, `POST /calls/{callId}/finalize` for bounded finalized records.

Prefer the OpenAI unified WebRTC interface initially: the backend remains a small initialization/lifecycle control plane rather than a media proxy, keeps the standard key server-side, and avoids returning even an ephemeral bearer token to plugin code. Capture the OpenAI `Location` call ID so cancellation can invoke upstream hang-up instead of leaving an orphaned session. If remote-backend latency or deployment behavior makes that materially worse, retain ephemeral bootstrap as the measured fallback with equivalent lifecycle ownership.

### 7. Future release gates applied to the first release

DAVE, Discord UDP/NAT, MTProto session encryption, tgcalls ABI packaging, distributed leases, and every provider/endpoint live smoke are unrelated to shipping Desktop/OpenAI.

**Decision:** each expansion has its own release gate. Desktop/OpenAI must not wait for xAI, Discord, or Telegram.

### 8. Verified Hermes seams expose important implementation gaps

Current Hermes supports the Desktop runtime plugin, `composer.actions`, scoped JSON `ctx.rest()`, and dashboard `plugin_api.py`, but the backend route is not a generic in-process agent plugin context:

- Desktop and backend halves install separately; remote mode requires installing/enabling/restarting the remote backend plugin.
- Desktop plugin ID/folder and dashboard manifest name must match for `ctx.rest()` routing.
- `plugin_api.py` is mounted once at backend startup and receives no `PluginContext`, active agent, tool policy, session, or approval identity.
- No public plugin API appends externally produced turns; `SessionDB` is an internal compatibility dependency today.
- `hermes serve` and the messaging gateway are separate processes and cannot share in-memory call/provider state by assumption.
- Runtime plugins must be genuinely single-file; future AudioWorklets need inline Blob source or a new asset seam.

**Decision:** make each gap an explicit compatibility or upstream gate. None is papered over as already-supported infrastructure, and tools/persistence do not block the OpenAI/Desktop v0.1 media release.

## Lean target architecture

```text
Hermes Desktop runtime plugin                         Hermes backend plugin
┌──────────────────────────────────────┐              ┌──────────────────────────┐
│ Voice button + compact call panel    │ authenticated│ OpenAI config + /session │
│ CallController                       ├─────────────►│ later: tool execution    │
│ OpenAIRealtimeClient                 │ REST         │ later: final turn append │
│  ├─ getUserMedia                     │              │ OPENAI_API_KEY            │
│  ├─ RTCPeerConnection                │              └─────────────┬────────────┘
│  ├─ remote <audio>                   │                            │
│  └─ oai-events data channel          │                            │ standard key
└──────────────────┬───────────────────┘                            │
                   └════════ WebRTC media/events ══════════════════▼
                                                        OpenAI Realtime
```

### First-release modules

Desktop:

- `plugin.ts` — registrations only.
- `voice-panel.ts` — UI and status.
- `call-controller.ts` — deterministic state machine and cleanup owner.
- `providers/openai/client.ts` — WebRTC + OpenAI events.
- `providers/openai/events.ts` — parse only events the product consumes.

Backend:

- `plugin.yaml` and dashboard manifest/API declaration.
- `dashboard/plugin_api.py` — authenticated plugin routes.
- `openai_session.py` — request validation, OpenAI call, error redaction.
- `hermes_compat.py` — only when tools/session append are introduced.

Do not create `core/`, provider/endpoint registries, `MediaBridge`, `FloorController`, sidecar protocols, or distributed coordination in the first release.

## State ownership

Use two small state machines, not one universal event bus.

Connection:

```text
idle → requesting_permission → negotiating → connected → closing → idle
                         └───────────────→ failed ────────┘
```

Conversation display:

```text
listening ↔ user_speaking → responding ↔ assistant_speaking
                                      ↘ tool_running   (after tools ship)
```

`CallController` owns all resources and exposes state snapshots to React. `OpenAIRealtimeClient` translates OpenAI events into the small set of domain events consumed by the controller. Unknown events are ignored with redacted diagnostics; malformed required events fail the call safely.

## Expansion rule: extract on the second implementation

### xAI

When xAI is built, extract a provider interface from the working OpenAI client. Generalize only the operations both clients need: connect, close, mute/input control, cancel, tool output, and normalized lifecycle/transcript events. Transport and media remain provider-owned because WebRTC tracks and explicit PCM frames are materially different.

### Discord

When Discord is built, introduce the reusable live endpoint/runtime seam, long-lived server-hosted realtime provider sessions, audio format conversion, participant identity, floor control, and linked-text approvals. Telegram's independent asynchronous voice-message part may use a bounded provider-specific server session without importing Discord participant/floor abstractions.

### Telegram

Build Bot API voice notes independently as asynchronous voice exchange. Treat MTProto/tgcalls live calls as a separate opt-in project after Discord validates server-hosted media orchestration. Do not force both through one release milestone.

## Required tests for Desktop/OpenAI

Default CI, no microphone/network/key:

- reducer/state transition tests;
- mocked `getUserMedia`, `RTCPeerConnection`, data channel, remote track, and backend request;
- SDP/config request and backend auth/no-store/redaction tests;
- permission denial, bootstrap error, ICE/data-channel failure, autoplay rejection, hang-up during every intermediate state;
- hot unload, profile/connection switch, route unmount, and window close cleanup;
- transcript delta/final handling and interruption representation;
- bundle/import inspection and plugin load smoke in a temporary `HERMES_HOME`.

Opt-in live smoke, budget capped:

1. start from a disposable Hermes profile;
2. open Desktop plugin;
3. exchange bidirectional audio with pinned OpenAI model;
4. interrupt assistant speech;
5. hang up;
6. verify all tracks/peer/data-channel/audio resources are closed;
7. after later milestones, verify one harmless tool and resumable finalized history.

## Release boundaries

### Desktop/OpenAI v0.1

Required: real duplex call, mute/hang-up, basic status and final transcript UI, barge-in, deterministic cleanup, secure backend negotiation, accessibility, installation/rollback hardening, mocked CI, and opt-in live smoke.

Not required: xAI, tools, persistence, provider selector, device selector, recovery/resumption, Discord, Telegram, sidecars, distributed leases, generic capability negotiation.

### Desktop/OpenAI v0.2

Add one harmless read-only Hermes tool and dedicated voice-session continuity. Final transcript UI, accessibility, and installation/rollback hardening are part of the v0.1 release gate.

### Expansions

1. xAI provider.
2. Discord live endpoint.
3. Telegram Bot API voice notes.
4. Telegram MTProto live endpoint, only if still desired and operationally justified.

## Final assessment

The proposal should proceed, but not in its current sequence. The core architectural discipline is sound; the scope discipline is not. Ship the narrow vertical slice, learn from a real OpenAI/Desktop implementation, and let the second provider and second endpoint pay for the abstractions they require. Wark, but with fewer registries.
