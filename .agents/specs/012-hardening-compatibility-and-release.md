# Spec 012: Hardening, compatibility, and release

## Objective

Ship a reproducible release candidate with explicit Hermes, internal-contract, OpenAI, xAI, Desktop, Discord, and Telegram compatibility plus drift detection, security review, live validation, and rollback.

## Dependencies

- Specs 001–011

## Deliverables

- CI across provider and endpoint contract matrices.
- Versioned/redacted fixtures for both providers and every endpoint mode.
- Opt-in provider, Discord, Telegram Bot API, and Telegram MTProto live smoke harnesses.
- Model/protocol drift canary and compatibility report.
- Security review, release artifacts/checksums, install/upgrade/rollback docs.

## Tests

Default CI runs formatting/lint/typecheck/unit/build, core/provider/endpoint dependency rules, both provider contract suites, every endpoint contract suite, fixture/schema snapshots, backend API contracts, Hermes compatibility, installer tests, secret scanning, bundle inspection, dependency audit, and documentation checks. It requires no provider/platform key or paid service.

Separate budget-capped live tests verify for each provider through Desktop: ephemeral bootstrap, real browser transport, input/output audio, final transcripts, barge-in/cancel, one harmless Hermes tool, cleanup, and valid Hermes history. xAI additionally verifies binary framing/cumulative corrections/resumption; OpenAI verifies SDP/data-channel/WebRTC truncation behavior.

Endpoint live tests separately verify Discord Voice Gateway v8/DAVE media and participant attribution, Telegram Bot API voice-note behavior, and the opt-in Telegram MTProto sidecar. The Telegram Bot API test is explicitly asynchronous and cannot satisfy the live-duplex gate.

Drift canaries:

- use pinned production models;
- compare discovered provider capabilities/models with recorded compatibility data;
- report—not auto-adopt—new `latest` targets;
- tolerate additive unknown events while alerting;
- fail when required events/capabilities disappear;
- capture new redacted fixtures only through explicit review; and
- record exact provider/model/adapter versions in results.

Security review covers malicious bootstrap, credential replay/exfiltration, endpoint redirection/SSRF, cross-provider/profile/call confusion, stale capabilities/catalog, participant impersonation, authority inheritance, Discord voice tokens/DAVE material, Telegram MTProto session theft, sidecar IPC, duplicate tools, approval bypass, payload injection/limits, media backpressure, transcript/log leakage, and abandoned media.

Release artifacts version desktop/backend/sidecar protocols together and publish supported Hermes versions plus provider and endpoint compatibility matrices. Installation creates rollback before replacement.

## Acceptance criteria

1. All offline gates pass cleanly.
2. Live smoke passes for both OpenAI and xAI through Desktop and Discord on Linux before stable release; Telegram Bot API voice notes pass separately.
3. Telegram MTProto live-call support remains opt-in/experimental until its dedicated account, attribution, and call-engine matrix passes on every advertised platform.
4. Deleting either provider or any endpoint adapter leaves remaining builds/tests healthy.
5. No critical/high unresolved security findings or credentials in artifacts/logs/history.
6. Unsupported Hermes/provider/platform drift fails closed with actionable diagnostics.
7. Install/upgrade/rollback/uninstall are verified in a disposable profile.
8. Privacy docs identify media destinations, visible call identities, participant attribution level, and persisted metadata for each endpoint/provider pair.

## Non-goals

- automatic migration to new provider models/protocols
- SIP/PSTN and cross-platform conference bridging
- encrypted raw-audio recording
- seamless mid-call provider handoff
