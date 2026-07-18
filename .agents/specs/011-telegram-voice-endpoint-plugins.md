# Spec 011: Telegram voice endpoint plugins

## Objective

Add Telegram voice frontends while accurately separating Bot API voice-message exchange from true live group-call audio, which requires an explicitly authorized MTProto user session and call engine.

## Dependencies

- Specs 001–010

## Deliverables

- `telegram_voice_message` Bot API endpoint plugin for asynchronous native voice-note exchange.
- Optional `telegram_group_call` MTProto/tgcalls sidecar endpoint for live duplex Telegram video chats/voice chats.
- Separate capability descriptors, credentials, deployment units, policy defaults, and UX for the two modes.
- Linked Bot API control/status/approval channel for the live sidecar.
- Protocol fixtures, account-session security tests, and opt-in live smoke harnesses.

## Platform truth and mode separation

Telegram's HTTP Bot API can receive `Message.voice` and send `sendVoice`, but it has no method for a bot account to join a live group call. Telegram's current `phone.joinGroupCall` MTProto method is explicitly user-only and requires a join payload from a local tgcalls call engine.

Therefore:

| Endpoint | Authentication | Mode | True duplex |
|---|---|---|---|
| `telegram_voice_message` | existing Bot API token | asynchronous voice notes | no |
| `telegram_group_call` | MTProto `api_id`/`api_hash` plus an authorized user session | live group video chat/voice chat | yes |

The voice-message plugin must advertise `mode: voice_message`, `continuousInput: false`, `continuousOutput: false`, and no barge-in. Product UI and docs must not call it realtime or use it to satisfy the live-duplex release gate.

## Bot API voice-message plugin

Reuse Hermes's existing Telegram adapter, sender identity, chat/topic mapping, allowlist, download limits, and native Opus/OGG `sendVoice` delivery through a supported extension seam.

For each authorized voice message:

1. bind exact Telegram sender, chat, topic, and message IDs to a Hermes voice-message turn;
2. download within configured size/duration limits;
3. decode Opus/OGG and feed the complete utterance to a short-lived provider session using explicit manual commit/response;
4. collect the completed provider audio response, encode Opus/OGG, and send one native voice note plus optional finalized transcript;
5. close the provider session and deduplicate retries by Telegram update/message ID.

This path uses a native speech-to-speech model but remains store-and-forward. Existing Hermes STT/TTS behavior remains available as a separate mode.

## MTProto live group-call sidecar

The live endpoint is a separate trusted process because it requires native call-engine dependencies and user-account credentials that do not belong in the Bot API adapter.

- Authenticate interactively as a dedicated Telegram user account using `api_id`/`api_hash`; support 2FA without persisting plaintext passwords or login codes.
- Store the MTProto authorization session encrypted at rest with owner-only permissions and an explicit revoke/logout procedure.
- Join only allowlisted chats after an authorized text-side `/duplex join` request and a visible call announcement.
- Use a pinned, maintained tgcalls-compatible engine to generate WebRTC join parameters and exchange PCM frames.
- Initial support targets ordinary Telegram group video chats/voice chats. New E2E conference-call blockchain/key-management flows and RTMP broadcast mode are unsupported unless the adapter advertises and tests them explicitly.
- Do not attempt `phone.joinGroupCall` with a Bot API token or represent the user session as a bot account.
- The sidecar connects to the local Hermes duplex backend over mutually authenticated, loopback/Unix-socket IPC. It never receives provider keys or unrestricted Hermes credentials.
- Provider audio is streamed back through the call engine. On inbound speech during output, flush sidecar output and cancel the provider response when the endpoint can supply a reliable speech-start signal.

Track the authenticated MTProto account, visible `join_as` peer, and Hermes authorization subject as distinct identities. An eligible user/channel `join_as` does not turn a Bot API bot into a participant. Before publishing audio, verify mute/suppression state plus invite-hash/self-unmute or administrator-granted speaking permission.

## Identity and tool safety

Telegram call-engine participant attribution varies by call type and library. The adapter advertises `strong`, `best_effort`, or `none` based on verified runtime behavior; it must not invent speaker IDs.

Initial live-call policy is conservative:

- the authorized text-command initiator owns the Hermes session;
- all voice participants are conversational input only unless strongly mapped to an allowlisted Telegram user;
- no live speaker inherits full Hermes tool scope merely by being in the call;
- consequential tools require approval from an authorized user in the linked Bot API chat/topic;
- if attribution is unavailable, expose only a restricted read-only tool catalog or no tools;
- finalized records label unattributed or best-effort speakers honestly.

The sidecar identity is visible in the participant list. The plugin posts listening/leave status and never joins or records silently. Raw call audio is not persisted by default.

## Commands and UX

Control remains in the existing Telegram Bot API chat/topic:

```text
/duplex note [provider-profile]
/duplex join [provider-profile]
/duplex leave
/duplex status
/duplex provider [profile]
```

`/duplex join` is displayed only when the MTProto sidecar is configured and the active chat has a supported group call. Setup clearly explains that a dedicated Telegram user account—not the bot token—will appear in the call and may be subject to Telegram account policies.

## Tests

- Bot API fixtures for incoming voice, file limits, update dedupe, topic mapping, Opus decode/encode, native `sendVoice`, provider failure, and retry.
- Capability tests proving the Bot API endpoint cannot negotiate `live_duplex`.
- MTProto/tgcalls fixtures for login/session loading, join/leave, join-as identity, participant updates, PCM input/output, reconnect, mute state, and unsupported call types.
- Speaking-policy tests for suppressed/listener joins, invite-hash self-unmute, administrator action, and mismatch among account, visible, and Hermes identities.
- Secrets tests for session-file permissions/encryption, redacted login errors, logout/revocation, IPC authentication, and no credential leakage.
- Policy tests for allowlisted chats, authorized initiator, attribution downgrade, restricted tools, linked-chat approvals, and no speaker authority inheritance.
- Opt-in smoke uses a disposable test group and dedicated test account; it verifies visible join, bidirectional audio, barge-in when supported, transcript honesty, leave, and cleanup.

## Acceptance criteria

1. Telegram users can exchange provider-native voice notes through the existing bot with the mode clearly labeled asynchronous.
2. A configured dedicated MTProto user can join a supported Telegram group voice chat and exchange continuous audio through the shared duplex core.
3. Bot API credentials are never used or claimed to join group calls.
4. MTProto session material, provider keys, login codes, and raw audio are absent from logs and durable history.
5. Tool scope remains restricted unless participant identity is strongly verified, with consequential approvals routed to the linked text chat.
6. Disabling/removing the MTProto sidecar leaves ordinary Telegram bot messaging and voice notes intact.

## Non-goals

- pretending voice messages are realtime duplex
- joining Telegram live calls with a Bot API token
- private one-to-one Telegram calls
- RTMP livestream publishing
- E2E conference-call support in the initial release
- stealth recording, moderation, or account automation outside explicit call control
