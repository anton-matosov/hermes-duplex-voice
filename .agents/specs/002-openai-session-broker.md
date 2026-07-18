# Spec 002: OpenAI session broker

## Objective

Add the smallest authenticated backend API that can initialize an OpenAI Realtime WebRTC session while keeping `OPENAI_API_KEY` server-side.

## Dependencies

- Spec 001.

## Decision spike

Implement and measure OpenAI's unified WebRTC initialization first:

```text
Desktop {sdp: offer} JSON → authenticated plugin route → OpenAI /v1/realtime/calls → {sdp: answer} JSON
```

Hermes Desktop's current `ctx.rest()` bridge is JSON-only for ordinary requests and responses. The plugin API therefore wraps SDP in bounded JSON; only the backend-to-OpenAI leg uses multipart/SDP.

Record setup latency for local and remote Hermes backends. Use ephemeral client secrets only if the unified path is materially unsuitable; do not maintain both production paths without evidence.

## Deliverables

- `GET /config` with `apiVersion`, readiness, pinned model, voice, VAD mode, and safe limits.
- `POST /session` accepting bounded JSON `{ "sdp": "..." }` and returning `{ "sdp": "..." }` with `Cache-Control: no-store`.
- Backend-owned OpenAI session configuration.
- Compiled OpenAI endpoint; no caller-supplied URL.
- Privacy-preserving `OpenAI-Safety-Identifier` derived on the trusted backend where a stable user identity is available.
- Session-creation timeout, concurrency/rate limit, and redacted upstream error mapping.
- If selected by the spike, an equivalent ephemeral-secret response as a tagged internal result with expiry.

## Configuration

Keep first-release configuration narrow:

- enabled;
- pinned OpenAI Realtime model;
- output voice;
- instructions or prompt reference;
- semantic/server VAD choice supported by the pinned model;
- session timeout and maximum concurrent calls.

Credentials use Hermes secret/configuration workflows. Do not invent a generic provider profile system yet.

## Tests

- Successful mocked OpenAI multipart request and SDP answer.
- Backend auth and active-profile binding.
- Missing key/readiness response.
- Unsupported content type, malformed/oversized SDP, timeout, upstream 4xx/5xx/rate limit, and malformed answer.
- `no-store` on every secret/session response.
- Endpoint allowlist/SSRF resistance.
- Assert API key, authorization header, SDP, safety identifier source, and upstream body never appear in normal logs or client-visible errors.
- Bound concurrent session creation and idempotent cancellation when the client disconnects.

## Acceptance criteria

1. A real opt-in request produces an SDP answer usable by Desktop.
2. `OPENAI_API_KEY` never leaves the backend.
3. The renderer cannot choose an arbitrary endpoint, model, or permanent credential.
4. Errors are actionable but redacted.
5. Local and remote initialization measurements justify the chosen unified or ephemeral path.

## Non-goals

- WebRTC peer/media implementation;
- generic call registry;
- server-hosted provider sessions;
- xAI bootstrap;
- tools, transcripts, or Hermes history.
