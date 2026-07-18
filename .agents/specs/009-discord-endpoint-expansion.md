# Spec 009: Discord endpoint expansion

## Objective

Add the first server-hosted, multi-user live endpoint and introduce endpoint/runtime abstractions only where Discord proves they are required.

## Dependencies

- Track A release.
- Preferably Spec 008 complete so provider/server placement boundaries are informed by two providers, but OpenAI-only Discord may proceed if prioritized.
- A verified supported Hermes Discord media seam.

## Precondition

Hermes must expose a generic supported seam for its existing Discord adapter to provide authorized join/leave, linked text context, participant events, continuous pre-STT audio frames, output enqueue/flush, transport state, and cleanup. If absent, propose that small upstream seam. Stable installation never patches the user's checkout or opens a duplicate bot login.

## Process ownership prerequisite

`hermes serve` (which hosts Desktop plugin REST routes) and the Hermes gateway (which owns Discord platform adapters) are separate processes. They cannot share an in-memory call registry or provider session by assumption. Before implementation, choose and test one owner for each Discord call—prefer the gateway or a dedicated voice worker for endpoint media and server provider transport—and define authenticated bounded IPC/API calls for tool execution, persistence, and control-plane status. Process loss and restart semantics are part of this contract.

## Abstractions introduced here

- endpoint session for continuous input/output and flush;
- renderer versus server runtime placement;
- server-hosted OpenAI provider session;
- explicit audio format conversion/queue bounds;
- participant identity and one-active-speaker floor;
- linked-text status and approval context;
- one-worker ownership for a joined channel.

Do not create Telegram-specific or arbitrary sidecar abstractions.

## Deliverables

- Discord endpoint plugin reusing Hermes bot identity, allowlists, role checks, command context, and voice connection.
- Current maintained Voice Gateway/RTP/DAVE-capable media implementation; no custom cryptography.
- Per-user source mapping and bot-audio exclusion.
- 48 kHz Discord input/output conversion with paced output and atomic flush.
- `/duplex join|leave|status` controls in linked text channel.
- Conservative participant tool policy and native linked-text approvals before consequential tools.
- Fixtures and opt-in dedicated-guild smoke.

## Tests

- Gateway/voice-state ordering, heartbeat/resume, SSRC mapping, packet gaps, DAVE transitions, pacing, silence frames, and cleanup.
- Authorization, guild/channel isolation, participant non-inheritance, overlap rejection, and linked-text approvals.
- Barge-in latency from admitted participant speech to Discord output flush and provider cancellation.
- Duplicate join/worker ownership, UDP failure, permission loss, move/kick, provider failure, shutdown, and reconnect.

## Acceptance criteria

1. Authorized Discord users converse continuously without Hermes's STT/TTS turn pipeline.
2. Participant identity is preserved where Discord proves it.
3. A second speaker never inherits the initiator's tool authority.
4. Barge-in atomically flushes endpoint output and cancels provider generation.
5. Plugin uses a supported Hermes seam and current maintained Discord security behavior.

## Non-goals

- Telegram or generic sidecars;
- mixed-speaker inference or diarization;
- DMs/group-DM calls, video, or Go Live;
- custom RTP/DAVE cryptography;
- unrestricted public-server listening.
