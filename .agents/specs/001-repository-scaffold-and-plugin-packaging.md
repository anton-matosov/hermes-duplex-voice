# Spec 001: Repository scaffold and plugin packaging

## Objective

Create buildable Hermes Desktop/backend plugins, provider modules, channel endpoint modules, and an optional Telegram sidecar that evolve independently without patching Hermes.

## Dependencies

None.

## Deliverables

```text
desktop/src/
├── plugin.tsx
├── endpoint/                # Desktop media/UI
└── providers/{openai,xai}/
backend/src/hermes_duplex_voice/
├── core/                    # calls, policy, persistence
├── hermes_compat.py
└── providers/{base,openai,xai}.py
endpoints/
├── discord/
├── telegram_bot/
└── telegram_mtproto/        # optional sidecar
tests/fixtures/providers/{openai,xai}/
tests/fixtures/endpoints/{discord,telegram_bot,telegram_mtproto}/
scripts/{install,uninstall,verify_install}.py
```

- Strict TypeScript desktop source bundled to one ESM `plugin.js`.
- Python backend plugin with authenticated API router.
- Provider and endpoint registries with explicit IDs, adapter versions, and runtime placements.
- `/capabilities` with contract version, Hermes compatibility, and per-provider/endpoint readiness.
- Optional sidecar artifact with versioned local IPC and separate install/rollback state.
- Reversible, profile-aware installer using official Hermes workflows where supported.

## Boundaries

- Shared `core` modules may depend on provider/endpoint interfaces, never implementations or raw wire types.
- Provider and endpoint implementations may depend on core contracts, never each other's wire protocols.
- Build/lint rejects provider/endpoint-name conditionals outside registries, adapter directories, configuration presentation, and tests.
- Runtime bundle may externalize only Hermes-supported imports.
- Direct Hermes imports live only in `hermes_compat.py`.
- Provider credentials remain in the backend secret environment, Discord/Telegram Bot credentials remain in the gateway, and MTProto session material remains in the sidecar.

Initial capability response:

```json
{
  "contract_versions": [1],
  "plugin_version": "0.1.0-dev",
  "hermes": {"compatible": true, "version": "..."},
  "providers": {
    "openai": {"ready": false, "adapter_version": "1.0.0", "reasons": ["missing_credential"]},
    "xai": {"ready": false, "adapter_version": "1.0.0", "reasons": ["missing_credential"]}
  },
  "endpoints": {
    "desktop": {"ready": true, "mode": "live_duplex"},
    "discord_voice": {"ready": false, "mode": "live_duplex", "reasons": ["missing_platform"]},
    "telegram_voice_message": {"ready": false, "mode": "voice_message", "reasons": ["missing_platform"]},
    "telegram_group_call": {"ready": false, "mode": "live_duplex", "reasons": ["sidecar_not_configured"]}
  }
}
```

No secret or arbitrary endpoint appears in this response.

## Tests

- Desktop typecheck/bundle and unsupported-import checks.
- Dependency-boundary test for core/provider/endpoint imports.
- Provider registry duplicate/unknown ID tests.
- Capabilities/redaction tests with zero, one, and both credentials present.
- Installer fresh/upgrade/rollback/profile/uninstall tests in temporary `HERMES_HOME`.
- Hermes plugin load and `/capabilities` smoke.

## Acceptance criteria

1. Clean build produces Desktop/backend, endpoint, and optional sidecar artifacts.
2. Removing either provider or any endpoint implementation and registry entry leaves remaining builds/tests passing.
3. Installation modifies no Hermes core or unrelated profile files and creates a rollback point.
4. `/capabilities` distinguishes provider readiness without revealing credentials.
5. Tests require no network, microphone, or provider key.

## Non-goals

- provider authentication calls
- audio transports
- tool execution
- transcript persistence
