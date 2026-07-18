# Architecture

## 1. Purpose

Hermes Duplex Voice adds low-latency, native speech-to-speech conversation to multiple Hermes frontends. OpenAI Realtime is the default provider and xAI Grok Voice is a second complete provider. Hermes Desktop, Discord voice channels, Telegram voice messages, and an optional Telegram MTProto group-call sidecar are endpoint adapters above the same provider/session/tool core.

The defining live path is:

```text
participant audio → native duplex model → generated audio
```

Text transcripts are observability and continuity artifacts, not an intermediate inference pipeline. Telegram Bot API voice messages use the same native audio model but are store-and-forward and are never described as realtime duplex.

## 2. Design principles

1. **Providers and frontends are separate axes.** A provider adapter connects to a model; an endpoint adapter connects to Desktop, Discord, or Telegram.
2. **Portable semantics, platform-specific wire protocols.** Shared code understands participant speech and response interruption, not OpenAI, Discord, or Telegram event names.
3. **Transport is not identity.** WebRTC, WebSocket, Discord RTP/Opus, Telegram Bot API files, and tgcalls media implement replaceable transport/media contracts.
4. **Capabilities beat conditionals.** Orchestration branches on advertised capabilities, runtime placement, attribution, and formats—not provider or platform names.
5. **Do not flatten useful differences.** Provider and endpoint extensions remain namespaced while a small portable core stays stable.
6. **Choose the shortest secure media path.** Desktop connects directly to providers with ephemeral credentials. Server-hosted channel media terminates at the Hermes gateway/sidecar and uses server-to-server provider sessions.
7. **Secrets stay in their trust zone.** Permanent provider keys remain in the backend; Discord tokens remain in the gateway; Telegram MTProto sessions remain in the sidecar.
8. **Hermes remains the action boundary.** The model may request tools, but Hermes resolves scope, participant authority, approvals, execution, and durable history.
9. **Identity is not audio presence.** Joining a voice channel never grants the initiator's tool permissions to every speaker.
10. **Be honest about modality.** Voice messages cannot satisfy a live-duplex requirement, and uncertain speaker attribution is recorded as uncertain.
11. **Fail closed.** Unknown tools, unsupported encryption, unattributed consequential actions, malformed events, and protocol drift cannot widen access.
12. **Pin production behavior.** Production profiles pin model, provider adapter, endpoint adapter, voice protocol, and call-engine compatibility ranges.

## 3. Terminology and support matrix

- **Provider:** native duplex model service, initially OpenAI Realtime or xAI Grok Voice.
- **Endpoint:** human-facing audio surface, such as Desktop microphone/speaker or a Discord voice channel.
- **Runtime placement:** process that owns an adapter: renderer, Hermes gateway/backend, or isolated sidecar.
- **Call:** one provider conversation bound to one endpoint scope and one dedicated Hermes session.
- **Participant:** endpoint-native identity attached to input audio when the endpoint can prove it.

Initial endpoint matrix:

| Endpoint | Runtime | Input/output | Mode | Attribution | Initial status |
|---|---|---|---|---|---|
| Hermes Desktop | renderer | microphone/speaker | live duplex | strong local profile | required |
| Discord voice channel | gateway | Voice Gateway v8, RTP/Opus, DAVE | live duplex | strong per Discord user when mapped | required |
| Telegram Bot API voice message | gateway | OGG/Opus voice notes | store-and-forward | strong message sender | supported, not duplex |
| Telegram group call | isolated MTProto/tgcalls sidecar | live call-engine PCM | live duplex | capability-dependent | opt-in experimental |

Telegram's HTTP Bot API can receive and send voice messages but cannot join a live group call. Telegram currently marks `phone.joinGroupCall` as user-only and requires a local tgcalls-generated join payload. The live endpoint therefore uses a separately authorized, visible Telegram user account—not the bot token.

## 4. System context

