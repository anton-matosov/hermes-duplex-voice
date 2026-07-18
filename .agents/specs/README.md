# Implementation specifications

These specifications are ordered. A later spec may depend on contracts established by every earlier spec.

| Order | Specification | Outcome |
|---:|---|---|
| 001 | [Repository scaffold and plugin packaging](001-repository-scaffold-and-plugin-packaging.md) | Buildable desktop/backend plugins and repeatable install/rollback |
| 002 | [Secure Realtime session broker](002-secure-realtime-session-broker.md) | Authenticated client-secret minting and call lifecycle |
| 003 | [Desktop WebRTC voice runtime](003-desktop-webrtc-voice-runtime.md) | Working direct duplex media connection |
| 004 | [Realtime event and transcript state machine](004-realtime-events-and-transcripts.md) | Deterministic conversation/UI state and finalized transcripts |
| 005 | [Hermes function-tool bridge](005-hermes-function-tool-bridge.md) | Scoped, guarded Realtime tool execution |
| 006 | [Hermes session continuity](006-hermes-session-continuity.md) | Durable, resumable voice-call history |
| 007 | [Desktop product UX and configuration](007-desktop-product-ux-and-configuration.md) | Native controls, settings, diagnostics, and accessibility |
| 008 | [Hardening, compatibility, and release](008-hardening-compatibility-and-release.md) | Tested, packaged release candidate |

## Definition of done for every spec

- Scope and non-goals remain explicit.
- New behavior has automated tests.
- Security-sensitive logs are redacted.
- Failure paths are visible and recoverable.
- Public contracts are documented before dependent work begins.
- The implementation does not patch the user's Hermes checkout.
- Verification commands pass from a clean checkout.

## Cross-cutting constraints

1. Standard OpenAI API keys remain backend-only.
2. WebRTC carries audio directly between Desktop and OpenAI.
3. Tool execution is server-authoritative and fail-closed.
4. Raw audio is not stored by default.
5. Voice calls use dedicated Hermes sessions rather than mutating an active text session.
6. Runtime desktop output is a single ESM file compatible with Hermes's import restrictions.
7. External network calls and live paid tests are opt-in.
