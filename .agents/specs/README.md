# Implementation specifications

These specifications are ordered. A later spec may depend on contracts established by every earlier spec.

| Order | Specification | Outcome |
|---:|---|---|
| 001 | [Repository scaffold and plugin packaging](001-repository-scaffold-and-plugin-packaging.md) | Buildable provider-structured desktop/backend plugins and rollback |
| 002 | [Secure provider bootstrap broker](002-secure-realtime-session-broker.md) | OpenAI client secrets, xAI ephemeral tokens, and call lifecycle |
| 003 | [Provider, transport, and media contracts](003-provider-adapter-contracts.md) | Stable capability-driven seams and reusable contract suite |
| 004 | [Desktop duplex voice runtime](004-desktop-duplex-voice-runtime.md) | OpenAI WebRTC and xAI WebSocket/AudioWorklet media |
| 005 | [Provider events and transcripts](005-provider-events-and-transcripts.md) | Normalized lifecycle plus delta/cumulative transcript handling |
| 006 | [Hermes function-tool bridge](006-hermes-function-tool-bridge.md) | Provider-neutral, scoped Hermes tool execution |
| 007 | [Hermes session continuity](007-hermes-session-continuity.md) | Durable provider-attributed voice history and replay dedupe |
| 008 | [Desktop product UX and configuration](008-desktop-product-ux-and-configuration.md) | Capability-driven provider UI, settings, diagnostics, accessibility |
| 009 | [Hardening, compatibility, and release](009-hardening-compatibility-and-release.md) | Drift-tested OpenAI+xAI release candidate |

## Definition of done for every spec

- Scope and non-goals remain explicit.
- New behavior has automated tests.
- Provider wire types remain inside provider modules.
- Shared behavior branches on capabilities rather than provider IDs.
- Security-sensitive logs are redacted.
- Failure paths are visible, bounded, and recoverable.
- Public/versioned contracts are documented before dependent work.
- No implementation patches the user's Hermes checkout.
- Verification commands pass from a clean checkout.

## Cross-cutting constraints

1. `OPENAI_API_KEY`, `XAI_API_KEY`, and future permanent provider credentials stay backend-only.
2. Provider media flows directly between Desktop and the selected voice API.
3. OpenAI uses WebRTC; initial xAI uses WebSocket with explicit AudioWorklet media handling.
4. Tool execution is server-authoritative and fail-closed.
5. Raw audio is not stored by default.
6. Voice calls use dedicated Hermes sessions rather than mutating an active text session.
7. Runtime desktop output is one ESM file compatible with Hermes import restrictions.
8. Production profiles pin versioned models; floating aliases never auto-upgrade stable installs.
9. Provider capability removal fails bootstrap instead of silently changing safety/behavior.
10. External network calls and live paid tests are opt-in.