```text
                                 ┌────────────────────────────────────┐
                                 │ Hermes duplex backend/core         │
                                 │                                    │
                                 │ CallRouter + capability negotiation│
                                 │ Provider registry                  │
                                 │ Endpoint registry                  │
                                 │ Floor/participant controller       │
                                 │ Hermes tools + approvals           │
                                 │ Session/event repository           │
                                 └──────┬───────────────┬─────────────┘
                                        │               │
                    authenticated REST  │               │ local IPC/in-process
                                        │               │
┌───────────────────────────────────────▼──┐     ┌──────▼───────────────────────┐
│ Hermes Desktop renderer                  │     │ Gateway/channel runtimes       │
│ Call UI + Desktop endpoint               │     │                                │
│ OpenAI WebRTC provider runtime           │     │ Discord endpoint               │
│ xAI WebSocket/AudioWorklet runtime       │     │ Telegram voice-message endpoint│
└───────────────┬──────────────────────────┘     │ Telegram MTProto sidecar       │
                │ direct ephemeral media/events  │ Server provider runtimes       │
                │                                └───────┬────────────────────────┘
                │                                        │ server WebSocket media
                ▼                                        ▼
        OpenAI / xAI voice APIs                   OpenAI / xAI voice APIs

External endpoint media:
Discord users ◄══ Voice Gateway/RTP/DAVE ══► Discord endpoint
Telegram users ◄══ Bot API voice notes ═════► Telegram message endpoint
Telegram users ◄══ MTProto/tgcalls media ═══► Telegram live sidecar
```

The backend and channel runtime may share a process, but contracts do not require that deployment. The Telegram live sidecar is isolated because native call-engine dependencies and MTProto user sessions have a different security and operational profile.

## 5. Stable internal contracts

Internal contracts are versioned independently from upstream APIs. Breaking changes increment `contractVersion`; additive fields do not. All runtimes exchange supported ranges and capability fingerprints before opening media.

### 5.1 Provider identity and capabilities

```ts
type ProviderId = 'openai' | 'xai';
type RuntimePlacement = 'renderer' | 'gateway' | 'sidecar';

type VoiceModelRef = {
  provider: ProviderId;
  model: string;
  adapterVersion: string;
};

type DuplexCapabilities = {
  placements: RuntimePlacement[];
  transports: Array<'webrtc' | 'websocket'>;
  clientAuthentication: Array<'ephemeral_token' | 'backend_key'>;
  inputAudio: AudioFormat[];
  outputAudio: AudioFormat[];
  turnDetection: Array<'semantic_vad' | 'server_vad' | 'manual'>;
  automaticResponse: boolean;
  automaticInterruption: boolean;
  manualCancellation: boolean;
  transcriptMode: 'delta' | 'cumulative' | 'final_only';
  functionTools: boolean;
  parallelToolCalls: boolean;
  resumableConversation: boolean;
  providerRateLimitEvents: boolean;
  mutableDuringSession: string[];
  extensions: Record<string, unknown>;
};
```

Known endpoints and provider URLs are compiled into trusted adapters. Configuration selects known IDs and pinned models; it cannot supply arbitrary credential destinations.

### 5.2 Endpoint identity and capabilities

```ts
type EndpointMode = 'live_duplex' | 'voice_message';

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

type ParticipantRef = {
  endpoint: string;
  accountId: string;
  conversationId: string;
  participantId: string;
  displayName?: string;
  authorizationSubject?: string;
};
```

The capability fingerprint is captured at call creation. A Telegram voice-message endpoint with `continuousInput: false` cannot negotiate a call requiring `live_duplex` or barge-in.

### 5.3 Provider runtime contract

```ts
interface DuplexProviderAdapter {
  readonly id: ProviderId;
  readonly adapterVersion: string;
  capabilities(): DuplexCapabilities;

  connect(args: ProviderConnectArgs): Promise<DuplexProviderSession>;
}

interface DuplexProviderSession {
  updateSession(patch: PortableSessionPatch): Promise<void>;
  writeInput(frame: AudioFrame): void;
  commitInput?(): Promise<void>;
  sendToolOutputs(outputs: ToolOutput[]): Promise<void>;
  requestResponse(): Promise<void>;
  cancelResponse(reason: CancelReason): Promise<void>;
  close(reason: CloseReason): Promise<void>;
  onAudio(listener: (frame: AudioFrame) => void): Unsubscribe;
  onEvent(listener: (event: DuplexEvent) => void): Unsubscribe;
}
```

