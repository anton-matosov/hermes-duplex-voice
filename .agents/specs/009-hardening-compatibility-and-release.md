# Spec 009: Hardening, compatibility, and release

## Objective

Ship a reproducible release candidate with explicit Hermes, internal-contract, OpenAI, and xAI compatibility plus drift detection, security review, live validation, and rollback.

## Dependencies

- Specs 001–008

## Deliverables

- CI and provider contract matrix.
- Versioned/redacted fixtures for both providers.
- Opt-in OpenAI and xAI live smoke harnesses.
- Model/protocol drift canary and compatibility report.
- Security review, release artifacts/checksums, install/upgrade/rollback docs.

## Tests

Default CI runs formatting/lint/typecheck/unit/build, core/provider dependency rules, both provider contract suites, fixture/schema snapshots, backend API contracts, Hermes compatibility, installer tests, secret scanning, bundle inspection, dependency audit, and documentation checks. It requires no provider key or paid service.

Separate budget-capped live tests verify for each provider: ephemeral bootstrap, real browser transport, input/output audio, final transcripts, barge-in/cancel, one harmless Hermes tool, cleanup, and valid Hermes history. xAI additionally verifies binary framing/cumulative corrections/resumption; OpenAI verifies SDP/data-channel/WebRTC truncation behavior.

Drift canaries:

- use pinned production models;
- compare discovered provider capabilities/models with recorded compatibility data;
- report—not auto-adopt—new `latest` targets;
- tolerate additive unknown events while alerting;
- fail when required events/capabilities disappear;
- capture new redacted fixtures only through explicit review; and
- record exact provider/model/adapter versions in results.

Security review covers malicious bootstrap, credential replay/exfiltration, endpoint redirection/SSRF, cross-provider/profile/call confusion, stale capabilities/catalog, duplicate tools, approval bypass, payload injection/limits, WebSocket backpressure, transcript/log leakage, and abandoned media.

Release artifacts version desktop/backend protocol together and publish supported Hermes versions plus provider model/protocol matrix. Installation creates rollback before replacement.

## Acceptance criteria

1. All offline gates pass cleanly.
2. Live smoke passes for both OpenAI and xAI on Linux and at least one additional supported OS before stable release.
3. Deleting either adapter leaves the remaining provider build/tests healthy.
4. No critical/high unresolved security findings or credentials in artifacts/logs/history.
5. Unsupported Hermes/provider drift fails closed with actionable diagnostics.
6. Install/upgrade/rollback/uninstall are verified in a disposable profile.
7. Privacy docs identify direct media destinations and persisted metadata for each provider.

## Non-goals

- automatic migration to new provider models/protocols
- SIP/PSTN or multi-party calls
- encrypted raw-audio recording
- seamless mid-call provider handoff
