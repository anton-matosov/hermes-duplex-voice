# Hermes Duplex Voice

Native, full-duplex speech-to-speech for Hermes, delivered as plugins.

> **Status:** reviewed architecture and implementation specifications; no production code yet.

## First product

The first implementation and release is deliberately narrow:

> **OpenAI Realtime + Hermes Desktop runtime plugin + Hermes backend plugin**

It uses direct WebRTC media, secure backend session initialization, natural turn-taking, real barge-in, and deterministic cleanup. Other providers and endpoints follow only after this path is built and tested.

```text
Hermes Desktop plugin ── authenticated session setup ──► Hermes backend plugin
        │                                                    │
        └════════════ direct WebRTC audio/events ════════════╪══► OpenAI Realtime
                                                             └── OPENAI_API_KEY
```

This is not record → STT → text model → TTS. OpenAI consumes and produces audio natively.

## Delivery order

### Track A — Desktop/OpenAI release

1. Verify Hermes Desktop/backend plugin seams and create reversible packaging.
2. Build secure OpenAI WebRTC session negotiation.
3. Ship a real Desktop tracer-bullet call: microphone, remote audio, mute, hang-up, and cleanup.
4. Add the minimal OpenAI event reducer, transcripts, and interruption state.
5. Harden UX, accessibility, installation/rollback, offline CI, and an opt-in budget-capped live smoke test; release Desktop/OpenAI v0.1 here.
6. Add one harmless read-only Hermes function tool only through a verified server execution-context seam.
7. Persist finalized turns into a dedicated Hermes voice session through an explicit versioned compatibility seam.

Desktop/OpenAI v0.1 ships at step 5. Steps 6–7 form the v0.2 Hermes-integration increment. Neither waits for another provider or channel.

### Track B — provider expansion

8. Add xAI/Grok Voice. Extract a provider contract from the working OpenAI implementation, then implement xAI WebSocket + AudioWorklet media behind it.

### Track C — endpoint expansion

9. Add Discord live voice. This is when reusable long-lived server-hosted realtime provider sessions, participant identity, floor control, audio conversion, linked-text approvals, and worker ownership become justified.
10. Add Telegram Bot API voice notes as an explicitly asynchronous mode with a bounded short-lived server provider session; reuse the Discord runtime when available, but do not require Discord abstractions.
11. Consider Telegram MTProto/tgcalls live calls as a separate opt-in project, not as a requirement for Telegram voice notes or the initial release.

## Core decisions

- **Vertical slice first:** no provider/endpoint framework before a real Desktop/OpenAI call.
- **Direct media:** Desktop audio does not traverse Hermes.
- **Backend-only permanent key:** renderer receives an SDP answer or short-lived authorization, never `OPENAI_API_KEY`.
- **OpenAI isolation, not premature neutrality:** wire types stay in `providers/openai`; the provider interface is extracted when xAI is implemented.
- **Desktop is not a channel:** participant mapping, floor control, codec bridges, sidecars, and distributed leases are deferred until Discord/Telegram need them.
- **Hermes owns actions:** provider function calls execute through identical server-resolved schema and tool scope.
- **Honest history:** persist only finalized turns in a dedicated voice session and mark interrupted output.
- **No raw audio by default.**
- **Plugins first:** use verified Hermes extension surfaces; stable installation never patches the user's Hermes checkout.

## Expansion targets

| Target | Mode | Runtime | Status |
|---|---|---|---|
| Hermes Desktop + OpenAI | live duplex | renderer + backend plugin | first release |
| Hermes Desktop + xAI | live duplex | renderer + backend plugin | next provider |
| Discord voice channel | live duplex | gateway/server runtime | later endpoint |
| Telegram Bot API voice note | asynchronous | gateway | later, not duplex |
| Telegram group call | live duplex | isolated MTProto/tgcalls sidecar | optional/experimental |

## Documentation

- [Architecture](docs/architecture.md)
- [Critical architecture review](docs/architecture-review.md)
- [Ordered implementation specs](.agents/specs/README.md)

## Non-goals for the first release

- xAI, Discord, or Telegram support;
- generic provider/endpoint registries or marketplaces;
- server media bridges and codecs;
- multi-participant identity/floor control;
- session recovery/resumption framework;
- distributed worker leases;
- arbitrary user-supplied upstream endpoints;
- raw-audio retention;
- every Hermes tool enabled by default;
- SIP/PSTN or conference bridging.

## References

- [OpenAI Realtime API with WebRTC](https://developers.openai.com/api/docs/guides/realtime-webrtc)
- [OpenAI Realtime conversations](https://developers.openai.com/api/docs/guides/realtime-conversations)
- [Hermes Desktop](https://hermes-agent.nousresearch.com/docs/user-guide/desktop)
- [Hermes plugins](https://hermes-agent.nousresearch.com/docs/user-guide/features/plugins)

## License

[MIT](LICENSE)
