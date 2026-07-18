# Spec 008: xAI provider expansion

## Objective

Add xAI/Grok Voice to the already released Desktop product and extract the minimum provider seam justified by two working providers.

## Dependencies

- Track A release (Specs 001–007).

## Extraction rule

Start from the public behavior of the working `OpenAIRealtimeClient`. Refactor only what xAI must also implement. Do not retrofit channel endpoints, participant identity, server media bridges, or a provider marketplace.

The shared interface should cover only demonstrated common operations:

- connect/close;
- mute or input control;
- cancel active response;
- send tool output and continue;
- normalized lifecycle/transcript/tool/error events;
- provider-owned cleanup.

Transport and media remain inside provider implementations. OpenAI keeps WebRTC tracks; xAI owns WebSocket framing, AudioWorklet capture/resampling, bounded playback, backpressure, and local truncation/flush behavior.

## Deliverables

- Minimal provider interface and registry for `openai` and `xai` only.
- Backend xAI ephemeral-token broker with `XAI_API_KEY` server-side.
- Desktop xAI WebSocket client using subprotocol auth.
- Binary PCM capture/resampling and bounded playback worklet. Because runtime plugins load from a generated Blob URL and cannot rely on relative emitted assets, worklet source is inlined into the single bundle and loaded through an in-memory Blob URL unless Hermes first adds a supported asset-serving seam.
- Correct cumulative transcript replacement and provider-specific event fixtures.
- Provider selection/readiness UI.
- Shared behavior tests plus provider-owned transport tests.

## Tests

- Deleting xAI registration leaves OpenAI build/tests unchanged.
- Common call, event, tool, history, and cleanup behavior runs against both clients.
- xAI token expiry/auth redaction, subprotocol construction, model pinning, JSON/binary framing, queue bounds, `bufferedAmount`, local flush/truncation, and close codes.
- Bundle inspection proves there are no dynamic chunks, relative runtime imports, or separately emitted worklet assets.
- Cumulative transcript corrections replace provisional text rather than append.
- Opt-in xAI Desktop live smoke with budget cap.

## Acceptance criteria

1. Users can select either ready provider on Desktop.
2. OpenAI remains behaviorally unchanged.
3. Shared UI/tools/history import no provider wire type.
4. xAI queue growth is bounded and barge-in flushes local playback promptly.
5. The extracted interface contains no channel-only concept.

## Non-goals

- generic third-party provider loading;
- server-hosted provider sessions;
- Discord/Telegram;
- forcing WebRTC and PCM into one media abstraction;
- seamless mid-call provider handoff.
