# Implementation specifications

The specs are organized as releasable vertical slices. Track A must produce and validate OpenAI + Desktop before any general provider or endpoint framework is built.

## Track A — OpenAI + Hermes Desktop

| Order | Specification | Exit outcome |
|---:|---|---|
| 001 | [Verify extension seams and package plugins](001-verify-extension-seams-and-package.md) | A reversible paired plugin skeleton loads through supported Hermes surfaces |
| 002 | [OpenAI session broker](002-openai-session-broker.md) | Authenticated backend negotiation keeps the permanent key server-side |
| 003 | [Desktop OpenAI tracer bullet](003-desktop-openai-tracer-bullet.md) | A real bidirectional WebRTC call works and always cleans up |
| 004 | [OpenAI events and call state](004-openai-events-and-call-state.md) | Deterministic UI state, transcripts, interruption, and event fixtures |
| 005 | [Desktop/OpenAI v0.1 release hardening](005-desktop-v0.1-release-hardening.md) | Installable OpenAI/Desktop v0.1 with offline CI and live smoke |

**v0.1 ships here.** It does not depend on tools, persistence, or Specs 008–010.

### Desktop/OpenAI v0.2 integrations

| Order | Specification | Exit outcome |
|---:|---|---|
| 006 | [Hermes function-tool bridge](006-hermes-function-tool-bridge.md) | One harmless scoped Hermes tool executes exactly once and speech continues, if a valid host execution seam exists |
| 007 | [Hermes voice-session continuity](007-hermes-voice-session-continuity.md) | Finalized turns resume as valid dedicated Hermes history through an explicit compatibility seam |

Specs 006–007 enhance the shipped Desktop/OpenAI path. A missing Hermes host seam fails that integration closed; it does not withdraw v0.1 voice support.

## Track B — provider expansion

| Order | Specification | Exit outcome |
|---:|---|---|
| 008 | [xAI provider expansion](008-xai-provider-expansion.md) | Provider seam is extracted from two working implementations; xAI works on Desktop |

## Track C — endpoint expansion

| Order | Specification | Exit outcome |
|---:|---|---|
| 009 | [Discord endpoint expansion](009-discord-endpoint-expansion.md) | First server-hosted multi-user duplex endpoint validates endpoint abstractions |
| 010 | [Telegram endpoint expansion](010-telegram-endpoint-expansion.md) | Async Bot API voice notes ship separately; optional live sidecar has its own gate |

## Definition of done for every spec

- Scope and non-goals are explicit.
- New behavior has tests at the lowest useful layer and at its real integration seam.
- Security-sensitive logs and errors are redacted.
- Failure and cleanup paths are bounded and observable.
- No implementation patches the user's Hermes checkout.
- Default CI requires no provider key, paid API, microphone, or platform account.
- Verification commands pass from a clean checkout.
- An abstraction is introduced only when the current spec has at least two concrete implementations or an unavoidable compatibility boundary.

## Cross-cutting invariants

1. Native audio-to-audio; no hidden STT → text LLM → TTS substitution.
2. Desktop media is direct to the provider.
3. Permanent provider keys stay in trusted backend runtimes.
4. Runtime Desktop output is one ESM file using only Hermes-supported imports.
5. Tool schema generation and execution use identical server-resolved scope; failure exposes zero tools.
6. Raw audio is not stored by default.
7. Live calls use dedicated Hermes sessions rather than mutating an active text session.
8. Desktop/OpenAI is released and measured before xAI or channel work begins.
9. Telegram voice messages are never represented as live duplex.
