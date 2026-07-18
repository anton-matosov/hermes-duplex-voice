# Hermes Duplex Voice

Provider- and frontend-pluggable, full-duplex speech-to-speech for Hermes.

> **Status:** architecture and implementation specifications. No production code has shipped yet.

## Goal

Add a natural voice conversation mode to Hermes that:

- streams participant audio directly into a native duplex voice model;
- plays model audio with low first-audio latency;
- supports provider VAD, natural turn-taking, and real barge-in on live endpoints;
- supports Hermes Desktop and server-hosted Discord/Telegram frontends;
- preserves endpoint participant identity without widening Hermes tool permissions;
- exposes an explicitly scoped subset of Hermes tools;
- stores finalized transcripts and call metadata rather than raw audio;
- keeps permanent provider and platform credentials in their trusted runtimes; and
- survives provider and platform API changes through versioned capability-driven adapters.

This is intentionally **not** a record → transcribe → text model → TTS pipeline. The selected provider consumes and produces audio natively. Telegram Bot API voice notes are store-and-forward and are clearly labeled non-duplex.

## Initial providers

| Provider | Model family | Desktop transport | Server/channel transport | Authentication |
|---|---|---|---|---|
| OpenAI | Realtime (`gpt-realtime-*`) | WebRTC | server WebSocket | ephemeral Desktop secret or backend key |
| xAI | Grok Voice (`grok-voice-*`) | WebSocket + AudioWorklet | server WebSocket | ephemeral Desktop token or backend key |

OpenAI remains the default. xAI/Grok proves that the core is not coupled to WebRTC, OpenAI event names, or one session-secret shape.

## Voice frontends

| Frontend endpoint | Mode | Runtime | Notes |
|---|---|---|---|
| Hermes Desktop | full duplex | renderer | direct ephemeral provider media |
| Discord voice channel | full duplex | Hermes gateway | Voice Gateway v8, RTP/Opus, DAVE, per-user floor control |
| Telegram Bot API voice message | asynchronous voice notes | Hermes gateway | exact sender identity; not realtime/barge-in |
| Telegram group call | full duplex | isolated MTProto/tgcalls sidecar | dedicated visible user account; opt-in experimental |

> **Telegram constraint:** the HTTP Bot API can send and receive voice messages but cannot join a live group call. Telegram's `phone.joinGroupCall` method is user-only and requires a local tgcalls engine. The live plugin therefore uses a separately authorized Telegram user session—not the bot token—and has stricter tool defaults.

## Proposed shape

```text
                         ┌───────────────────────────────────┐
                         │ Hermes duplex backend/core        │
                         │ provider + endpoint registries    │
                         │ floor/identity/tool/session policy│
                         └───────────────┬───────────────────┘
                                         │
                   ┌─────────────────────┴────────────────────┐
                   │                                          │
Hermes Desktop runtime plugin                    Gateway/channel endpoint plugins
  ├─ Desktop endpoint                              ├─ Discord Voice
  ├─ OpenAI WebRTC runtime                         ├─ Telegram voice messages
  └─ xAI WebSocket/AudioWorklet                     ├─ Telegram MTProto sidecar
                   │                                └─ server provider runtimes
                   │                                          │
                   └────────────► OpenAI / xAI ◄───────────────┘
```

Providers and frontends are independent axes. The same Discord endpoint can use OpenAI or xAI without importing either wire protocol; the same provider can serve Desktop or a gateway-hosted channel through different runtime placements.

See [`docs/architecture.md`](docs/architecture.md) for the complete contracts, media topology, identity model, and platform constraints.

## High-level plan

1. **Package extension surfaces** — Desktop runtime, Hermes backend, provider modules, endpoint modules, optional Telegram sidecar, development tooling, and rollback-safe installer.
2. **Secure provider bootstrap** — select configured providers and mint short-lived Desktop credentials while retaining backend keys for server-hosted provider sessions.
3. **Freeze provider contracts** — versioned provider, transport, media, event, tool, error, and capability interfaces.
4. **Freeze endpoint contracts** — versioned frontend capabilities, participant identity, runtime placement, audio bridge, floor control, and recovery.
5. **Implement Desktop duplex** — OpenAI WebRTC and xAI WebSocket/AudioWorklet with bounded playback and cancellation.
6. **Normalize events** — map provider and endpoint events into stable participant, transcript, interruption, error, and tool-call records.
7. **Bridge Hermes tools** — enforce initiator/participant scope and linked-channel approvals before returning results to the provider.
8. **Preserve continuity** — record finalized turns with endpoint/provider/participant attribution in dedicated Hermes voice sessions.
9. **Productize Desktop UX** — capability-driven provider settings, call controls, diagnostics, accessibility, and recovery.
10. **Add Discord duplex** — reuse Hermes's existing Discord identity/authorization/voice connection through a stable continuous-media seam and bypass its STT/TTS path during duplex mode.
11. **Add Telegram voice plugins** — Bot API voice notes plus an optional isolated MTProto live-call sidecar with conservative tool policy.
12. **Harden and release** — provider/endpoint fixtures, DAVE and MTProto compatibility gates, live smoke tests, secret scanning, install/rollback, and drift reports.

