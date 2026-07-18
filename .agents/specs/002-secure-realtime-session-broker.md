# Spec 002: Secure Realtime session broker

## Objective

Add an authenticated backend call-lifecycle service that creates a dedicated Hermes voice session and mints a short-lived OpenAI Realtime client secret without exposing the standard API key.

## Dependencies

- Spec 001

## Deliverables

- Typed configuration loader and validation.
- OpenAI Realtime bootstrap client.
- In-memory call registry with expiry and idempotency.
- `POST /calls`, `GET /calls/{id}`, and `POST /calls/{id}/close` routes.
- Mocked OpenAI contract tests.

## Configuration

Behavioral configuration belongs under a single Hermes config block such as:

```yaml
plugins:
  hermes_duplex_voice:
    enabled: false
    model: gpt-realtime-2.1
    voice: marin
    turn_detection: semantic_vad
    max_call_minutes: 30
    toolsets: []
```

The exact nesting must be verified against Hermes's current plugin configuration convention during implementation. `OPENAI_API_KEY` is read from Hermes's secret environment only. Model and voice defaults live in one backend module and are returned to the client; the desktop must not duplicate them.

## `POST /calls` request

```json
{
  "request_id": "uuid",
  "profile": "default",
  "client": {
    "plugin_version": "0.1.0-dev",
    "supports_webrtc": true
  }
}
```

The authenticated backend profile is authoritative. A request may not select an arbitrary profile merely by changing JSON.

## `POST /calls` response

```json
{
  "call_id": "uuid",
  "hermes_session_id": "...",
  "client_secret": {
    "value": "ek_...",
    "expires_at": 0
  },
  "realtime": {
    "model": "gpt-realtime-2.1",
    "voice": "marin",
    "session": {}
  },
  "tools": [],
  "expires_at": "ISO-8601"
}
```

The response is `Cache-Control: no-store`. Logs show only the call ID and expiry, never the client-secret value.

## Session configuration

The backend builds the authoritative GA Realtime session payload:

- `session.type = realtime`;
- configured model and output voice;
- output modality set to audio;
- semantic VAD by default with automatic response creation and interruption;
- input transcription enabled only as needed for durable transcript events;
- concise voice-specific instructions;
- empty or low-risk tool list until Spec 005; and
- maximum duration below OpenAI's session limit and the configured local ceiling.

Call `POST https://api.openai.com/v1/realtime/client_secrets` with the standard API key and a stable, privacy-preserving `OpenAI-Safety-Identifier` derived server-side. Do not use the legacy beta header.

## Call registry

Each call record binds:

- call ID;
- authenticated Hermes profile;
- dedicated Hermes session ID;
- creation, expiry, close, and terminal timestamps;
- effective model/voice/toolset configuration;
- idempotency keys seen;
- accepted Realtime tool-call IDs; and
- lifecycle state: `created`, `active`, `closing`, `closed`, or `expired`.

For the first implementation, process-local state is acceptable. Backend restart invalidates live calls cleanly. Durable call metadata is addressed in Spec 006.

## Error behavior

- `401/403`: missing Hermes authentication or profile mismatch.
- `409`: conflicting replay of an idempotency key.
- `422`: unsupported model/voice/configuration.
- `424`: OpenAI credential absent or bootstrap dependency failed.
- `429`: local or OpenAI rate limit.
- `503`: Hermes compatibility probe failed.

Upstream error bodies are normalized and redacted.

## Tests

- Successful client-secret minting with captured request assertions.
- No-store response headers and log redaction.
- Missing credential, OpenAI timeout, malformed upstream JSON, and rate limit.
- Idempotent replay returns the same safe call descriptor without minting a second secret when still valid.
- Conflicting replay fails.
- Call expiry and idempotent close.
- Profile binding and safety-identifier hashing.
- Model/voice values come from backend configuration only.

## Acceptance criteria

1. A renderer can obtain a valid short-lived client secret through authenticated plugin REST.
2. The standard API key is never returned, persisted, or logged.
3. Every call is bound to a dedicated Hermes session and authenticated profile.
4. Call creation and close are idempotent and bounded.
5. Unit tests use no live network or paid API.

## Non-goals

- microphone capture or WebRTC
- Hermes tool execution
- final transcript persistence
- transparent live-call recovery after backend restart
