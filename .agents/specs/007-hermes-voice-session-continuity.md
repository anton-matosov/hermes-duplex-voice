# Spec 007: Hermes voice-session continuity

## Objective

Persist finalized OpenAI voice turns and tool activity in a dedicated Hermes session without mutating a concurrently active text-agent session.

## Dependencies

- Specs 004–006.

## Deliverables

- Voice-session repository behind `hermes_compat.py`.
- One dedicated Hermes voice session per call.
- Bounded `POST /calls/{call_id}/finalize` ingestion or equivalent trusted backend finalization path.
- Final transcript and canonical tool-call/result mapping.
- Deduplication and idempotent call close.
- Verification that history lists, exports, and resumes through normal Hermes surfaces.

## Session model

Persist:

- finalized user/assistant text;
- canonical tool call/result adjacency;
- provider/model/voice and safe provider item/call IDs;
- call start/end, duration, interruption state, and terminal reason;
- linkage to the originating Desktop profile/session when safe.

Do not persist raw audio, SDP, authorization, data-channel payloads, provisional transcript deltas, or raw OpenAI events.

A call creates a dedicated non-running voice session. It never appends to the live text conversation that launched the panel. A later voice call may start a new linked session seeded from finalized history; it is not represented as a resumed OpenAI transport unless OpenAI confirms actual provider-session recovery.

The current Hermes plugin API has no public external session-writer surface. Track A therefore treats version-pinned `SessionDB.create_session()`/`append_message()` access inside `hermes_compat.py` as an explicit compatibility dependency unless Hermes adds a supported writer first. Test it against a disposable real database; do not issue ad-hoc SQL. Map only metadata fields the verified API actually supports—do not assume arbitrary provider metadata can be attached.

## Tests

- Final user/assistant records create valid Hermes history with strict role/tool ordering.
- Interrupted assistant output is marked honestly.
- Duplicate, retried, and out-of-order final batches do not duplicate messages or tools.
- Active text session is unchanged.
- Crash/close retry is idempotent and abandoned calls are finalized safely.
- Normal Hermes list/export/text resume works from a disposable profile.
- Stored-data scan rejects raw audio, credentials, SDP, provisional deltas, and raw provider dumps.

## Acceptance criteria

1. A closed voice call appears as valid dedicated Hermes history.
2. Retrying finalization cannot duplicate turns or tools.
3. The active text-agent session is never mutated concurrently.
4. Interrupted output is not represented as fully delivered.
5. Stored data contains no audio or credential material.

## Non-goals

- raw-audio history;
- automatic durable-memory extraction;
- provider-independent live migration;
- xAI resumption;
- channel participant attribution.
