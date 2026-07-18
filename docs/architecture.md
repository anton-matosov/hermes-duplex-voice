# Architecture

## 1. Purpose

Hermes Duplex Voice adds low-latency, full-duplex, native speech-to-speech conversation to Hermes Desktop. OpenAI Realtime is the default provider; xAI Grok Voice is a second complete provider. The two deliberately use different browser transports—WebRTC and WebSocket—to force a durable internal boundary instead of an OpenAI-shaped UI with a renamed base URL.

The defining path is always:

```text
microphone audio → native duplex model → generated audio
```

Text transcripts are observability and continuity artifacts, not an intermediate inference pipeline.

## 2. Design principles

1. **Portable semantics, provider-specific wire protocols.** Shared code understands `user.speech_started`, not `input_audio_buffer.speech_started`; adapters own raw event names and payloads.
2. **Transport is not the provider.** WebRTC, data channels, WebSocket framing, audio capture, and playback implement replaceable transport/media contracts.
3. **Capabilities beat conditionals.** UI and orchestration branch on advertised capabilities—not `if provider === "xai"` scattered through the codebase.
4. **Do not flatten useful differences.** A small portable core is normalized; provider-only features remain accessible through namespaced extensions.
5. **Secrets stay trusted.** Permanent provider keys live only on the Hermes backend. The renderer receives a short-lived, call-bound authorization descriptor.
6. **Media stays direct.** Once a call is bootstrapped, media flows between Desktop and the selected provider, never through Hermes.
7. **Hermes remains the action boundary.** The voice model requests tools; Hermes resolves scope, approval, execution, and durable history.
8. **One authority per concern.** The provider owns the live model conversation, Desktop owns media/UI state, and Hermes owns policy and durable sessions.
9. **Fail closed.** Unknown tools, unsupported capabilities, malformed events, expired calls, and provider drift cannot widen access or silently degrade safety.
10. **Pin production behavior.** Production profiles use versioned models and adapter protocol versions. Floating `latest` aliases are discovery/development options only.

## 3. System context

```text
┌──────────────────────────────────────────────────────────────────────┐
│ Hermes Desktop renderer                                             │
│                                                                      │
│ Call UI ─► CallController ─► DuplexProviderRegistry                  │
│                    │                    │                            │
│                    │          ┌─────────┴──────────┐                 │
│                    │          │                    │                 │
│                    │    OpenAI adapter       xAI adapter             │
│                    │    WebRTC + data        WebSocket +             │
│                    │    channel              AudioWorklets           │
│                    │          │                    │                 │
│                    └──────────┴── normalized events ──────────────┐   │
│                                                                  │   │
│ ctx.rest() ──────────────────────────────────────────────────────┼───┼──┐
└──────────────────────────────────────────────────────────────────┼───┘  │
                                                                   │      │
                                                     authenticated │      │
                                                     plugin REST   │      │
                                                                   ▼      │
┌──────────────────────────────────────────────────────────────────────┐  │
│ Hermes backend                                                       │  │
│                                                                      │  │
│ Call API ─► BackendProviderRegistry                                  │  │
│                 ├─ OpenAI bootstrap ──► OpenAI client secrets        │  │
│                 └─ xAI bootstrap ─────► xAI ephemeral tokens         │  │
│                                                                      │  │
│ Hermes tool-policy bridge ─► dispatcher/approvals                    │  │
│ Voice-session repository ───► Hermes SessionDB                       │  │
└──────────────────────────────────────────────────────────────────────┘  │
                                                                          │
                 direct provider media/events                             │
Desktop adapter ◄═════════════════════════════════════════════════════════╝
       ├────────────────────────────► OpenAI Realtime API
       └────────────────────────────► xAI Voice Agent API
```

## 4. Stable internal contracts

Internal contracts are versioned independently from provider APIs. Breaking changes increment `contract_version`; additive fields do not. Desktop and backend exchange their supported version ranges during `/capabilities` and `POST /calls`.

### 4.1 Provider identity and model selection

A provider/model reference is structured, never inferred from a model-name prefix:

```ts
type ProviderId = 'openai' | 'xai';

type VoiceModelRef = {
  provider: ProviderId;
  model: string;              // versioned production model name
  adapterVersion: string;     // implementation compatibility version
};
```

