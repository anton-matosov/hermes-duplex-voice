# Spec 001: Verify extension seams and package plugins

## Objective

Prove the current Hermes Desktop and backend plugin surfaces needed by Duplex Voice, then create the smallest reversible paired-plugin skeleton. Do not build provider abstractions or media yet.

## Dependencies

None.

## Deliverables

```text
desktop/
├── src/plugin.ts
├── package.json
├── tsconfig.json
└── build.mjs
backend/
├── plugin.yaml
├── __init__.py
└── dashboard/
    ├── manifest.json
    └── plugin_api.py
scripts/
├── install.py
├── uninstall.py
└── verify_install.py
tests/
├── desktop/
└── backend/
```

- One Desktop action and small panel rendered by a runtime plugin.
- One authenticated `GET /api/plugins/hermes-duplex-voice/config` backend route.
- Exact ID invariant: Desktop plugin `id`, folder name, backend plugin name, and dashboard manifest `name` are all `hermes-duplex-voice`, because `ctx.rest()` scopes by the Desktop ID.
- Backend artifact uses the supported user-plugin directory layout: `plugins/hermes-duplex-voice/dashboard/manifest.json` declares `"api": "plugin_api.py"`, and that module exports `router = APIRouter()`.
- Strict TypeScript source bundled to one `plugin.js` with only `@hermes/plugin-sdk`, `react`, and `react/jsx-runtime` external.
- Profile-aware, reversible installation using official Hermes plugin workflows where available and a rollback copy before replacement.
- A compatibility note recording the verified Hermes version/commit and exact seams used.

## Required seam probes

Verify against a disposable `HERMES_HOME`:

1. `<HERMES_HOME>/desktop-plugins/hermes-duplex-voice/plugin.js` hot-loads and hot-unloads.
2. `composer.actions` renders without replacing built-in dictation.
3. Runtime plugin code can access `navigator.mediaDevices`, `RTCPeerConnection`, and an autoplay audio element after user gesture.
4. `ctx.rest('/config')` reaches the plugin-scoped backend route with local and authenticated remote backend modes.
5. Backend `plugin_api.py` is imported only when the user plugin is explicitly enabled, is mounted once at `hermes serve` startup, and backend route changes require restart.
6. Profile switch routes Desktop and backend calls to the same active profile and unload/cleanup callbacks run.
7. Remote mode documents and tests the two-part deployment: local Desktop `plugin.js`, plus the enabled backend directory plugin on the remote host/profile followed by backend restart. The local installer must not pretend it installed the remote half.

If a seam fails, document the smallest generic Hermes upstream change. Do not patch an installed Hermes checkout.

## Tests

- Desktop typecheck and bundle.
- Reject unsupported imports and accidental bundled secrets.
- Plugin registration/disposal smoke.
- Backend route auth, response schema, and disabled-plugin gate.
- Fresh install, upgrade with backup, rollback, uninstall, and named-profile isolation in temporary homes.
- Local and remote `ctx.rest()` routing smoke using Hermes's real plugin loader where practical.

## Acceptance criteria

1. The paired skeleton loads through supported Hermes surfaces with no checkout patch.
2. The Desktop action calls the authenticated backend route and displays safe readiness data.
3. Install/upgrade/uninstall is reversible and profile-scoped.
4. The runtime bundle contains no unsupported import or secret.
5. Every required media/API seam is verified or captured as a concrete blocker before OpenAI work starts.

## Non-goals

- OpenAI network calls;
- microphone capture beyond capability probing;
- provider/endpoint registries;
- tools or persistence;
- xAI, Discord, or Telegram packaging.
