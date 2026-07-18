# Hermes Duplex Voice

Native, full-duplex speech-to-speech for the Hermes Desktop app using OpenAI's Realtime API.

> **Status:** architecture and implementation specifications. No production code has been shipped yet.

## Goal

Add a natural voice conversation mode to Hermes Desktop that:

- streams microphone audio directly to an OpenAI Realtime model over WebRTC;
- plays model audio as a remote WebRTC track with low first-audio latency;
- supports semantic turn detection and real barge-in;
- exposes an explicitly scoped subset of Hermes tools to the Realtime model;
- preserves useful transcripts and call metadata in Hermes without putting a permanent OpenAI API key in the desktop renderer; and
- installs as Hermes plugins rather than requiring a permanent fork of Hermes Desktop.

This is intentionally **not** a record → transcribe → text model → TTS pipeline. The model consumes and produces audio natively.

## Proposed shape

```text
Hermes Desktop runtime plugin
  ├─ composer action + call panel
  ├─ getUserMedia / device controls
  ├─ RTCPeerConnection
  ├─ OpenAI Realtime audio track
  └─ Realtime data-channel events
                 │
                 │ authenticated plugin REST
                 ▼
Hermes backend plugin
  ├─ mints short-lived Realtime client secrets
  ├─ resolves the effective Hermes tool scope
  ├─ executes allowed tool calls through Hermes guardrails
  ├─ records durable transcript/call events
  └─ reads secrets only on the trusted backend
                 │
                 ├────────► OpenAI REST API (session bootstrap)
                 └────────► Hermes tools/session store

Desktop WebRTC peer ◄════════════════════════════► OpenAI Realtime API
                      continuous duplex audio
```

See [`docs/architecture.md`](docs/architecture.md) for the complete design.

## High-level plan

1. **Package the extension surfaces** — establish a bundled desktop runtime plugin, a Hermes backend/dashboard API plugin, development tooling, and an installer.
2. **Secure session bootstrap** — create authenticated backend endpoints that mint narrowly configured, short-lived OpenAI Realtime client secrets.
3. **WebRTC voice runtime** — capture the microphone, negotiate WebRTC, render remote audio, expose mute/device controls, and cleanly release resources.
4. **Conversation state machine** — normalize Realtime events, drive UI state, collect final transcripts, and handle reconnect/error/expiry behavior.
5. **Hermes tool bridge** — publish only the session's effective tools, execute calls server-side through Hermes's dispatcher and approval policy, and return `function_call_output` events.
6. **Durable continuity** — record completed voice turns in a dedicated Hermes voice session and make call history resumable without corrupting an active text session.
7. **Product polish** — settings, diagnostics, accessibility, remote-backend behavior, telemetry-free local logs, and clear failure recovery.
8. **Hardening and release** — automated tests, compatibility checks against supported Hermes versions, live Realtime smoke tests, packaging, and install/rollback documentation.

The ordered implementation specs live in [`.agents/specs/`](.agents/specs/).

## Core decisions

- **WebRTC, not browser WebSockets:** OpenAI recommends WebRTC for browser/mobile clients, and WebRTC handles jitter, media playback, and interruption-aware output buffering.
- **Ephemeral client secrets:** the standard OpenAI API key remains on the Hermes backend. The renderer receives only a short-lived client secret.
- **Direct OpenAI media path:** after bootstrap, audio flows between the desktop and OpenAI; Hermes is not an audio proxy.
- **Function tools by default:** Hermes remains the executor and policy boundary. The Realtime model receives schemas, while calls return to the authenticated backend plugin.
- **Dedicated voice sessions:** a call writes to its own durable Hermes session. This avoids mutating a concurrently active in-memory text conversation.
- **Plugin first:** use Hermes's desktop and backend plugin surfaces. Any missing stable seam is documented and proposed upstream rather than silently patched in a fork.

## Repository layout (target)

```text
.
├── desktop/                 # TypeScript source; bundled to one runtime plugin.js
├── backend/                 # Python Hermes plugin and authenticated API routes
├── scripts/                 # install, uninstall, package, and compatibility checks
├── tests/                   # cross-component contract and live-smoke harnesses
├── docs/
│   └── architecture.md
└── .agents/specs/           # ordered implementation specifications
```

## Non-goals for the first release

- SIP/PSTN calling or conference-room transport
- replacing Hermes's built-in microphone implementation in core
- proxying raw audio through the Hermes backend
- exposing every Hermes tool by default
- silently executing consequential tools without Hermes approval policy
- supporting arbitrary Realtime providers behind a premature abstraction

## References

- [OpenAI Realtime API with WebRTC](https://developers.openai.com/api/docs/guides/realtime-webrtc)
- [OpenAI Realtime conversations](https://developers.openai.com/api/docs/guides/realtime-conversations)
- [OpenAI Realtime tools](https://developers.openai.com/api/docs/guides/realtime-mcp)
- [Hermes Desktop](https://hermes-agent.nousresearch.com/docs/user-guide/desktop)
- [Hermes plugins](https://hermes-agent.nousresearch.com/docs/user-guide/features/plugins)

## License

[MIT](LICENSE)
