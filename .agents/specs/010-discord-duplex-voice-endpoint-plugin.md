# Spec 010: Discord duplex voice endpoint plugin

## Objective

Implement a full-duplex Discord voice-channel frontend that streams authorized participant audio to the selected native duplex provider and streams provider audio back with barge-in, while preserving Discord identity and Hermes policy.

## Dependencies

- Specs 001–009

## Deliverables

- Packaged `discord_voice` gateway endpoint plugin and configuration schema.
- Stable Hermes Discord media extension seam for joining, PCM/Opus input, participant events, output enqueue/flush, and leave.
- Discord Voice Gateway v8 connection lifecycle through a maintained DAVE-capable library.
- Per-guild call coordinator, participant/floor controller, text-side transcript/status/approval integration, and `/duplex` commands.
- Recorded protocol/media fixtures and opt-in live smoke harness.

## Integration strategy

Hermes already has a Discord platform adapter that joins voice channels, receives per-user audio, detects silence, runs STT → agent → TTS, and writes status to the linked text channel. The duplex plugin reuses its bot identity, allowlists, role checks, command context, and voice connection; it does not open a second Discord bot session with the same token.

A small upstream Hermes extension seam is required because the current receive path and mixer rely on adapter-private state and private `discord.py` internals. The seam exposes:

- authorized join/leave and linked text-channel context;
- participant join/leave/speaking events with stable Discord IDs;
- continuous timestamped PCM or Opus frames before utterance buffering/STT;
- output frame enqueue plus immediate output flush;
- transport state/recovery and idempotent close.

Until that seam exists in a supported Hermes version, a mutually exclusive replacement adapter may be used for development. Stable installation must not patch the user's Hermes checkout or run both adapters simultaneously.

## Discord media and lifecycle

- Use Discord Voice Gateway version 8 and wait for both Voice State Update and Voice Server Update before voice negotiation.
- Use a maintained Discord voice implementation that supports current RTP encryption and DAVE/MLS E2EE, preferably through maintained `libdave` bindings. As of the targeted 2026 release, DAVE support is mandatory; fail closed rather than downgrading to an obsolete receive path.
- Support required `aead_xchacha20_poly1305_rtpsize` transport encryption and prefer `aead_aes256_gcm_rtpsize` when offered.
- The host must support bidirectional UDP through NAT/firewall and voice WebSocket heartbeats/resume.
- Discord input is participant-attributed Opus/PCM at 48 kHz; decode/resample only at the explicit media bridge.
- Convert provider output to Discord-compatible 48 kHz stereo PCM or Opus frames. Maintain 20 ms pacing, send speaking state before audio, and emit the required silence frames at output end.
- Exclude the bot's SSRC/audio and do not pause receive while output is playing. Echo handling relies on self-audio exclusion plus platform/user echo cancellation—not disabling barge-in.
- On an authorized participant's speech start during assistant output, flush queued Discord output immediately and cancel the provider response.

## Commands and UX

Register a non-conflicting command family in the linked text channel:

```text
/duplex join [provider-profile]
/duplex leave
/duplex status
/duplex provider [profile]
/duplex mute
/duplex unmute
```

`/duplex join` requires the invoker to be in the target voice channel, pass existing Discord user/role authorization, and have an endpoint with effective View Channel, Connect, and Speak permissions. Respect channel user limits; Move Members may bypass a full channel only when the configured bot actually has it. Stage channels are rejected initially unless a later capability implements suppression/request-to-speak and moderator flows explicitly. The status message identifies the active provider/model, linked text channel, participant policy, transcript policy, and whether tools are restricted. The bot visibly announces join, recovery, and leave; it never listens invisibly.

## Identity, tools, and concurrency

- Scope one duplex call to `(discord account, guild_id, voice_channel_id)` and one linked text channel.
- Preserve Discord user ID on every participant frame and finalized user turn.
- Resolve Hermes tool policy per authenticated initiating session. Other allowed speakers may converse, but they do not inherit the initiator's consequential tool authority.
- Initial multi-user policy uses one active-speaker floor. Overlap is surfaced in diagnostics/text, not mixed into unattributed input.
- Consequential tool requests require the normal Hermes approval in the linked text channel from an authorized approver.
- Default concurrency is one joined channel per guild and a configurable global ceiling with admission control.
- A fenced distributed lease owns each `(bot account, guild, voice channel)` call; workers are sticky where possible and duplicate joins fail closed.

## Tests

- Fixture tests for Voice Gateway v8 handshake, voice state/server ordering, heartbeat, buffered resume, close codes, speaking/SSRC mapping, RTP sequence gaps, reconnect, and DAVE transitions.
- Deterministic Opus/PCM conversion, packet pacing, queue bounds, bot-audio exclusion, output flush, and required silence frames.
- Authorization tests for allowed users, roles, guild isolation, linked-channel approvals, participant join/leave, and no authority inheritance.
- Barge-in latency test from speaking event to local flush/provider cancel.
- Failure tests for missing View Channel/Connect/Speak permissions, channel full/Move Members behavior, Stage rejection, UDP failure, DAVE unsupported, endpoint migration, bot move/kick, provider failure, duplicate-worker lease, and shutdown.
- Opt-in smoke in a dedicated test guild with two human accounts verifies join, bidirectional audio, interruption, transcript attribution, harmless tool execution, reconnect, and cleanup.

## Acceptance criteria

1. An authorized Discord user can start and end a native duplex call without activating Hermes's STT/TTS turn pipeline.
2. Provider audio and participant input flow continuously with bounded queues and participant attribution.
3. Barge-in stops local Discord output promptly and cancels the active provider response.
4. Discord credentials, voice tokens, DAVE material, provider keys, and raw audio never enter logs or durable history.
5. A second speaker cannot silently inherit the initiator's tool permissions.
6. The plugin coexists with Hermes text/voice-message behavior and uses a supported extension seam rather than checkout patches.

## Non-goals

- Discord DMs/group-DM calls
- video or Go Live
- unrestricted public-server listening
- simultaneous mixed-speaker inference
- implementing Discord RTP/DAVE cryptography from scratch when a maintained compatible library exists