Provider endpoints are compiled into trusted adapters. Configuration may select a known provider/model but may not supply arbitrary WebSocket/HTTP URLs; that prevents credential exfiltration and SSRF.

### 4.2 Capability descriptor

Every adapter advertises a descriptor returned by backend `/capabilities` and refined by call bootstrap:

```ts
type DuplexCapabilities = {
  transports: Array<'webrtc' | 'websocket'>;
  authentication: 'ephemeral_token';
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

The core asks for required capabilities before creating a call. Unsupported optional features are disabled visibly; unsupported required features reject bootstrap with a specific reason.

Initial matrix:

| Capability | OpenAI Realtime | xAI Grok Voice |
|---|---|---|
| Browser transport | WebRTC | WebSocket |
| Browser media | WebRTC tracks | AudioWorklet PCM stream/jitter buffer |
| Ephemeral auth | client secret | ephemeral token via WebSocket subprotocol |
| Default VAD | semantic VAD | server VAD |
| Manual response cancel | supported | supported |
| Automatic barge-in | WebRTC/provider managed | server VAD + local playback flush |
| Function tools | supported | supported |
| Transcript updates | provider deltas/finals | cumulative `updated` plus finals when configured |
| Conversation resumption | not assumed | opt-in xAI extension, bounded by provider expiry |
| Rate-limit events | advertised when available | not emitted |

This matrix is generated from adapter descriptors and tested; it is not duplicated in UI components.

### 4.3 Backend bootstrap adapter

```py
class BackendProviderAdapter(Protocol):
    id: ProviderId
    adapter_version: str

    def capabilities(self, config) -> DuplexCapabilities: ...
    def validate_profile(self, profile) -> ValidatedProviderProfile: ...
    async def create_client_authorization(
        self, call, profile, requested_capabilities
    ) -> ClientAuthorization: ...
    def redact_upstream_error(self, error) -> ProviderError: ...
```

`ClientAuthorization` is a tagged union. Shared code never assumes a field named `client_secret`:

```json
{
  "kind": "ephemeral_token",
  "value": "redacted-in-logs",
  "expires_at": 0,
  "transport": {
    "kind": "webrtc",
    "endpoint_id": "openai-realtime-calls"
  }
}
```

or:

```json
{
  "kind": "ephemeral_token",
  "value": "redacted-in-logs",
  "expires_at": 0,
  "transport": {
    "kind": "websocket",
    "endpoint_id": "xai-realtime",
    "subprotocol_prefix": "xai-client-secret."
  }
}
```

Actual endpoint URLs remain adapter-owned and allowlisted. The descriptor exposes only endpoint IDs and safe connection parameters.

### 4.4 Desktop provider adapter

```ts
interface DuplexProviderAdapter {
  readonly id: ProviderId;
  readonly adapterVersion: string;
  capabilities(): DuplexCapabilities;

  connect(args: ConnectArgs): Promise<DuplexSession>;
  updateSession(patch: PortableSessionPatch): Promise<void>;
  setMuted(muted: boolean): Promise<void>;
  sendToolOutputs(outputs: ToolOutput[]): Promise<void>;
  requestResponse(): Promise<void>;
  cancelResponse(reason: CancelReason): Promise<void>;
  close(reason: CloseReason): Promise<void>;

  onEvent(listener: (event: DuplexEvent) => void): Unsubscribe;
}
```

The adapter composes a transport and media implementation. It translates portable session configuration into provider wire configuration and validates any provider extension before use.

### 4.5 Transport and media contracts

```ts
interface DuplexTransport {
  open(auth: ClientAuthorization): Promise<void>;
  sendControl(event: ProviderClientEvent): Promise<void>;
  close(): Promise<void>;
  onControl(listener: (event: unknown) => void): Unsubscribe;
  onState(listener: (state: TransportState) => void): Unsubscribe;
}

interface AudioSource {
  start(format: AudioFormat, sink: (frame: AudioFrame) => void): Promise<void>;
  setMuted(muted: boolean): void;
  stop(): Promise<void>;
}

