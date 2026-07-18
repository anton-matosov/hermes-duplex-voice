# Spec 003: Desktop OpenAI tracer bullet

## Objective

Deliver the first real user value: a bidirectional native OpenAI Realtime call from Hermes Desktop with mute, hang-up, barge-in, and deterministic cleanup.

## Dependencies

- Specs 001–002.

## Deliverables

- Explicit Duplex Voice composer action.
- Compact call panel showing connection/listening/speaking/error state.
- `CallController` with one connection state machine and one resource owner.
- `OpenAIRealtimeClient` using `RTCPeerConnection`, remote autoplay audio, and `oai-events` data channel.
- Microphone acquisition after explicit user gesture.
- Mute by toggling the local track without ending the call.
- Idempotent close on hang-up, error, hot unload, route unmount, profile/connection switch, and window close.
- Opt-in real-audio smoke harness with strict time and spend cap.

## State machine

```text
idle → requesting_permission → negotiating → connected → closing → idle
 requesting_permission | negotiating | connected → failed → closing → idle
```

Every asynchronous completion checks the current generation/call ID so late permission, SDP, ICE, or channel events cannot revive a closed call.

## Cleanup contract

`close(reason)` can run repeatedly and must:

1. prevent new state/event publication;
2. abort pending backend requests;
3. stop every local media track;
4. close the data channel and peer connection;
5. detach and clear the remote audio source;
6. remove listeners/timers;
7. clear sensitive in-memory session data;
8. return UI to a truthful terminal/idle state.

Cleanup correctness is a release gate; waveform polish is not.

## Tests

- Mocked `getUserMedia`, peer connection, SDP, ICE, data channel, remote track, audio play, and backend request.
- Happy-path ordering from click to connected.
- Permission denial, no device, backend failure, malformed SDP answer, ICE failure/disconnect, data-channel failure, autoplay rejection, and hang-up in every intermediate state.
- Late async event suppression after close or a newer call.
- Mute/unmute track behavior.
- Hot unload/profile switch/window close cleanup with no tracks, peers, channels, audio sources, timers, or listeners left.
- Assert no backend raw-audio request and no SDP/token/audio logging.
- Live smoke verifies two-way audio, interruption, hang-up, and resource release.

## Acceptance criteria

1. A user can start, converse, mute, interrupt, and hang up from Desktop.
2. Audio is native duplex over direct WebRTC.
3. OpenAI automatic WebRTC interruption/truncation is observed in a live call.
4. Every terminal path releases all media and transport resources.
5. No tools, persistence, xAI, or channel abstractions were required to deliver the call.

## Non-goals

- rich transcript UI;
- Hermes tools;
- durable history;
- device picker or provider selector;
- recovery/resumption;
- generic provider or endpoint contracts.
