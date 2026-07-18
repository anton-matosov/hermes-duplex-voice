# Spec 007: Hermes session continuity

## Objective

Persist normalized finalized turns, tool activity, and provider metadata in a dedicated Hermes session that remains valid across provider changes and call recovery.

## Dependencies

- Specs 002, 005, and 006

## Deliverables

- Voice-session repository behind Hermes compatibility adapter.
- Canonical normalized-event-to-Hermes message mapping.
- Provider replay dedupe and post-call finalization.
- Normal Hermes history/resume verification.

## Session model

Each call owns one dedicated Hermes session; never append into a concurrently active Desktop text session. Metadata records provider, versioned model, adapter/contract versions, capability fingerprint, transport, voice, provider session ID where safe, timestamps, duration, interruption count, and terminal reason.

No raw audio, credentials, SDP, WebSocket subprotocols, or raw provider dumps are stored.

Final user/assistant transcripts map to canonical messages. Interrupted assistant content carries delivery/truncation metadata. Tool calls/results preserve canonical adjacency. Lifecycle is metadata, not synthetic dialogue.

Idempotency keys combine provider, provider session, stable item/call ID, and event ID. xAI resumption replay and client retries produce no duplicates. Out-of-order records wait in a bounded reconciliation buffer.

A reconnect confirmed by provider recovery continues the same Hermes voice session. A non-resumable restart creates a linked child session seeded from finalized history and is labeled as a new provider conversation. Cross-provider continuation always starts a linked child session.

Prefer stable Hermes session APIs. If unavailable, `SessionDB` access is isolated, version-probed, and limited to the dedicated non-running session; no ad-hoc SQL.

## Tests

- Equivalent OpenAI/xAI normalized turns produce equivalent Hermes history.
- Provider/model/adapter metadata preserved.
- Interrupted output and tool adjacency.
- Duplicate, out-of-order, retry, and xAI replay handling.
- Confirmed resume versus new/child session behavior.
- Crash/restart abandonment and idempotent close.
- Normal Hermes list/export/text resume.
- Stored-data scan for forbidden secret/audio fields.

## Acceptance criteria

1. Both providers create valid, resumable Hermes voice histories.
2. Replay/retry cannot duplicate messages or tools.
3. Cross-provider continuation is honest and linked, not a fake transport resume.
4. No write touches the active text-agent session.
5. Stored content contains no raw audio or credentials.

## Non-goals

- live mutation of a running text agent
- raw-audio history
- automatic durable-memory extraction
- provider-independent live-session migration