interface AudioSink {
  start(format: AudioFormat): Promise<void>;
  enqueue(frame: AudioFrame): void;
  flush(reason: 'barge_in' | 'cancel' | 'close'): void;
  stop(): Promise<void>;
}
```

WebRTC transports can bind media tracks directly and bypass frame callbacks. Frame-based WebSocket media uses the same lifecycle and backpressure signals.

### 4.6 Normalized domain events

Only stable semantics enter reducers, persistence, and tool orchestration:

```ts
type DuplexEvent =
  | { type: 'session.ready'; providerSessionId: string }
  | { type: 'user.speech_started'; at: number }
  | { type: 'user.speech_stopped'; at: number }
  | { type: 'transcript.user_update'; itemId: string; text: string; mode: 'delta' | 'cumulative' }
  | { type: 'transcript.user_final'; itemId: string; text: string }
  | { type: 'response.started'; responseId: string }
  | { type: 'audio.output_started'; responseId: string }
  | { type: 'transcript.assistant_update'; itemId: string; text: string; mode: 'delta' | 'cumulative' }
  | { type: 'transcript.assistant_final'; itemId: string; text: string }
  | { type: 'response.interrupted'; responseId: string; locallyFlushed: boolean }
  | { type: 'response.completed'; responseId: string }
  | { type: 'tool.requested'; callId: string; name: string; argumentsJson: string; groupId?: string }
  | { type: 'provider.error'; error: ProviderError }
  | { type: 'provider.extension'; namespace: ProviderId; name: string; payload: unknown };
```

Every normalized event retains a redaction-safe provenance envelope outside persisted conversation content: provider ID, adapter version, provider event type, provider event ID, and receive sequence. Unknown raw events emit a metric/diagnostic and optional `provider.extension`; they never flow directly into UI or tool execution.

## 5. Provider implementations

### 5.1 OpenAI Realtime adapter

**Backend bootstrap**

- Read `OPENAI_API_KEY` only on the backend.
- Create a client secret through `POST https://api.openai.com/v1/realtime/client_secrets`.
- Bind the configured session/model where supported.
- Add a privacy-preserving `OpenAI-Safety-Identifier`.
- Return a short-lived authorization descriptor with endpoint ID `openai-realtime-calls`.

**Desktop connection**

1. Acquire microphone audio.
2. Create `RTCPeerConnection`, remote `<audio>`, and provider data channel.
3. Add the microphone track.
4. Create and set local SDP.
5. POST SDP directly to OpenAI's allowlisted Realtime calls endpoint using the ephemeral credential.
6. Set the SDP answer and wait for peer/data-channel readiness.

OpenAI's adapter owns all SDP, data-channel event names, WebRTC output buffering, and automatic unplayed-audio truncation behavior.

### 5.2 xAI Grok Voice adapter

**Backend bootstrap**

- Read `XAI_API_KEY` only on the backend.
- Create an ephemeral token through `POST https://api.x.ai/v1/realtime/client_secrets` with a bounded `expires_after.seconds`.
- Do not depend on session binding at token creation; send authoritative configuration after WebSocket open. This remains compatible with documented xAI deployments that differ on optional secret-bound session fields.
- Return a short-lived authorization descriptor with endpoint ID `xai-realtime` and the browser subprotocol prefix.

**Desktop connection**

1. Acquire the microphone and start an `AudioWorklet` capture pipeline.
2. Open the allowlisted `wss://api.x.ai/v1/realtime?model=<encoded-pinned-model>` endpoint with `xai-client-secret.<token>` as the WebSocket subprotocol. Browser code never attempts an `Authorization` header.
3. Send `session.update` with voice, instructions, `server_vad`, tools, and explicit audio formats.
4. Use binary 24 kHz mono PCM16 for initial input/output transport to avoid base64 overhead; the adapter may negotiate another advertised format later.
5. Resample browser input to the configured rate in the worklet path, packetize bounded frames, and apply WebSocket backpressure.
6. Feed binary output frames into an AudioWorklet jitter buffer; JSON frames remain lifecycle/events.
7. On server VAD interruption, flush queued local output immediately and mark the response interrupted. Use `response.cancel` for manual cancellation/non-VAD mode.
8. On close, clear queued PCM, terminate worklets, disconnect audio nodes, stop tracks, and close the WebSocket.