The ordered implementation specs live in [`.agents/specs/`](.agents/specs/).

## Core decisions

- **Provider and endpoint contracts are independent:** shared orchestration connects any compatible human endpoint to any compatible duplex provider.
- **Runtime placement is negotiated:** Desktop uses direct ephemeral connections; Discord and Telegram live media use trusted server provider sessions.
- **Capability negotiation replaces name branching:** transport, codecs, VAD, cancellation, attribution, live/store-and-forward mode, and recovery are explicit.
- **Discord is genuinely duplex:** participant audio is streamed continuously to the provider; provider audio is paced back into the channel; speech flushes output and cancels the response.
- **Telegram is split honestly:** Bot API voice notes are asynchronous. Live calls require an MTProto user/tgcalls sidecar and are opt-in experimental initially.
- **Voice presence is not tool authority:** other speakers never inherit the call initiator's consequential Hermes permissions.
- **Uncertain attribution stays uncertain:** endpoints downgrade capabilities and tool catalogs rather than inventing speaker identity.
- **Hermes owns actions:** provider function calls still pass through Hermes policy, approvals, middleware, and bounded result shaping.
- **No raw audio by default:** only finalized transcripts, tool records, participant attribution level, and call metadata persist.
- **Pinned production behavior:** model, provider adapter, endpoint adapter, voice protocol, and native call-engine versions do not auto-upgrade.
- **Plugin first:** missing stable Hermes media seams are proposed upstream; stable installation does not patch the user's checkout.

## Repository layout (target)

```text
.
├── core/
│   └── src/                     # call router, contracts, media bridge, floor control
├── desktop/
│   └── src/
│       ├── endpoint/            # microphone, speaker, call UI
│       └── providers/
│           ├── openai/          # WebRTC + OpenAI protocol
│           └── xai/             # WebSocket + AudioWorklet PCM
├── backend/
│   └── src/hermes_duplex_voice/
│       ├── core/                # call registry, policy, persistence
│       └── providers/           # OpenAI/xAI server runtimes + bootstrap
├── endpoints/
│   ├── discord/                 # Gateway voice endpoint plugin
│   ├── telegram-bot/            # Bot API voice-message endpoint
│   └── telegram-mtproto/        # isolated optional live-call sidecar
├── tests/fixtures/
│   ├── providers/
│   └── endpoints/
├── docs/
│   └── architecture.md
└── .agents/specs/
```

## Non-goals for the first stable release

- SIP/PSTN calling
- cross-platform conference bridging
- Discord video or Go Live
- Telegram private calls, RTMP publishing, or new E2E conference-call support
- treating Telegram voice messages as realtime duplex
- acoustic diarization of mixed anonymous audio
- arbitrary user-supplied provider/platform endpoints
- stealth recording or raw-audio retention
- exposing every Hermes tool to every voice participant
- silently executing consequential tools without Hermes approval policy

## References

- [OpenAI Realtime API with WebRTC](https://developers.openai.com/api/docs/guides/realtime-webrtc)
- [OpenAI Realtime API with WebSocket](https://developers.openai.com/api/docs/guides/realtime-websocket)
- [xAI Voice Agent API](https://docs.x.ai/developers/model-capabilities/audio/voice-agent)
- [xAI ephemeral tokens](https://docs.x.ai/developers/model-capabilities/audio/ephemeral-tokens)
- [Discord voice connections](https://docs.discord.com/developers/topics/voice-connections)
- [Telegram Bot API](https://core.telegram.org/bots/api)
- [Telegram `phone.joinGroupCall`](https://core.telegram.org/method/phone.joinGroupCall)
- [Hermes voice mode](https://hermes-agent.nousresearch.com/docs/user-guide/features/voice-mode)
- [Hermes plugins](https://hermes-agent.nousresearch.com/docs/user-guide/features/plugins)

## License

[MIT](LICENSE)
