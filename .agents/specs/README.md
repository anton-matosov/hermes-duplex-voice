# Implementation specifications

Product specifications 001–012 are ordered; a later product spec may depend on contracts established by every earlier product spec. Project-automation specifications 013–014 are operationally independent of that product delivery sequence.

| Order | Specification | Outcome |
|---:|---|---|
| 001 | [Repository scaffold and plugin packaging](001-repository-scaffold-and-plugin-packaging.md) | Buildable provider/endpoint-structured plugins, optional sidecar, and rollback |
| 002 | [Secure provider bootstrap broker](002-secure-realtime-session-broker.md) | Desktop ephemeral credentials, server provider sessions, and call lifecycle |
| 003 | [Provider, transport, and media contracts](003-provider-adapter-contracts.md) | Stable provider capability seams and reusable contract suite |
| 004 | [Voice endpoint and channel contracts](004-voice-endpoint-and-channel-contracts.md) | Frontend capabilities, placement, participant identity, media bridge, and floor control |
| 005 | [Desktop duplex voice runtime](005-desktop-duplex-voice-runtime.md) | OpenAI WebRTC and xAI WebSocket/AudioWorklet media |
| 006 | [Provider events and transcripts](006-provider-events-and-transcripts.md) | Normalized lifecycle plus delta/cumulative transcript handling |
| 007 | [Hermes function-tool bridge](007-hermes-function-tool-bridge.md) | Endpoint-aware, provider-neutral, scoped Hermes tool execution |
| 008 | [Hermes session continuity](008-hermes-session-continuity.md) | Durable provider/endpoint/participant-attributed history and replay dedupe |
| 009 | [Desktop product UX and configuration](009-desktop-product-ux-and-configuration.md) | Capability-driven provider UI, settings, diagnostics, accessibility |
| 010 | [Discord duplex voice endpoint plugin](010-discord-duplex-voice-endpoint-plugin.md) | Voice Gateway v8/DAVE full-duplex channel adapter with identity and barge-in |
| 011 | [Telegram voice endpoint plugins](011-telegram-voice-endpoint-plugins.md) | Bot API voice notes plus optional MTProto/tgcalls live sidecar |
| 012 | [Hardening, compatibility, and release](012-hardening-compatibility-and-release.md) | Drift-tested provider and endpoint release candidate |

## Project automation

These specs are coordinated through the [automated code-review user guide](../../docs/automated-code-reviews.md). They configure repository and Hermes infrastructure and do not gate Duplex Voice product delivery.

| Order | Specification | Outcome |
|---:|---|---|
| 013 | [Automated ChocoBot code reviews](013-automated-chocobot-code-reviews.md) | Signed, isolated, exact-head reviews run only for the requested PR triggers and publish once per head |
| 014 | [Create the ChocoBot GitHub App](014-create-chocobot-github-app.md) | A private least-privilege app identity is installed only on this repository and issues short-lived review credentials |

## Definition of done for every spec

- Scope and non-goals remain explicit.
- New behavior has automated tests.
- Provider and platform wire types remain inside their adapters.
- Shared behavior branches on capabilities rather than provider/endpoint IDs.
- Security-sensitive logs are redacted.
- Failure paths are visible, bounded, and recoverable.
- Public/versioned contracts are documented before dependent work.
- No implementation patches the user's Hermes checkout.
- Verification commands pass from a clean checkout.

## Cross-cutting constraints

1. Provider keys stay in trusted provider runtimes; Discord/Telegram credentials stay in their platform runtimes.
2. Desktop media connects directly to providers; channel media uses trusted server provider sessions.
3. OpenAI Desktop uses WebRTC; initial xAI Desktop uses WebSocket with explicit AudioWorklet media handling.
4. Endpoint mode, placement, participant attribution, and media capabilities are negotiated before call creation.
5. Tool execution is server-authoritative, participant-aware, and fail-closed.
6. Voice-channel presence never grants another participant's Hermes authority.
7. Raw audio is not stored by default.
8. Live calls use dedicated Hermes sessions rather than mutating an active text session.
9. Runtime Desktop output is one ESM file compatible with Hermes import restrictions.
10. Discord requires current Voice Gateway/DAVE compatibility; obsolete encryption paths fail closed.
11. Telegram Bot API voice messages are never represented as live duplex.
12. Telegram live calls require an isolated, explicitly authorized MTProto user session and conservative tool policy.
13. Production profiles pin provider models, adapter ranges, platform protocols, and native call-engine versions.
14. Capability removal fails bootstrap instead of silently changing safety or behavior.
15. External network calls and live paid/account tests are opt-in.