xAI transcript `updated` events are cumulative and may correct earlier text. The xAI adapter emits `mode: cumulative`; shared transcript assembly replaces the prior provisional text rather than appending it.

xAI session resumption is an optional namespaced capability. If enabled, the adapter stores the provider conversation ID only in active call state, reconnects within the provider's documented expiry, deduplicates replayed events, and still persists only finalized turns once. Core behavior must not require resumption.

### 5.3 Adding a future provider

A new provider must implement both bootstrap and desktop adapters, declare capabilities, provide protocol fixtures, pass the provider contract suite, and document its direct media data flow. It may not add provider checks to shared UI, tool, or persistence modules.

The acceptance gate is architectural: deleting the new adapter directory and registry entry must leave existing provider builds and tests passing.

## 6. Provider-neutral call flow

1. Desktop requests `/capabilities` and shows configured, ready providers.
2. User selects a provider profile and explicitly starts a call.
3. Desktop sends `POST /calls` with provider ID, adapter version, contract range, and required capabilities.
4. Backend validates the profile, creates a dedicated Hermes voice session, resolves Hermes tool policy, and invokes the selected bootstrap adapter.
5. Backend returns normalized call configuration, tools, capabilities, and short-lived authorization.
6. Desktop selects the matching adapter and verifies adapter/contract/capability compatibility before opening media.
7. Provider adapter establishes direct media and emits normalized events.
8. Tool requests travel to Hermes backend; results return through the provider adapter.
9. Final transcript/lifecycle events are batched to Hermes.
10. Close is idempotent across UI, adapter, transport/media, backend call registry, and durable session.

## 7. Backend API contracts

| Method | Route | Purpose |
|---|---|---|
| `GET` | `/capabilities` | Contract versions, provider readiness, models, voices, and safe feature descriptors |
| `POST` | `/calls` | Select provider, create voice session, resolve tools, mint short-lived authorization |
| `POST` | `/calls/{voice_call_id}/tools/{provider_call_id}` | Validate and execute one provider function call |
| `POST` | `/calls/{voice_call_id}/events` | Idempotently append normalized final transcript/lifecycle events |
| `POST` | `/calls/{voice_call_id}/close` | Finalize metadata and invalidate call state |

`POST /calls` includes:

```json
{
  "request_id": "uuid",
  "contract_versions": [1],
  "provider": "xai",
  "provider_adapter_version": "1.x",
  "required_capabilities": ["function_tools", "automatic_interruption"]
}
```

The response includes negotiated contract version, provider/model, capability snapshot and fingerprint, provider adapter version, transport descriptor, ephemeral authorization, tool catalog, Hermes session ID, call ID, and expiry. It never returns a permanent provider key or arbitrary endpoint URL.

Every mutating request uses an idempotency key. Backend call state binds provider ID, adapter version, capability fingerprint, authenticated profile, Hermes session, model, and provider tool-call IDs.

## 8. Tool bridge

Hermes function tools are provider-neutral internally. Each provider adapter serializes the server-authorized catalog into its wire schema and parses completed calls into `tool.requested`.

Flow:

1. Backend resolves exact enabled/disabled Hermes toolsets.
2. Backend produces canonical `PortableFunctionTool` schemas and a catalog fingerprint.
3. Selected adapter converts the catalog to provider session syntax.
4. Provider emits one or more completed calls.
5. Desktop posts each normalized request to the backend with provider and catalog fingerprints.
6. Backend rechecks scope and dispatches through Hermes.
7. Desktop sends outputs through the adapter; the adapter handles provider ordering rules before requesting continuation.

Provider-native search/MCP tools are disabled initially. They would bypass Hermes policy, logging, and approval boundaries. Only Hermes-backed custom function tools are exposed.

## 9. Durable continuity

Each call owns a dedicated Hermes voice session. Persisted metadata includes provider, versioned model, adapter version, capability fingerprint, voice, transport, duration, and terminal reason. Final user/assistant transcripts and canonical tool records are stored once; raw audio and ephemeral authorization are not.

