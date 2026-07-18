# Spec 002: Secure provider bootstrap broker

## Objective

Create provider-neutral call lifecycle APIs and backend adapters that mint short-lived OpenAI and xAI browser credentials without exposing permanent keys.

## Dependencies

- Spec 001

## Deliverables

- Versioned `BackendProviderAdapter` and tagged `ClientAuthorization` models.
- OpenAI bootstrap using `OPENAI_API_KEY` and `/v1/realtime/client_secrets`.
- xAI bootstrap using `XAI_API_KEY` and `/v1/realtime/client_secrets`.
- Provider/model profiles, capability negotiation, call registry, expiry, and idempotency.
- `POST /calls`, `GET /calls/{id}`, and `POST /calls/{id}/close`.

Example configuration:

```yaml
plugins:
  hermes_duplex_voice:
    enabled: false
    default_profile: openai-default
    profiles:
      openai-default:
        provider: openai
        model: gpt-realtime-2.1
        voice: marin
      xai-default:
        provider: xai
        model: grok-voice-think-fast-1.0
        voice: eve
```

Production profiles require versioned models. Floating `latest` aliases require an explicit development override. Exact Hermes config nesting is verified during implementation.

`POST /calls` supplies provider/profile, supported contract versions, desktop adapter version, and required capabilities. Backend authentication—not request JSON—selects the Hermes profile.

The response contains call/Hermes-session IDs, negotiated contract, provider/model/adapter, capability snapshot/fingerprint, tools, transport descriptor, and an in-memory ephemeral authorization value. It is `Cache-Control: no-store`.

OpenAI configuration may be bound during secret creation where supported. xAI token creation uses bounded `expires_after` and does not require secret-bound session fields; authoritative `session.update` occurs after connect. Provider endpoints are compiled allowlist IDs, never caller URLs.

## Tests

- Successful mocked bootstrap and exact upstream request per provider.
- Missing key, timeout, malformed response, expiry, rate limit, and redacted upstream errors.
- Contract/adapter/capability negotiation and fail-closed unsupported requirements.
- Versioned model enforcement.
- Endpoint allowlist and SSRF resistance.
- Idempotent replay returns the same unexpired descriptor; conflicting replay fails.
- Provider/profile binding and call expiry/close.
- Assert permanent keys never appear in response, state snapshots, or logs.

## Acceptance criteria

1. Desktop can obtain a usable short-lived authorization for either configured provider.
2. Permanent keys remain backend-only.
3. Shared call code does not assume a field named `client_secret` or one transport.
4. Calls bind provider, model, adapter, capability fingerprint, Hermes profile/session, and expiry.
5. No default test uses paid APIs.

## Non-goals

- media connection
- tool execution
- transcript persistence
- arbitrary third-party endpoint configuration
