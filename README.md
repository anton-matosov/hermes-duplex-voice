# Hermes Duplex Voice

Provider-pluggable, full-duplex speech-to-speech for Hermes Desktop.

> **Status:** architecture and implementation specifications. No production code has shipped yet.

## Goal

Add a natural voice conversation mode to Hermes Desktop that:

- streams microphone audio directly to a native duplex voice model;
- plays model audio with low first-audio latency;
- supports provider VAD, natural turn-taking, and real barge-in;
- exposes an explicitly scoped subset of Hermes tools;
- preserves finalized transcripts and call metadata in Hermes;
- keeps permanent provider credentials out of the desktop renderer; and
- survives provider API changes by isolating authentication, transport, protocol events, media, and capabilities behind versioned adapters.

This is intentionally **not** a record → transcribe → text model → TTS pipeline. The selected model consumes and produces audio natively.

## Initial providers

| Provider | Model family | Client transport | Browser authentication | Media handling |
|---|---|---|---|---|
| OpenAI | Realtime (`gpt-realtime-*`) | WebRTC | short-lived client secret | browser/WebRTC media tracks |
| xAI | Grok Voice (`grok-voice-*`) | WebSocket | ephemeral token in `Sec-WebSocket-Protocol` | AudioWorklet + binary 24 kHz PCM |

OpenAI remains the default. xAI/Grok is the second end-to-end provider and proves that the core is not coupled to WebRTC, OpenAI event names, or one session-secret shape.

## Proposed shape

```text
Hermes Desktop runtime plugin
  ├─ composer action + provider-aware call panel
  ├─ shared microphone/device manager
  ├─ provider-neutral call controller
  ├─ OpenAI adapter ── WebRTC media + data channel
  └─ xAI adapter ───── WebSocket events + AudioWorklet PCM
                         │
                         │ authenticated plugin REST
                         ▼
Hermes backend plugin
  ├─ provider registry + capability discovery
  ├─ OpenAI bootstrap adapter ── OPENAI_API_KEY
  ├─ xAI bootstrap adapter ───── XAI_API_KEY
  ├─ Hermes tool-policy bridge
  └─ durable transcript/call events
                         │
                         └────────► Hermes tools/session store

Desktop transport ◄══════════════════════════► selected voice provider
                   continuous duplex audio
```

See [`docs/architecture.md`](docs/architecture.md) for the full design and provider contracts.

## High-level plan

1. **Package the extension surfaces** — bundled desktop runtime plugin, Hermes backend plugin, provider modules, development tooling, and reversible installer.
2. **Secure provider bootstrap** — authenticated backend endpoints select a configured provider and mint provider-specific short-lived credentials without leaking permanent keys.
3. **Freeze internal contracts** — versioned provider, transport, media, event, tool, and capability interfaces with contract fixtures for OpenAI and xAI.
4. **Implement duplex transports** — OpenAI WebRTC plus xAI WebSocket with AudioWorklet capture/playback, bounded jitter buffering, and cancellation.
5. **Normalize conversation semantics** — map provider events into stable lifecycle, transcript, interruption, error, and tool-call events while preserving provider metadata behind an extension envelope.
6. **Bridge Hermes tools** — convert schemas per provider, execute only server-authorized tools through Hermes, and return results through the selected adapter.
7. **Preserve continuity** — record finalized turns and provider/model metadata in a dedicated Hermes voice session.
8. **Productize provider selection** — capability-driven settings and UI, voice/model discovery, diagnostics, accessibility, and graceful unsupported-feature handling.
9. **Harden against drift** — pinned production models, protocol fixtures, unknown-event canaries, live smoke tests per provider, compatibility matrices, and documented adapter migration rules.

The ordered implementation specs live in [`.agents/specs/`](.agents/specs/).

## Core decisions

- **Provider contract above wire protocol:** UI, Hermes tools, and persistence consume stable domain events—not raw OpenAI/xAI messages.
- **Transport is replaceable:** WebRTC, event channels, WebSocket, and Web Audio are separate adapters under one duplex transport interface.
- **Capability negotiation, not provider-name branching:** semantic VAD, resumption, codecs, voices, rate-limit events, cancellation, and transport support are advertised at runtime.
- **Provider-specific behavior is retained:** the shared contract covers portable semantics, while a namespaced extension envelope preserves useful features such as xAI resumption or OpenAI WebRTC output truncation.
- **Ephemeral browser credentials:** `OPENAI_API_KEY` and `XAI_API_KEY` stay on the backend. The renderer receives only a short-lived, call-bound authorization descriptor.
- **Direct media path:** after bootstrap, audio flows between Desktop and the selected provider; Hermes is not an audio proxy.
- **Hermes owns actions:** providers receive function schemas, but Hermes remains the authenticated executor and policy boundary.
- **Pinned production models:** aliases such as `*-latest` are allowed for development discovery, not default production configuration.
- **Dedicated voice sessions:** each call writes to its own durable Hermes session, avoiding mutation of a concurrently active text agent.
- **Plugin first:** missing stable Hermes seams are proposed upstream rather than patched into a fork.

## Repository layout (target)

```text
.
├── desktop/
│   └── src/
│       ├── core/             # call state, domain events, shared media contracts
│       └── providers/
│           ├── openai/       # WebRTC + OpenAI protocol adapter
│           └── xai/          # WebSocket + PCM audio adapter
├── backend/
│   └── src/hermes_duplex_voice/
│       ├── core/             # call registry, policy, persistence
│       └── providers/
│           ├── openai.py     # client-secret bootstrap
│           └── xai.py        # ephemeral-token bootstrap
├── scripts/
├── tests/fixtures/providers/
├── docs/
│   └── architecture.md
└── .agents/specs/
```

## Non-goals for the first release

- SIP/PSTN calling or conference-room transport
- replacing Hermes's built-in microphone implementation in core
- proxying raw audio through Hermes
- pretending all providers have identical capabilities
- generic user-supplied endpoints or arbitrary protocol plugins
- exposing every Hermes tool by default
- silently executing consequential tools without Hermes approval policy

## References

- [OpenAI Realtime API with WebRTC](https://developers.openai.com/api/docs/guides/realtime-webrtc)
- [OpenAI Realtime conversations](https://developers.openai.com/api/docs/guides/realtime-conversations)
- [xAI Voice Agent API](https://docs.x.ai/developers/model-capabilities/audio/voice-agent)
- [xAI ephemeral tokens](https://docs.x.ai/developers/model-capabilities/audio/ephemeral-tokens)
- [Hermes Desktop](https://hermes-agent.nousresearch.com/docs/user-guide/desktop)
- [Hermes plugins](https://hermes-agent.nousresearch.com/docs/user-guide/features/plugins)

## License

[MIT](LICENSE)