Provider replay/resumption can repeat events. Deduplication therefore keys on `(provider, provider_session_id, provider_event_id or stable item/call id)` plus local sequence safeguards. Interrupted assistant output records only provider-confirmed/truncated content and delivery metadata.

## 10. Security model

- `OPENAI_API_KEY` and `XAI_API_KEY` never enter renderer state, storage, logs, Git, or API responses.
- Ephemeral credentials are held only in memory, redacted by type, and discarded after negotiation/close/expiry.
- Endpoint IDs resolve through compiled allowlists; user configuration cannot redirect credentials to arbitrary hosts.
- Browser WebSocket auth uses the documented ephemeral subprotocol, never query-string tokens.
- Provider ID/model/adapter/capability fingerprints are bound to each call and revalidated on tool/event requests.
- Tool names are selected from the server catalog and rechecked at execution.
- Provider-native tools remain off unless routed through an equivalent Hermes policy boundary.
- Raw audio is never persisted by default.
- Logs redact transcripts, tokens, authorization, SDP, tool payloads, WebSocket subprotocol values, and audio frames.
- Remote Hermes backends remain supported: only bootstrap, tool, and transcript metadata traverse Hermes; direct provider media originates on Desktop.

## 11. Reliability and API-drift strategy

### 11.1 Drift containment

- Raw provider payload types exist only inside provider directories.
- Adapters expose an implementation version and accepted upstream protocol/model range.
- Models are pinned in production; an explicit discovery job reports newer aliases without auto-switching.
- Captured, redacted protocol fixtures cover every required lifecycle and error path.
- Unknown event types and missing required events increment local diagnostics; they do not crash the call.
- A call records the exact provider/model/adapter/capability snapshot that created it.
- Startup probes verify credential endpoint shape and configured model availability without exposing credentials.
- Live canaries run opt-in and budget-capped per provider.
- Capability removal fails bootstrap instead of silently emulating unsafe behavior.

### 11.2 Backpressure and cleanup

- WebSocket input checks `bufferedAmount` and uses a bounded audio queue; overflow ends the call rather than growing memory indefinitely.
- Output uses a bounded jitter buffer and can flush in one worklet command on barge-in.
- Event/transcript batches are bounded and idempotent.
- All transports implement one idempotent close contract.
- Token expiry, permission denial, device loss, ICE failure, WebSocket close codes, provider rate limits, backend restart, and unsupported adapter versions map to actionable error categories.

### 11.3 Reconnection

OpenAI WebRTC does not assume resumability. xAI may advertise opt-in bounded resumption. The core asks the adapter for a `RecoveryPlan`:

- `not_resumable`: start a new provider/Hermes child session seeded from finalized transcript;
- `resume_transport`: reconnect with provider conversation ID and deduplicate replay;
- `retry_bootstrap`: mint fresh authorization before reconnecting.

The UI never labels a new conversation as resumed unless the adapter confirms provider context restoration.

## 12. Testing strategy

- **Provider contract suite:** run identical lifecycle, capability, cancellation, tool, redaction, and cleanup tests against OpenAI and xAI adapters.
- **OpenAI fixtures:** WebRTC/data-channel session, transcripts, interruption, tool call, errors, and completion.
- **xAI fixtures:** WebSocket JSON/binary framing, cumulative transcript correction, server VAD interruption, parallel tool calls, resumption replay, unsupported events, and close codes.
- **Media tests:** mocked WebRTC plus deterministic AudioWorklet/resampler/jitter-buffer tests with generated PCM fixtures.
- **Backend tests:** both secret endpoints, credential isolation, endpoint allowlist, provider selection, capability negotiation, idempotency, and error redaction.
- **Hermes integration:** provider-neutral tool and dedicated-session paths run once per adapter fixture.
- **Live smoke:** separate opt-in, budget-capped OpenAI and xAI tests covering bidirectional audio, barge-in, transcript, harmless Hermes tool, and cleanup.
- **Drift tests:** archived old/new fixtures prove additive unknown events survive and removed required semantics fail clearly.

## 13. Delivery phases

The implementation is divided into the ordered specifications in [`.agents/specs/`](../.agents/specs/). The provider contract is implemented before either transport, and both initial providers are required for the first release candidate.