A renderer implementation may bind WebRTC tracks directly instead of exposing frames. A gateway implementation uses bounded frame queues and server WebSockets. Provider wire types remain inside `providers/<id>`.

### 5.4 Backend provider bootstrap

```py
class BackendProviderAdapter(Protocol):
    id: ProviderId
    adapter_version: str

    def capabilities(self, config) -> DuplexCapabilities: ...
    def validate_profile(self, profile) -> ValidatedProviderProfile: ...
    async def create_client_authorization(
        self, call, profile, requested_capabilities
    ) -> ClientAuthorization: ...
    async def open_server_session(self, call, profile) -> ServerProviderSession: ...
    def redact_upstream_error(self, error) -> ProviderError: ...
```

`ClientAuthorization` is a tagged union. Shared code never assumes a field named `client_secret`. Server-hosted endpoints receive no client authorization descriptor and never receive provider keys; the trusted provider runtime opens the upstream session.

### 5.5 Endpoint adapter contract

```ts
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

A `MediaBridge` negotiates formats or inserts explicit resampler, channel mapper, jitter buffer, and codec stages. It never infers format from provider or endpoint names.

### 5.6 Normalized domain events

```ts
type DuplexEvent =
  | { type: 'session.ready'; providerSessionId: string }
  | { type: 'participant.joined'; participant: ParticipantRef }
  | { type: 'participant.left'; participant: ParticipantRef }
  | { type: 'user.speech_started'; participant?: ParticipantRef; at: number }
  | { type: 'user.speech_stopped'; participant?: ParticipantRef; at: number }
  | { type: 'transcript.user_update'; itemId: string; participant?: ParticipantRef; text: string; mode: 'delta' | 'cumulative' }
  | { type: 'transcript.user_final'; itemId: string; participant?: ParticipantRef; text: string }
  | { type: 'response.started'; responseId: string }
  | { type: 'audio.output_started'; responseId: string }
  | { type: 'transcript.assistant_update'; itemId: string; text: string; mode: 'delta' | 'cumulative' }
  | { type: 'transcript.assistant_final'; itemId: string; text: string }
  | { type: 'response.interrupted'; responseId: string; locallyFlushed: boolean }
  | { type: 'response.completed'; responseId: string }
  | { type: 'tool.requested'; callId: string; name: string; argumentsJson: string; groupId?: string }
  | { type: 'endpoint.error'; error: EndpointError }
  | { type: 'provider.error'; error: ProviderError }
  | { type: 'extension'; namespace: string; name: string; payload: unknown };
