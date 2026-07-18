# Spec 001: Repository scaffold and plugin packaging

## Objective

Create a development and distribution structure that produces a Hermes Desktop runtime plugin and a Hermes backend plugin without modifying the Hermes source tree.

## Dependencies

None.

## Deliverables

```text
desktop/
├── package.json
├── tsconfig.json
├── build.mjs
├── src/
│   ├── plugin.tsx
│   └── lib/
└── test/
backend/
├── pyproject.toml
├── src/hermes_duplex_voice/
│   ├── __init__.py
│   ├── api.py
│   ├── compat.py
│   ├── config.py
│   └── models.py
└── tests/
plugin/
└── hermes-duplex-voice/
    ├── plugin.yaml
    ├── __init__.py
    └── dashboard/
        ├── manifest.json
        ├── plugin_api.py
        └── dist/index.js
scripts/
├── install.py
├── uninstall.py
└── verify_install.py
```

The exact packaging may be adjusted if Hermes's current plugin installer requires a different standalone-repository shape, but source and generated artifacts must remain clearly separated.

## Requirements

### Desktop build

- Author in TypeScript with strict type checking.
- Bundle to one plain-JavaScript ESM file named `plugin.js`.
- Mark only `@hermes/plugin-sdk`, `react`, and `react/jsx-runtime` as runtime externals.
- Reject every other bare import during build.
- Do not require a Node runtime after installation.
- Register a disabled-by-default placeholder composer action and panel proving hot load/unload works.

### Backend package

- Use a Python package importable from the Hermes runtime.
- Export a Hermes `register(ctx)` entry point and an API router mounted under the plugin namespace.
- Keep all direct Hermes imports inside `compat.py`.
- Provide `/capabilities` returning plugin version, readiness, and compatibility diagnostics.
- Keep behavioral settings in Hermes `config.yaml`; reserve `.env` for credentials.

### Installer and rollback

- Install the desktop artifact under `$HERMES_HOME/desktop-plugins/hermes-duplex-voice/plugin.js`.
- Install the backend plugin under `$HERMES_HOME/plugins/hermes-duplex-voice/` using the official Hermes plugin workflow where supported.
- Detect the active profile and refuse to modify another profile unless explicitly selected.
- Before replacing existing files, create a timestamped backup archive and print the rollback command.
- Never edit Hermes core files.
- Uninstall only files owned by this project and restore the previous backup when requested.

## Public contracts

`GET /api/plugins/hermes-duplex-voice/capabilities` initially returns:

```json
{
  "plugin_version": "0.1.0-dev",
  "compatible": true,
  "hermes_version": "...",
  "desktop_plugin_api": true,
  "backend_plugin_api": true,
  "openai_credential_available": false,
  "reasons": []
}
```

No secret value may appear in this response.

## Tests

- Desktop typecheck and bundle tests.
- Static assertion that the bundle contains no unsupported bare imports.
- Python unit tests for capabilities and redaction.
- Installer tests against a temporary `HERMES_HOME` covering fresh install, upgrade backup, rollback, profile scoping, and uninstall.
- Hermes integration smoke test that starts the backend against the temporary home and calls `/capabilities`.

## Acceptance criteria

1. A clean build produces exactly the expected desktop and backend distributable artifacts.
2. Installation does not modify the Hermes repository or unrelated profile files.
3. Hermes Desktop inventories the runtime plugin and the backend serves `/capabilities`.
4. A second install is idempotent and creates a recoverable backup before replacement.
5. All tests run without OpenAI credentials or network access.

## Non-goals

- WebRTC negotiation
- OpenAI API calls
- tool execution
- transcript persistence
