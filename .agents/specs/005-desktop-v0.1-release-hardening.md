# Spec 005: Desktop/OpenAI v0.1 release hardening

## Objective

Ship a reversible, accessible, supportable OpenAI + Hermes Desktop release. This is the Track A release gate; later providers and endpoints are not dependencies.

## Dependencies

- Specs 001–004.

Specs 006–007 are v0.2 integrations and do not block this release.

## Deliverables

- Polished but compact call panel and setup flow.
- OpenAI model/voice readiness and safe diagnostics.
- Keyboard/accessibility coverage.
- Release artifacts for Desktop and backend plugins.
- Install, upgrade, rollback, uninstall, and troubleshooting docs.
- Offline CI and opt-in budget-capped OpenAI live smoke.
- Supported Hermes version range based on actual compatibility tests.

## Required UX

- Start, mute/unmute, hang up.
- Clear connection/listening/user-speaking/responding/assistant-speaking/error states.
- Final transcript display; provisional transcript is visually distinct.
- Actionable microphone, backend, OpenAI auth/model, ICE, autoplay, and compatibility errors.
- No provider selector or speculative capability matrix; this release is OpenAI-only.
- Diagnostics show plugin/API/Hermes/model version, safe request IDs, state transitions/timings, and error category without transcripts, SDP, credentials, tool payloads, or audio.
- Keyboard operation, non-color-only status, screen-reader announcements for final turns, and reduced-motion behavior.

Device selection may be added only if the tracer-bullet evidence shows the default device is insufficient for release; mute and device-loss handling are mandatory.

## Offline gates

- Format/lint/typecheck/unit/build.
- Bundle/import and secret scan.
- Real plugin load through a temporary `HERMES_HOME`.
- Backend auth/config/session/close contract tests, including provider-call capture and upstream hang-up.
- State/event/cleanup tests, including hot unload and profile/connection switch.
- Install/upgrade/rollback/uninstall/profile isolation.
- Dependency and documentation checks.

No default gate requires a key, network, paid API, or microphone.

## Live gate

In a disposable profile with a pinned model and hard duration/spend cap:

1. initialize through the supported backend plugin path;
2. exchange real bidirectional audio;
3. interrupt assistant speech;
4. hang up;
5. verify no live track/peer/channel/audio resource remains;
6. verify final in-memory transcript/interruption state and no forbidden stored/logged data.

## Acceptance criteria

1. All offline gates pass from a clean checkout.
2. The OpenAI live gate passes on each advertised Desktop platform, or the release clearly limits platform support.
3. Install, upgrade, rollback, and uninstall are verified in disposable profiles.
4. No high/critical unresolved security issue or credential/audio leakage exists.
5. Unsupported Hermes/OpenAI drift fails with actionable diagnostics.
6. OpenAI/Desktop can release without xAI, Discord, or Telegram.

## Non-goals

- multi-provider UI;
- Discord/Telegram;
- distributed deployment;
- generalized adapter compatibility matrices;
- automatic model migration;
- seamless call recovery or provider handoff.
