# Spec 003: Provider, transport, and media contracts

## Objective

Implement the stable internal seam that insulates UI, tools, and persistence from provider authentication, transport, audio, and event changes.

## Dependencies

- Specs 001–002

## Deliverables

- Versioned renderer/server `DuplexProviderAdapter`, `DuplexTransport`, `AudioSource`, and `AudioSink` interfaces.
- Typed capabilities, authorization, session configuration, normalized event, error, and recovery-plan unions.
- Provider contract test kit executed against OpenAI and xAI test adapters.
- Architecture rule enforcement and fixture versioning convention.

## Contract rules

- Raw provider payloads exist only below `providers/<id>`.
- Shared reducers consume only normalized events.
- Core branches on capability values, never provider names.
- Portable configuration contains instructions, tools, voice intent, turn-detection intent, and required semantics; adapters produce wire payloads.
- Provider extensions use `{namespace, name, version, payload}` and are optional to core.
- Every adapter reports ID, adapter version, accepted contract range, renderer/gateway placements, capabilities, and supported model/protocol range.
- Unknown additive upstream events are diagnostic, not fatal.
- Missing required semantics or incompatible payload changes fail connection clearly.
- All close/cancel/mute methods are idempotent.

Required portable semantics include session readiness, user speech start/stop, provisional/final user and assistant transcripts, response start/completion/interruption, output-audio start, function calls, errors, and terminal close.

Transcript updates explicitly identify `delta` versus `cumulative`; this prevents xAI corrective `updated` text from being appended like an OpenAI delta.

Capabilities cover runtime placement, transport, client/server authentication, codecs/rates, turn detection, automatic interruption, manual cancellation, transcripts, function tools, parallel calls, resumption, rate-limit events, and mutable session fields.

## Tests

The reusable provider contract suite injects fixtures and verifies:

- lifecycle ordering and legal state transitions;
- delta/cumulative transcript semantics;
- cancellation and local playback flush signal;
- single and grouped tool calls;
- provider error redaction and retry classification;
- unknown event survival;
- capability honesty;
- recovery-plan behavior; and
- complete listener/media/transport cleanup.

Add compile-time dependency checks proving shared UI/tool/persistence modules cannot import provider wire types. Add a deletion test/build variant for each adapter.

## Acceptance criteria

1. Both provider adapters can satisfy one contract suite across their advertised renderer and server placements despite different transports.
2. No raw provider event reaches shared UI, Hermes tools, or persistence.
3. Unsupported features are discoverable before call creation.
4. Provider-only features remain available through namespaced extensions without contaminating portable state.
5. A future provider requires registry entries and adapter modules, not edits across shared call code.

## Non-goals

- production WebRTC/WebSocket implementations
- provider-specific UI branding beyond registry metadata
- a dynamically loaded untrusted provider marketplace
