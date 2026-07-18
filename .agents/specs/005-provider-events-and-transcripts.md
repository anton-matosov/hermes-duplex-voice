# Spec 005: Provider events and transcripts

## Objective

Map OpenAI and xAI wire events into deterministic lifecycle/UI state and interruption-aware finalized transcript records.

## Dependencies

- Specs 003–004

## Deliverables

- OpenAI and xAI protocol adapters with redacted fixtures.
- Pure provider-neutral call reducer.
- Delta/cumulative transcript assembler and dedupe ledger.
- Idempotent backend event batches.
- Minimal live transcript display.

## Provider mapping

OpenAI mapping owns data-channel event names, response lifecycle, transcript deltas/finals, WebRTC interruption semantics, tool calls, and rate-limit updates where present.

xAI mapping owns WebSocket JSON events and differences including cumulative `conversation.item.input_audio_transcription.updated`, absent events such as `conversation.item.done`/`rate_limits.updated`, binary audio frames, grouped function calls, and optional resumption replay. Corrective cumulative updates replace provisional text.

Unknown events record provider type, adapter version, and count in redacted diagnostics. They do not crash media or enter persistence. Missing required lifecycle semantics produces a typed compatibility failure.

## Persistence ingestion

`POST /calls/{id}/events` accepts bounded normalized batches with contract version, provider/capability fingerprint, batch/event IDs, sequence, type, timestamp, and validated payload. Backend binds these to the active call, deduplicates, rejects raw audio/provider credentials, and permits a short closing grace period.

Persist only final transcripts. Interrupted assistant turns include delivery/truncation metadata so unheard text is not represented as fully delivered. Provider resumption replay is deduplicated using stable session/item/call IDs.

## Tests

- Fixture mapping for both providers: session, VAD, transcripts, response start/done/interruption, tools, error, and close.
- OpenAI delta assembly versus xAI cumulative correction.
- Unknown/additive events and removed-required-event failure.
- Resumption replay dedupe.
- Pure reducer transition table.
- Backend ordering, idempotency, batch/field limits, closing grace, and raw-audio rejection.
- Bounded client retry queue.

## Acceptance criteria

1. Shared reducers/components import no provider wire type.
2. OpenAI and xAI produce equivalent portable state for equivalent conversations.
3. Provider differences do not fabricate unsupported events.
4. Final records are ordered, deduplicated, and interruption-aware.
5. Malformed/unknown provider payloads cannot crash media or execute tools.

## Non-goals

- executing tools
- canonical Hermes message persistence
- transcript summarization/search
