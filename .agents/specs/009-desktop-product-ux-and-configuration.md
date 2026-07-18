# Spec 009: Desktop product UX and configuration

## Objective

Deliver a native, accessible provider-aware voice experience driven by runtime capabilities rather than hardcoded provider assumptions.

## Dependencies

- Specs 001–008

## Deliverables

- Composer action, call panel, commands/keybinds.
- Provider/profile/model/voice/device settings.
- Capability-driven controls and diagnostics.
- Approval integration where proven.
- Setup/recovery/uninstall docs for OpenAI and xAI.

## UX requirements

The call panel shows provider, pinned model, voice, connection/listening/speaking/tool state, mute/device controls, transcript, duration, and recovery action. It distinguishes OpenAI WebRTC state from xAI WebSocket state only in diagnostics; normal labels remain portable.

Provider selection comes from `/capabilities`. Hide or disable unavailable features with a reason: semantic VAD, resumption, rate-limit detail, mutable voice, codecs, etc. Components may use capability flags and registry display metadata, not raw provider event names.

Settings support named provider profiles and backend-approved models/voices. Floating aliases require a development warning. Permanent keys use Hermes credential workflows and never appear in renderer settings.

Remote Hermes backends remain valid: microphone/media are local; bootstrap/tools/transcripts use authenticated `ctx.rest()`; provider media is direct from Desktop.

On recoverable failure, apply the adapter `RecoveryPlan`. Show “Resumed” only after provider confirmation; otherwise offer a new linked call from finalized transcript.

All controls are keyboard accessible, status is not color-only, final—not delta—turns are announced to screen readers, reduced motion is respected, and microphone permission follows an explicit gesture.

Diagnostics include plugin/Hermes/contract/provider/model/adapter versions, capability fingerprint, transport transitions/timings, safe provider request IDs, queue/backpressure metrics, and error category. Redact transcript, tokens, SDP, subprotocols, tool payloads, and audio.

## Tests

- Capability combinations and provider availability.
- Provider/profile switch and pinned-model warning.
- Unsupported feature gating and truthful recovery labels.
- Remote backend routing.
- Accessibility/keyboard/live-region behavior.
- Diagnostic redaction snapshots for both transports.
- Hot unload/profile switch/window close cleanup.

## Acceptance criteria

1. Users can select and call through either ready provider without a terminal.
2. UI contains no provider wire-protocol parsing.
3. Unsupported features are explained, never silently simulated.
4. Remote backend mode keeps media direct.
5. Accessibility and redaction tests pass.

## Non-goals

- replacing core dictation UI
- arbitrary user-installed providers
- mobile/SIP UI
- provider feature parity where capabilities differ
