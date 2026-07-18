# Spec 004: Voice endpoint and channel contracts

## Objective

Implement the stable frontend seam that lets Hermes Desktop, Discord voice channels, Telegram voice messages, and an optional Telegram MTProto group-call bridge share one provider/session/tool core without sharing platform wire protocols.

## Dependencies

- Specs 001–003

## Deliverables

- Versioned `VoiceEndpointAdapter`, `VoiceEndpointSession`, `ParticipantRef`, and `EndpointCapabilities` contracts.
- A runtime-placement-aware call router for renderer-direct and gateway-hosted media.
- Audio format negotiation and conversion boundaries between endpoint and provider transports.
- Participant/floor-control, authorization-context, text-side-channel, and endpoint recovery contracts.
- A reusable endpoint contract test kit.

## Contract rules

- A **provider adapter** connects Hermes to a duplex model; an **endpoint adapter** connects a human-facing surface to Hermes. Neither knows the other's wire protocol.
- Endpoint IDs (`desktop`, `discord_voice`, `telegram_voice_message`, `telegram_group_call`) are structured configuration, not conditionals spread through orchestration.
- Every endpoint advertises whether it is `live_duplex` or `voice_message`. A voice-message endpoint must never claim realtime barge-in, continuous input, or sub-second output.
- Every call binds an endpoint scope, authenticated initiator, authorized participant policy, Hermes session, provider profile, capability fingerprint, and tool-policy fingerprint.
- Raw Discord RTP/gateway payloads and Telegram Bot API/MTProto/tgcalls payloads remain below their endpoint modules.
- Permanent platform credentials and MTProto session material never cross into provider adapters or renderer state.
- Frame queues, participant maps, speaking state, and output buffers are bounded and have idempotent cleanup.

```ts
type EndpointMode = 'live_duplex' | 'voice_message';
type RuntimePlacement = 'renderer' | 'gateway' | 'sidecar';

type ParticipantRef = {
  endpoint: string;
  accountId: string;
  conversationId: string;
  participantId: string;
  displayName?: string;
  authorizationSubject?: string;
};

type EndpointCapabilities = {
  mode: EndpointMode;
  placements: RuntimePlacement[];
  inputAudio: AudioFormat[];
  outputAudio: AudioFormat[];
  continuousInput: boolean;
  continuousOutput: boolean;
  participantAttribution: 'strong' | 'best_effort' | 'none';
  separateParticipantStreams: boolean;
  multiParticipant: boolean;
  localOutputFlush: boolean;
  textSideChannel: boolean;
  resumableTransport: boolean;
  extensions: Record<string, unknown>;
};

interface VoiceEndpointAdapter {
  readonly id: string;
  readonly adapterVersion: string;
  capabilities(): EndpointCapabilities;
  connect(args: EndpointConnectArgs): Promise<VoiceEndpointSession>;
}

interface VoiceEndpointSession {
  input(listener: (frame: ParticipantAudioFrame) => void): Unsubscribe;
  events(listener: (event: EndpointEvent) => void): Unsubscribe;
  writeOutput(frame: AudioFrame): void;
  flushOutput(reason: 'barge_in' | 'cancel' | 'close'): void;
  postStatus(message: EndpointStatusMessage): Promise<void>;
  close(reason: CloseReason): Promise<void>;
}
```

A `MediaBridge` chooses compatible formats or inserts an explicit resampler/channel mapper/codec stage. It never guesses audio geometry from endpoint or provider names.

## Runtime placement and routing

Desktop may connect directly to a provider with ephemeral credentials. Discord and Telegram media terminate in trusted gateway/sidecar processes, so their provider sessions are server-to-server:

- OpenAI uses its documented server WebSocket path with the backend API key kept in process memory.
- xAI uses its server WebSocket path with the backend API key kept in process memory.
- Channel adapters never receive provider API keys; the provider runtime and endpoint runtime communicate through in-process contracts or an authenticated, local-only media bridge.

The router rejects a provider/endpoint pair if no compatible runtime placement, media format, cancellation behavior, or security policy exists.

## Multi-participant floor control

The first release runs one provider conversation per joined channel and one active speaker at a time:

1. endpoint speaking events nominate a participant;
2. the floor controller grants the floor to the first authorized speaker;
3. frames from other speakers are dropped or marked overlap while the floor is held;
4. endpoint silence/VAD or provider speech-stop releases the floor;
5. an authorized speaker beginning while assistant output is active flushes endpoint output and requests provider cancellation;
6. participant identity is attached to normalized user transcript events and durable turns.

The floor policy is deterministic and configurable, but overlapping speakers are not mixed into one unattributed model input. An endpoint lacking strong attribution receives a restricted tool policy and requires text-side approval for consequential actions.

## Tests

The endpoint contract suite verifies:

- truthful live-duplex versus voice-message capabilities;
- renderer, gateway, and sidecar placement negotiation;
- format negotiation, resampling boundaries, queue limits, and output flush;
- participant join/leave/speaking mapping and bot/self-audio exclusion;
- deterministic floor grant, overlap rejection, release, and barge-in;
- endpoint scope to Hermes authorization/tool scope binding;
- text-side status and approval routing;
- transport recovery without duplicating finalized turns; and
- complete teardown of sockets, media tasks, queues, timers, and participant state.

## Acceptance criteria

1. Shared orchestration can run the same provider session through Desktop and a fake gateway-hosted endpoint without provider-name or endpoint-name branches.
2. A voice-message endpoint cannot satisfy a `live_duplex` requirement.
3. No participant can inherit another participant's Hermes authorization scope.
4. Unsupported media, attribution, or placement combinations fail before opening a provider session.
5. Deleting any endpoint adapter leaves provider/core builds and tests healthy.

## Non-goals

- acoustic diarization of mixed anonymous audio
- arbitrary untrusted endpoint plugins
- cross-platform audio conferencing
- treating voice-note exchange as realtime duplex
