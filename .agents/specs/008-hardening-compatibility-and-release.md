# Spec 008: Hardening, compatibility, and release

## Objective

Produce a reproducible release candidate with explicit Hermes/OpenAI compatibility, security review, live end-to-end validation, packaging, and rollback instructions.

## Dependencies

- Specs 001–007

## Deliverables

- CI workflows and test matrix.
- Hermes compatibility contract tests.
- Opt-in live OpenAI/Electron smoke harness.
- Security and privacy checklist with threat-model findings resolved.
- Versioned release artifacts, checksums, install/upgrade/rollback docs, and changelog.

## Tests

### Automated quality gates

Every pull request runs:

- desktop formatting, lint, strict typecheck, unit tests, and production bundle;
- backend formatting/lint/typecheck/unit tests;
- cross-component API fixture tests;
- installer tests in a temporary `HERMES_HOME`;
- plugin load test against supported Hermes versions;
- secret scanning and generated-bundle inspection;
- dependency vulnerability audit with reviewed exceptions; and
- documentation link/structure checks.

No default CI job requires an OpenAI key, microphone, audio device, or paid service.

## Hermes compatibility

Define and publish a supported version range. For each supported release fixture/check-out:

- load the backend plugin;
- call `/capabilities`;
- resolve the intended Desktop tool scope;
- generate schemas;
- execute a harmless tool through the real dispatcher;
- create/finalize/export/resume a dedicated voice session; and
- verify approval-required tools remain disabled or complete the native approval flow.

A changed Hermes signature must fail a clear contract test rather than degrading to unrestricted tools or direct database writes.

## Live Realtime smoke test

An explicitly invoked, budget-capped test validates:

1. backend mints a client secret;
2. a real browser/Electron peer negotiates WebRTC;
3. known input audio reaches OpenAI;
4. model audio is received and measurable;
5. final user and assistant transcript events arrive;
6. speech interruption cancels/truncates output;
7. one harmless Hermes function tool completes; and
8. close releases media and persists a valid Hermes session.

The harness prints request IDs and timing summaries but redacts content and credentials. It enforces a hard duration/token/cost ceiling and always closes the call in `finally` cleanup.

## Performance validation

Capture, without transcript content:

- bootstrap latency;
- peer negotiation latency;
- speech-stop to first model audio;
- barge-in detection to model-audio stop;
- tool request to tool result; and
- cleanup completion.

Initial release targets are defined in `docs/architecture.md`. A regression budget should be encoded once a stable baseline exists.

## Security review

Threat model at minimum:

- malicious renderer/plugin caller attempting to mint secrets;
- stolen/replayed ephemeral secret;
- cross-profile/cross-call request confusion;
- tool name/argument tampering;
- stale tool catalog and permission revocation;
- duplicate side-effect execution;
- approval bypass or timeout confusion;
- transcript/tool payload leakage in logs;
- prompt/tool-output injection;
- backend SSRF or configurable endpoint abuse;
- oversized event/tool payload denial of service; and
- abandoned calls retaining microphone or backend state.

Resolve high-severity findings before release. Document accepted residual risks.

## Packaging

Release artifacts include:

- desktop `plugin.js`;
- backend plugin package/tree;
- installer/uninstaller;
- source archive;
- SHA-256 checksums; and
- generated SBOM if practical.

Installation creates a rollback point before replacing files. Artifacts are versioned together and reject incompatible desktop/backend protocol versions.

## Documentation

Publish:

- prerequisites and supported platforms;
- OpenAI credential setup through Hermes-supported secret handling;
- install, enable, update, rollback, and uninstall;
- microphone permissions by OS;
- configuration reference;
- remote-backend behavior;
- tool-safety/approval limitations;
- privacy and data-flow statement;
- troubleshooting and redacted diagnostics; and
- known limitations.

## Acceptance criteria

1. All automated gates pass from a clean checkout.
2. Live smoke passes on at least Linux and one of macOS/Windows; remaining supported platforms receive documented manual validation.
3. No critical/high unresolved security findings.
4. No standard or ephemeral credential appears in artifacts, logs, tests, or persisted sessions.
5. Install, upgrade, rollback, and uninstall are verified against a disposable profile.
6. Supported Hermes versions fail closed when compatibility assumptions break.
7. The release notes state that audio goes directly to OpenAI and identify what transcript/tool metadata Hermes stores.

## Non-goals

Deferred follow-ups:

- additional Realtime providers
- SIP/PSTN and mobile transports
- encrypted raw-audio recording
- multi-party calls
- seamless handoff between an active text agent loop and a live voice session
