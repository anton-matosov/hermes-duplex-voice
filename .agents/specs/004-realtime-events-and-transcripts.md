# Spec 004: Realtime events and transcripts

## Objective

Convert OpenAI Realtime data-channel events into deterministic call/UI state, finalized transcript records, interruption-aware assistant turns, and a bounded backend event stream.

## Dependencies

- Spec 003

## Deliverables

- Version-isolated Realtime protocol adapter.
- Pure event reducer and typed domain events.
- Transcript assembly and deduplication.
- Idempotent backend event-batch endpoint.
- Minimal live transcript display.

## Protocol boundary

Raw OpenAI events may change independently of UI code. One adapter module must:

- validate known GA event shapes;
- map them to internal domain events;
- retain unknown event type names only for redacted diagnostics;
- reject malformed JSON without crashing the call; and
- keep model/version-specific event aliases out of reducers and components.

Representative domain events:

```ts
type CallEvent =
  | { type: 'session.ready'; realtimeSessionId: string }
  | { type: 'user.speech_started'; at: number }
  | { type: 'user.speech_stopped'; at: number }
  | { type: 'transcript.user_final'; itemId: string; text: string }
  | { type: 'response.started'; responseId: string }
  | { type: 'transcript.assistant_delta'; itemId: string; delta: string }
  | { type: 'transcript.assistant_final'; itemId: string; text: string }
  | { type: 'response.interrupted'; responseId: string }
  | { type: 'response.completed'; responseId: string }
  | { type: 'tool.requested'; callId: string; name: string; argumentsJson: string }
  | { type: 'provider.error'; category: string; retryable: boolean }
```

## Conversation state

The reducer derives mutually intelligible states:

- listening;
- user speaking;
- model thinking;
- model speaking;
- tool running;
- interrupted;
- failed; and
- closed.

Audio/connection state from Spec 003 remains separate. UI composes the two rather than inferring connection health from transcript events.

## Transcript policy

- Show assistant deltas for responsiveness but persist only finalized records.
- Persist user turns from final input-transcription events.
- Assign every record the provider item ID, response ID where applicable, local sequence, speaker, final text, and interruption/delivery metadata.
- Normalize whitespace without rewriting words.
- Empty or known-noise transcripts are omitted from persistence but may be counted diagnostically.
- A repeated final event is idempotent.
- If the assistant is interrupted, mark the turn interrupted. Persist only provider-confirmed final/truncated transcript content; never represent unplayed text as fully heard.
- Do not persist raw audio.

## Backend event ingestion

`POST /calls/{call_id}/events` accepts a bounded batch:

```json
{
  "batch_id": "uuid",
  "events": [
    {
      "event_id": "provider-or-client-stable-id",
      "sequence": 1,
      "type": "transcript.user_final",
      "occurred_at": "ISO-8601",
      "payload": {}
    }
  ]
}
```

The backend:

- verifies call ownership and active/closing state;
- validates event type, size, ordering, and payload;
- deduplicates batch and event IDs;
- accepts late final events during a bounded closing grace period;
- rejects raw audio fields; and
- returns accepted, duplicate, and rejected IDs.

The desktop keeps a bounded in-memory queue, flushes final events promptly, retries with backoff, and discards only after explicit acceptance or call-close timeout.

## Error and rate-limit behavior

- Provider `error` events are tied to outbound client `event_id` where possible.
- Rate-limit updates are visible in diagnostics without exposing account data.
- Unknown events do not fail the call unless they violate a required lifecycle invariant.
- Queue overflow ends the call with a clear local integrity error rather than silently losing unlimited state.

## Tests

- Fixture-driven adapter tests for session, VAD, transcripts, response completion, interruption, function call, errors, and unknown events.
- Pure reducer transition table tests.
- Delta assembly, duplicate finals, out-of-order events, empty transcripts, and interrupted responses.
- Backend validation, idempotency, close grace period, size limits, and raw-audio rejection.
- Client batching retry and bounded queue behavior.
- Contract test using captured, redacted GA Realtime event fixtures.

## Acceptance criteria

1. UI state tracks speaking/listening/thinking/tool activity without parsing raw events in components.
2. Final transcript turns are stable, ordered, deduplicated, and interruption-aware.
3. Backend retries cannot duplicate durable events.
4. Unknown or malformed provider events cannot crash the media connection.
5. No raw audio or credential material enters transcript payloads or logs.

## Non-goals

- executing tool calls (Spec 005)
- canonical Hermes message persistence (Spec 006)
- transcript search or summarization
