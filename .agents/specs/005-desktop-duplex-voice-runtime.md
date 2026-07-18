# Spec 005: Desktop duplex voice runtime

## Objective

Implement the Desktop endpoint's direct, low-latency media for OpenAI over WebRTC and xAI over WebSocket/AudioWorklet behind the provider and endpoint contracts from Specs 003–004.

## Dependencies

- Specs 001–004

## Deliverables

- Shared permission/device manager and provider-neutral call controller.
- OpenAI `RTCPeerConnection`/data-channel transport.
- xAI browser WebSocket transport, PCM capture/resampler, playback jitter buffer, and backpressure.
- Minimal provider-aware call panel and deterministic cleanup.

## OpenAI path

Acquire one microphone track, create peer/data channel/remote audio, negotiate SDP directly with the allowlisted OpenAI Realtime calls endpoint using the ephemeral credential, and wait for peer plus channel readiness. WebRTC owns output media and provider-aware unplayed-audio truncation.

## xAI path

Open the allowlisted Realtime WebSocket with the ephemeral token prefixed in `Sec-WebSocket-Protocol`; never use query-token auth or a browser `Authorization` header. Send pinned model as encoded query data and authoritative `session.update` after open.

Use explicit binary 24 kHz mono PCM16 initially. An AudioWorklet captures/resamples microphone frames; another worklet plays a bounded jitter queue. Check `WebSocket.bufferedAmount`, bound input queues, and terminate on sustained overflow. JSON frames are controls; binary frames are output audio. On server-VAD interruption, flush playback immediately; use `response.cancel` for manual cancellation.

## Shared behavior

- Explicit user gesture before microphone permission.
- Echo cancellation/noise suppression/AGC where available.
- Mute without ending the call.
- Device selection/removal handling.
- Actionable permission, device, bootstrap, transport, auth-expiry, autoplay, backpressure, and provider-close errors.
- One idempotent cleanup path for hang-up, unload, profile switch, route unmount, and failure.
- Never log authorization, SDP, subprotocol token, or audio frames.

## Tests

- Mocked media, WebRTC, WebSocket, AudioContext/Worklet, and fetch.
- OpenAI negotiation and remote-track playback.
- xAI subprotocol auth, session ordering, JSON/binary demux, resampling, queue bounds, jitter flush, and close codes.
- Barge-in/cancel under both adapters.
- Mute, device loss, autoplay rejection, token expiry, and every intermediate cleanup state.
- Assert no backend audio requests and no leaked tracks/nodes/listeners.

## Acceptance criteria

1. Manual development calls exchange native duplex audio with both providers.
2. OpenAI audio uses WebRTC; xAI audio uses direct browser WebSocket.
3. xAI barge-in stops queued local playback promptly.
4. No raw audio traverses Hermes.
5. Close/failure releases all media and transport resources.

## Non-goals

- rich transcript UI
- tools
- durable session history
- transparent cross-provider handoff mid-call
