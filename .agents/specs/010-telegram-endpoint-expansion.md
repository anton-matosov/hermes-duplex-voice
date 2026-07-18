# Spec 010: Telegram endpoint expansion

## Objective

Add Telegram voice capabilities without conflating asynchronous Bot API voice notes with optional live MTProto group calls.

## Dependencies

- Track A release.
- Bot API voice notes may proceed independently of Discord.
- Live MTProto/tgcalls work should follow Spec 009 so server-hosted media and multi-user policy have a proven implementation.

## Part A: Bot API voice notes

This is an asynchronous endpoint, not duplex.

Deliver:

- reuse of Hermes Telegram bot identity, allowlist, sender/chat/topic mapping, download limits, and `sendVoice` through a supported seam;
- OGG/Opus decode → bounded complete audio input to a short-lived provider session → completed output → OGG/Opus `sendVoice`;
- strong sender attribution and update/message ID deduplication;
- explicit `voice_message` labeling with no barge-in or continuous-stream claims.

Tests cover file/duration limits, decode/encode, topic mapping, update retry, provider failure, temporary-file cleanup, and proof that the endpoint cannot satisfy `live_duplex`.

Part A has its own release gate and does not wait for Part B.

## Part B: Optional MTProto/tgcalls live sidecar

Treat this as a separate opt-in project. Re-evaluate demand, maintained library support, Telegram protocol state, account policy, and operational cost before implementation.

If approved, deliver:

- visible dedicated Telegram user account with interactive authorization and 2FA;
- encrypted owner-only session storage plus logout/revoke;
- allowlisted chats and linked Bot API join/leave/status/approval controls;
- isolated pinned tgcalls-compatible sidecar;
- mutually authenticated local IPC with call-scoped privilege;
- capability-truthful participant attribution and conservative/no-tool fallback;
- PCM bridge, output flush, provider cancellation, cleanup, and opt-in test-call smoke.

Keep authenticated MTProto account, visible `join_as` identity, and Hermes authorization subject distinct. Never use a bot token for `phone.joinGroupCall`.

## Acceptance criteria

### Part A

1. Authorized users exchange provider-native voice notes through the existing bot.
2. Product/docs clearly label the mode asynchronous.
3. Retries do not duplicate provider work or replies.
4. Temporary audio and credentials do not leak or persist.

### Part B

1. A dedicated visible user can join and leave a supported group call with continuous audio.
2. Attribution is never stronger than verified call-engine behavior.
3. Voice participants do not inherit text-command authority.
4. Sidecar/session/provider credentials and raw audio are absent from logs/history.
5. Removing the sidecar leaves Bot API messaging and voice notes intact.

## Non-goals

- calling Bot API voice notes realtime;
- joining live calls with a bot token;
- requiring Part B to ship Part A;
- private calls, RTMP, or unsupported E2E conference flows;
- stealth listening/recording;
- a generic cross-platform sidecar framework.
