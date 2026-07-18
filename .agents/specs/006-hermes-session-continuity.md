# Spec 006: Hermes session continuity

## Objective

Persist finalized voice turns, tool activity, and call metadata into a dedicated Hermes session so calls appear in history and can be continued safely after the Realtime connection ends.

## Dependencies

- Spec 002 dedicated session creation
- Spec 004 finalized event ingestion
- Spec 005 tool-call records

## Deliverables

- Voice-session repository behind the Hermes compatibility adapter.
- Canonical mapping from Duplex Voice records to Hermes messages.
- Post-call finalization and history verification.
- Optional explicit summary/memory handoff hook.

## Session ownership

Each voice call owns one dedicated Hermes session. It must not append into the active Desktop text session because that session may have an in-memory agent and strict role/tool-call invariants.

Required metadata:

- source: `desktop-duplex-voice`;
- profile;
- model and voice;
- call ID and Realtime session ID where safe;
- started, ended, duration, and terminal reason;
- interrupted/error counts; and
- plugin version.

No raw audio, standard credentials, ephemeral credentials, SDP, or full provider event dumps are stored.

## Message mapping

- Final user transcript → Hermes `user` message.
- Final assistant transcript → Hermes `assistant` message.
- Interrupted assistant transcript → assistant content plus structured interruption metadata where Hermes supports it; otherwise an unambiguous textual marker outside spoken content.
- Tool call/result → Hermes canonical assistant tool call and tool result representation where supported.
- Call lifecycle → session metadata or a compact system/event record, not synthetic conversation prose.

Preserve strict message ordering and tool-call/result adjacency. If an event arrives out of order, hold it in a bounded reconciliation buffer rather than writing invalid history.

## Idempotency

Use stable provider item IDs and tool call IDs as `platform_message_id`/dedupe keys where Hermes supports them. Otherwise maintain a plugin-side event ledger keyed by call and event ID. Reprocessing the same accepted batch must produce zero additional messages.

## Active-call durability

- Append finalized turns promptly rather than waiting until hang-up.
- Transactions commit complete message units.
- Backend crash may lose only unfinalized deltas and in-flight records.
- On restart, mark previously active calls abandoned and keep their completed transcript.
- Closing reconciles buffered events, records terminal metadata, and marks the session complete.

## Resume behavior

After close, the session must be visible through Hermes's normal session list and readable by session search/history. Resuming in text mode should load valid role alternation and tool history.

A new voice call seeded from an older transcript is a new Realtime session and a new Hermes child/linked session. It does not claim to resume OpenAI's expired media session.

## Summary and memory

Post-call summarization is opt-in and outside the live latency path. Provide an explicit action that can:

1. run a normal Hermes turn or supported memory hook against the finalized transcript;
2. display what will be stored;
3. respect the active profile's memory configuration; and
4. avoid storing transient call mechanics as durable memory.

Do not automatically send every transcript to a second model.

## Compatibility fallback

Prefer a stable Hermes session API. If none exists, use `SessionDB` only through the compatibility adapter and only for the dedicated non-running voice session. Verify schema/version before writes and fail closed if incompatible. Never manipulate SQLite directly with ad-hoc SQL.

## Tests

- User/assistant turn mapping and strict alternation.
- Interrupted assistant mapping.
- Tool-call/result adjacency.
- Duplicate and out-of-order event handling.
- Crash/restart abandonment and close idempotency.
- Session appears in normal Hermes list and exports correctly.
- Resume through Hermes loads without history repair.
- No raw audio/token/SDP in stored rows.
- Temporary-profile isolation.

## Acceptance criteria

1. Completed voice turns appear once in a dedicated Hermes session.
2. A closed voice session can be resumed as a normal text conversation.
3. Interruptions do not persist unheard assistant content as fully delivered.
4. Backend restart preserves finalized turns and marks abandoned calls honestly.
5. No write touches the concurrently active text session.

## Non-goals

- live synchronization into an already running text agent
- raw audio storage or playback history
- automatic durable-memory extraction
- pretending an expired OpenAI Realtime connection can resume
