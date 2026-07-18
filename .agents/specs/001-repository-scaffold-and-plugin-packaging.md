# Spec 001: Repository scaffold and plugin packaging

## Objective

Create buildable Hermes Desktop/backend plugins with provider modules that can evolve independently without patching Hermes.

## Dependencies

None.

## Deliverables

```text
desktop/src/
├── plugin.tsx
├── core/                    # no provider imports
└── providers/{openai,xai}/
backend/src/hermes_duplex_voice/
├── core/                    # calls, policy, persistence
├── hermes_compat.py
└── providers/{base,openai,xai}.py
tests/fixtures/providers/{openai,xai}/
scripts/{install,uninstall,verify_install}.py
```

- Strict TypeScript desktop source bundled to one ESM `plugin.js`.
- Python backend plugin with authenticated API router.
- A desktop and backend provider registry with explicit IDs and adapter versions.
- `/capabilities` with contract version, Hermes compatibility, and per-provider readiness.
- Reversible, profile-aware installer using official Hermes workflows where supported.

## Boundaries

- Shared `core` modules may depend on provider interfaces, never provider implementations or raw event types.
- Provider implementations may depend on core contracts.
- Build/lint rejects provider-name conditionals outside registry, provider directories, configuration presentation, and tests.
- Runtime bundle may externalize only Hermes-supported imports.
- Direct Hermes imports live only in `hermes_compat.py`.
- Permanent credentials remain in Hermes secret environment.

Initial capability response:

```json
{
  "contract_versions": [1],
  "plugin_version": "0.1.0-dev",
  "hermes": {"compatible": true, "version": "..."},
  "providers": {
    "openai": {"ready": false, "adapter_version": "1.0.0", "reasons": ["missing_credential"]},
    "xai": {"ready": false, "adapter_version": "1.0.0", "reasons": ["missing_credential"]}
  }
}
```

No secret or arbitrary endpoint appears in this response.

## Tests

- Desktop typecheck/bundle and unsupported-import checks.
- Dependency-boundary test for core/provider imports.
- Provider registry duplicate/unknown ID tests.
- Capabilities/redaction tests with zero, one, and both credentials present.
- Installer fresh/upgrade/rollback/profile/uninstall tests in temporary `HERMES_HOME`.
- Hermes plugin load and `/capabilities` smoke.

## Acceptance criteria

1. Clean build produces both plugin artifacts.
2. Removing either provider implementation and registry entry leaves the other provider build/tests passing.
3. Installation modifies no Hermes core or unrelated profile files and creates a rollback point.
4. `/capabilities` distinguishes provider readiness without revealing credentials.
5. Tests require no network, microphone, or provider key.

## Non-goals

- provider authentication calls
- audio transports
- tool execution
- transcript persistence