```

Every event retains a redaction-safe provenance envelope outside persisted conversation content: provider/endpoint IDs, adapter versions, upstream event type/ID, participant attribution level, and receive sequence. Unknown raw events emit diagnostics, never raw UI/tool input.

## 6. Provider implementations and placement

### 6.1 OpenAI Realtime

**Desktop renderer**

1. Backend creates a short-lived client secret with `OPENAI_API_KEY`.
2. Desktop creates `RTCPeerConnection`, microphone/remote tracks, and a provider data channel.
3. Desktop posts SDP directly to the allowlisted Realtime calls endpoint.
4. OpenAI's adapter owns data-channel events, WebRTC output buffering, and unplayed-audio truncation.

**Gateway/sidecar endpoint**

- The trusted backend opens OpenAI's documented server-to-server Realtime WebSocket using `OPENAI_API_KEY` in an authorization header.
- The media bridge sends bounded base64 audio events and decodes output audio events.
- The provider key stays inside the provider runtime; channel adapters receive only normalized audio/events.

### 6.2 xAI Grok Voice

**Desktop renderer**

1. Backend creates an ephemeral token with `XAI_API_KEY`.
2. Desktop opens the allowlisted xAI WebSocket using `xai-client-secret.<token>` as the subprotocol.
3. `AudioWorklet` capture/playback uses bounded binary 24 kHz mono PCM16 queues.
4. The adapter applies authoritative `session.update`, handles cumulative transcript corrections, and flushes playback on interruption.

**Gateway/sidecar endpoint**

- The trusted backend opens the server WebSocket with permanent-key authentication supported by the xAI server integration.
- The same normalized adapter contract handles JSON controls, binary PCM, server VAD, cancellation, tool calls, and optional bounded conversation resumption.
- Channel adapters never see `XAI_API_KEY` or xAI authentication headers.

### 6.3 Future providers

A new provider implements renderer and/or server placements honestly, declares capabilities, provides redacted fixtures, and passes the provider contract suite. It may not add provider checks to endpoint, tool, UI, or persistence modules.

## 7. Endpoint implementations

### 7.1 Hermes Desktop

Desktop owns the call UI, microphone/device manager, local audio sink, and ephemeral provider connection. OpenAI uses direct WebRTC; xAI uses direct WebSocket plus AudioWorklets. Backend traffic is bootstrap, tool, event, and close metadata only.

### 7.2 Discord voice endpoint

Hermes already has a turn-based Discord voice path: per-user receive, silence detection, STT, ordinary agent execution, and TTS playback. Duplex mode reuses the existing bot identity, allowlists, role checks, command context, and voice connection through a stable upstream media seam; it bypasses utterance buffering/STT/TTS while active.

The endpoint:

- joins through Voice Gateway v8 and waits for both voice state/server events;
- requires Connect/Speak and bidirectional UDP reachability;
- uses a maintained current Discord voice implementation with RTP encryption and mandatory DAVE E2EE support;
- receives per-user Opus/PCM with SSRC-to-user mapping and excludes bot audio;
- converts provider output to paced Discord 48 kHz stereo PCM/Opus;
- sends speaking state before output and required silence frames after it;
- uses one active-speaker floor per channel;
- flushes Discord output and cancels the provider on authorized participant barge-in;
- links transcript, status, and approvals to the text channel that issued `/duplex join`.

Initial scope is one call per guild with a configurable global limit. The plugin does not open a second gateway login using the same bot token. If Hermes lacks the stable frame/event/output seam, development may use a mutually exclusive replacement adapter; stable install never patches the user's checkout.

### 7.3 Telegram Bot API voice-message endpoint

This endpoint reuses the existing Hermes Telegram bot identity, chat/topic mapping, allowlist, downloads, and `sendVoice` delivery. Each authorized OGG/Opus message opens a short provider session in manual-commit mode, receives a completed audio response, sends one native voice note, and closes.

It advertises `voice_message`, not `live_duplex`; there is no continuous stream, live interruption, or sub-second output. Telegram update/message IDs provide deduplication and strong sender attribution.

### 7.4 Telegram MTProto group-call endpoint

True Telegram live voice requires a separate MTProto user session and local tgcalls-compatible engine:

- interactive authorization as a dedicated user using `api_id`/`api_hash` and optional 2FA;
- encrypted owner-only session storage with logout/revoke procedures;
- a visible participant identity and explicit text-side join announcement;
- allowlisted chats and an authorized Bot API `/duplex join` control path;
- mutually authenticated loopback/Unix-socket IPC to the Hermes duplex backend;
- PCM input/output through the shared media bridge;
- capability-truthful participant attribution.

Initial support targets ordinary group video chats/voice chats. RTMP publishing, private one-to-one calls, and newer E2E conference-call blockchain/key flows are unsupported until implemented and tested explicitly. The Bot API token is never passed to `phone.joinGroupCall`.

Because call-engine libraries may provide best-effort or no speaker mapping for some call types, live Telegram defaults to a restricted/no-tool catalog. Consequential actions require approval from the linked authorized text chat. Uncertain speaker attribution stays uncertain in transcripts.

## 8. Call routing, floor control, and lifecycle

### 8.1 Capability negotiation

1. Resolve endpoint instance and endpoint capabilities.
2. Resolve provider profile and runtime capabilities.
3. Select a shared runtime placement and compatible audio path.
4. Verify requested mode, cancellation, attribution, tools, and recovery requirements.
5. Bind endpoint scope, initiator authorization, participant policy, provider profile, Hermes session, and capability fingerprints.
6. Open endpoint and provider sessions; only then announce active listening.

Unsupported combinations fail before opening provider media. A Desktop-only WebRTC provider implementation cannot service Discord until it also advertises a gateway placement.

### 8.2 Active-speaker floor

The first release sends one participant at a time to one provider conversation:

1. endpoint speaking events nominate an authorized participant;
2. the floor controller grants the first free floor;
3. other simultaneous streams are dropped/marked overlap, never mixed into unattributed input;
4. endpoint silence/VAD or provider speech-stop releases the floor;
5. speech during assistant output flushes endpoint output and requests provider cancellation;
6. normalized transcript events retain the participant reference and attribution level.

This is a conversational channel, not speaker diarization or a conference mixer.

### 8.3 Recovery

Provider and endpoint recovery are independent. `RecoveryPlan` distinguishes:

- `resume_endpoint_transport`;
- `resume_provider_transport`;
- `retry_provider_bootstrap`;
- `new_provider_session_from_finalized_context`;
- `not_recoverable`.

A resumed Discord Voice Gateway does not imply provider conversation recovery, and vice versa. Replayed endpoint/provider events are deduplicated before persistence.

## 9. Backend and local IPC contracts

| Method | Route | Purpose |
|---|---|---|
| `GET` | `/capabilities` | Contract, provider, endpoint, model, voice, placement, and safe feature descriptors |
| `POST` | `/calls` | Create a Desktop or gateway call and bind endpoint/provider/Hermes policy |
| `POST` | `/calls/{call_id}/tools/{provider_call_id}` | Reauthorize and execute one provider function call |
| `POST` | `/calls/{call_id}/events` | Idempotently append normalized final transcript/lifecycle events |
| `POST` | `/calls/{call_id}/approvals` | Resolve linked endpoint approval identity/context |
| `POST` | `/calls/{call_id}/close` | Finalize metadata and invalidate call state |

`POST /calls` includes endpoint ID/instance/scope, runtime placement, initiator authorization subject, provider profile, adapter/contract ranges, and required capabilities. The response includes negotiated versions, capability fingerprints, tool catalog, Hermes session/call IDs, and—only for direct clients—short-lived provider authorization.

The Telegram sidecar uses an equivalent versioned protocol over mutually authenticated local IPC. It receives a call-scoped capability token, not global Hermes or provider credentials.

## 10. Tool and approval boundary

1. Backend resolves endpoint initiator and participant policy.
2. Hermes produces a canonical tool catalog and fingerprint.
3. The selected provider adapter serializes that catalog.
4. Provider emits a normalized tool request.
5. Backend rechecks call, endpoint, participant attribution, tool fingerprint, scope, and approval requirements.
6. Consequential requests route to an authorized text-side approver where the endpoint supports it.
7. Hermes dispatches through normal policy/middleware and returns a bounded result.
8. The provider adapter sends the output and resumes response generation.

Additional channel participants never inherit the initiator's unrestricted toolset. If speaker identity is best-effort/none, only a restricted read-only catalog or no tools is exposed. Provider-native tools remain disabled unless they traverse an equivalent Hermes policy boundary.

## 11. Durable continuity

Each live call owns a dedicated Hermes voice session. A voice-message exchange owns a bounded child turn/session according to platform session policy.

Persisted metadata includes:

- endpoint/account/conversation/call scope;
- provider, model, voice, provider/endpoint adapter versions;
- runtime placement and capability fingerprints;
- participant ID and attribution level on finalized user turns;
- linked text approval context;
- duration, recovery lineage, and terminal reason.

Final user/assistant transcripts and canonical tool records are stored once. Raw audio, Discord voice tokens/DAVE keys, provider authorization, Telegram login codes, and MTProto session material are not persisted in conversation history.

## 12. Security and privacy model

- `OPENAI_API_KEY` and `XAI_API_KEY` remain in trusted backend provider runtimes.
- Ephemeral Desktop credentials live only in memory and are discarded after negotiation/close/expiry.
- `DISCORD_BOT_TOKEN` remains in the Hermes gateway; voice tokens and DAVE key material are memory-only and redacted.
- Telegram Bot API tokens remain in the gateway. MTProto `api_id`/`api_hash` and encrypted user session remain in the isolated sidecar.
- Sidecar IPC is local-only and mutually authenticated with call-scoped least privilege.
- Provider and endpoint URLs are allowlisted; configuration cannot redirect credentials.
- Endpoint scope, initiator, participant policy, adapters, capabilities, tools, and provider model are bound to each call.
- Voice-channel presence is not authentication for tools.
- Discord/Telegram endpoints visibly announce join/listening/leave and do not record raw audio by default.
- Logs redact transcripts, tokens, SDP, subprotocols, RTP/PCM frames, tool payloads, participant private data, and session files.
- Raw audio recording requires a separate explicit consented feature; it is not part of this architecture.

## 13. Reliability and API-drift strategy

### 13.1 Drift containment

- Raw provider/platform payloads exist only inside their adapters.
- Provider and endpoint adapters expose implementation versions and accepted upstream protocol/library ranges.
- Production pins models, Discord Voice Gateway/DAVE compatibility, Telegram Bot API/MTProto layer, and tgcalls engine versions.
- Recorded redacted fixtures cover required lifecycle, media, identity, encryption transition, and error paths.
- Unknown additive events alert without crashing; missing required semantics fail clearly.
- Live canaries are opt-in and budget/account isolated.
- Deletion builds prove every provider and endpoint is independently removable.

### 13.2 Backpressure and cleanup

Every frame path has bounded queues, timestamps, overflow policy, and metrics. Sustained overflow ends the call rather than growing memory. Output sinks can flush atomically. All sockets, UDP transports, AudioWorklets, encoders/decoders, native call engines, timers, listeners, temp voice-note files, and participant maps share idempotent close semantics.

### 13.3 Failure taxonomy

Errors distinguish provider authentication/rate limit/model drift, endpoint permissions/encryption/media reachability, participant authorization, local codec/call-engine availability, sidecar authentication, and recoverable transport loss. User-facing status identifies which side failed without leaking upstream secrets.

## 14. Testing strategy

- **Provider contract suite:** lifecycle, capabilities, cancellation, tools, redaction, recovery, and cleanup for OpenAI/xAI renderer and server placements.
- **Endpoint contract suite:** mode honesty, placement/media negotiation, identity, floor control, barge-in, approval routing, recovery, and cleanup.
- **Desktop fixtures:** WebRTC/data channel and WebSocket/AudioWorklet paths.
- **Discord fixtures:** Gateway v8, voice state/server ordering, heartbeat/buffered resume, close codes, RTP/Opus, SSRC mapping, DAVE transitions, pacing, and silence frames.
- **Telegram Bot API fixtures:** incoming/sent voice notes, topic mapping, size limits, Opus conversion, update dedupe, and proof that live capabilities are rejected.
- **Telegram MTProto fixtures:** secure session loading, user-only join, tgcalls parameters, participant attribution downgrade, PCM, reconnect, unsupported call types, logout, and sidecar IPC.
- **Policy tests:** guild/chat isolation, initiator authority, participant non-inheritance, restricted unattributed tools, and linked text approvals.
- **Live smoke:** budget/account-isolated OpenAI/xAI Desktop, Discord test guild, Telegram Bot API test chat, and opt-in Telegram test account/group call.
- **Drift tests:** archived old/new provider and platform fixtures tolerate additive changes and reject removed required behavior.

## 15. Delivery phases

The implementation is divided into the ordered specifications in [`.agents/specs/`](../.agents/specs/). Provider and endpoint contracts precede runtimes. Desktop and Discord live duplex plus Telegram Bot API voice messages are required release surfaces. Telegram MTProto live calls remain opt-in experimental until their account-security, attribution, and call-engine compatibility gates pass.
