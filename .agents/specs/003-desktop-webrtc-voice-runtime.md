# Spec 003: Desktop WebRTC voice runtime

## Objective

Establish and tear down a direct WebRTC speech-to-speech connection between Hermes Desktop and OpenAI Realtime using the backend-issued ephemeral client secret.

## Dependencies

- Spec 001
- Spec 002

## Deliverables

- `RealtimePeer` transport abstraction around browser media and WebRTC primitives.
- Composer action and minimal call panel.
- Microphone permission, selection, mute, and remote audio playback.
- SDP negotiation against OpenAI's GA `/v1/realtime/calls` endpoint.
- Deterministic cleanup and mocked browser tests.

## Connection sequence

1. User explicitly starts a call.
2. Client verifies secure media APIs and requests microphone permission.
3. Client calls backend `POST /calls`.
4. Client creates `RTCPeerConnection` and an autoplay remote `<audio>` element.
5. Client adds exactly one local microphone audio track.
6. Client creates the `oai-events` data channel.
7. Client creates and sets a local SDP offer.
8. Client POSTs SDP directly to `https://api.openai.com/v1/realtime/calls` with the ephemeral bearer credential and `Content-Type: application/sdp`.
9. Client validates the response and sets the remote SDP answer.
10. Call becomes connected only after the peer and data channel reach usable states.
11. On hang-up or failure, client closes the backend call and releases all local resources.

## Media requirements

- Request audio only; never request camera permission.
- Default constraints enable echo cancellation, noise suppression, and automatic gain control where available.
- Allow the user to select a specific input device after permission is granted.
- Mute by disabling the outbound audio track, not by destroying the call.
- Remote model audio is played from the WebRTC track; do not decode `response.output_audio.delta` in the WebRTC path.
- Handle autoplay rejection with a visible "Enable audio" action.
- Observe device removal and fail over deliberately rather than silently opening a different microphone.

## Runtime state

The transport emits typed events but does not own product labels:

```ts
type PeerState =
  | 'idle'
  | 'requesting_permission'
  | 'bootstrapping'
  | 'negotiating'
  | 'connected'
  | 'closing'
  | 'closed'
  | 'failed'
```

It exposes read-only handles for connection state, data-channel state, selected device, mute state, and terminal error category. Raw client secrets and SDP are never exposed through UI state or logs.

## Cleanup contract

Cleanup is safe to call repeatedly and must:

- disable and stop every local media track;
- detach the remote stream and remove the audio element;
- close the data channel;
- close the peer connection;
- remove all event listeners and device-change subscriptions;
- clear timers and queued events;
- zero references to client secrets/SDP where practical; and
- issue one best-effort backend close request.

Plugin hot-unload, profile switch, route unmount, backend disconnect, and window close all invoke the same cleanup path.

## Failure categories

Normalize browser/provider errors into actionable categories:

- permission denied;
- no input device;
- input device lost;
- backend bootstrap failure;
- ephemeral credential expired;
- SDP negotiation rejected;
- ICE/peer connection failed;
- data channel failed;
- remote audio blocked;
- backend connection lost; and
- unsupported runtime.

Do not automatically retry permission denial. Do not reuse an expired client secret. A failed established session offers "Start new call," not a false resume.

## Tests

- Mock `getUserMedia`, tracks, `RTCPeerConnection`, data channel, audio element, and fetch.
- Assert correct SDP request headers and that the standard API key is absent.
- Permission denial and missing device.
- Negotiation success and all state transitions.
- ICE failure and data-channel failure.
- Autoplay rejection recovery.
- Mute/unmute and device removal.
- Cleanup from every intermediate state with no leaked track or listener.
- Plugin unload and profile switch invoke cleanup once.

## Acceptance criteria

1. A manual development run can establish a direct WebRTC connection and exchange live microphone/model audio.
2. Barge-in is audible through OpenAI's WebRTC interruption behavior without a local TTS queue.
3. No audio bytes traverse the Hermes backend in the normal path.
4. Every close/failure path releases microphone and peer resources.
5. CI tests are deterministic and require neither a microphone nor OpenAI credentials.

## Non-goals

- rich transcript UI
- tool calls
- session-history persistence
- automatic reconnection of the same Realtime conversation
