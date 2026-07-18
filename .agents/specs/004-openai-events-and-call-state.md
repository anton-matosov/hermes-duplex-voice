# Spec 004: OpenAI events and call state

## Objective

Add deterministic conversation state, minimal transcript UI, and interruption-aware finalized records without creating a universal event framework.

## Dependencies

- Spec 003.

## Deliverables

- OpenAI event parser isolated in `providers/openai/events.ts`.
- Small domain-event union used by the controller/UI.
- Pure conversation reducer.
- Transcript delta assembler keyed by stable OpenAI item IDs.
- Finalized in-memory turn ledger for later persistence.
- Redacted OpenAI event fixtures.

## Consumed domain events

Only normalize behavior the Desktop product uses:

- session ready/updated;
- user speech started/stopped;
- assistant response/audio started;
- user and assistant transcript delta/final;
- response interrupted/completed;
- function call ready (consumed in Spec 005);
- provider error/rate-limit status;
- terminal close.

Raw OpenAI payloads never reach React, tool execution, or persistence. Unknown additive events increment a redacted diagnostic counter. Malformed required payloads create a typed provider error; they cannot execute a tool or crash cleanup.

## Transcript rules

- Provisional deltas are display-only.
- Final records are keyed/deduplicated by provider item ID.
- Interrupted assistant output is marked interrupted and not represented as fully delivered.
- No raw event dump or audio is sent to the backend.
- Bounded in-memory transcript length prevents an unbounded call panel.

## Tests

- Captured/redacted fixtures for session, VAD, transcripts, response start/done/cancel, error, rate limit, and close.
- Delta ordering, duplicate final events, missing IDs, malformed JSON, unknown event survival, and bounded UI history.
- Reducer transition table for listening, user speaking, responding, assistant speaking, interruption, and failure.
- Barge-in state updates immediately and the finalized ledger remains honest.
- No raw wire event escapes the OpenAI module.

## Acceptance criteria

1. UI state is deterministic for supported OpenAI event sequences.
2. Provisional text is never persisted as final.
3. Duplicate/reordered final events do not duplicate the ledger.
4. Interrupted output is represented honestly.
5. The implementation remains OpenAI-specific at the wire boundary without speculative xAI capability fields.

## Non-goals

- backend persistence;
- provider-neutral contract suite;
- recovery/resumption;
- participant attribution;
- tools beyond parsing the final function-call signal.
